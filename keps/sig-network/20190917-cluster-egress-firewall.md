---
title: Cluster Egress Firewall
authors:
  - "@danwinship"
owning-sig: sig-network
participating-sigs:
reviewers:
  - TBD
  - "@thockin"
approvers:
  - TBD
  - "@thockin"
editor: TBD
creation-date: 2019-09-17
last-updated: 2019-09-17
status: provisional
see-also:
replaces:
superseded-by:
---

# ClusterEgressFirewall

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [Undecided Goals](#undecided-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Optional Cloud Metadata Blocking](#optional-cloud-metadata-blocking)
    - [Blocking Access to Similar Overly-Trusting Services](#blocking-access-to-similar-overly-trusting-services)
    - [Blocking Access to Services Used by the Node](#blocking-access-to-services-used-by-the-node)
  - [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Examples](#examples)
      - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
      - [Beta -&gt; GA Graduation](#beta---ga-graduation)
      - [Removing a deprecated flag](#removing-a-deprecated-flag)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
- [Drawbacks [optional]](#drawbacks-optional)
- [Alternatives [optional]](#alternatives-optional)
- [Infrastructure Needed [optional]](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

**ACTION REQUIRED:** In order to merge code into a release, there must be an issue in [kubernetes/enhancements] referencing this KEP and targeting a release milestone **before [Enhancement Freeze](https://github.com/kubernetes/sig-release/tree/master/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core Kubernetes i.e., [kubernetes/kubernetes], we require the following Release Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These checklist items _must_ be updated for the enhancement to be released.

- [ ] kubernetes/enhancements issue in release milestone, which links to KEP (this should be a link to the KEP location in kubernetes/enhancements, not the initial KEP PR)
- [ ] KEP approvers have set the KEP status to `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

**Note:** Any PRs to move a KEP to `implementable` or significant changes once it is marked `implementable` should be approved by each of the KEP approvers. If any of those approvers is no longer appropriate than changes to that list should be approved by the remaining approvers and/or the owning SIG (or SIG-arch for cross cutting KEPs).

**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://github.com/kubernetes/enhancements/issues
[kubernetes/kubernetes]: https://github.com/kubernetes/kubernetes
[kubernetes/website]: https://github.com/kubernetes/website

## Summary

`ClusterEgressFirewall` (possibly to-be-renamed) provides a means for
an administrator to reliably block pods from accessing certain
destinations outside the cluster network.

In particular, this attempts to solve two existing problems with
attempts to block egress traffic:

  - There are a wide variety of ways that traffic can leave the
    cluster, with new ones occasionally being added, and it is
    difficult to reliably block all of them.

  - In the common case of trying to block traffic from pods to ports
    on nodes or masters, the obvious approaches are insufficient, but
    not obviously so, leading people to think they have protected
    themselves when they haven't really.

Unlike with NetworkPolicy, which is expected to be implemented
entirely by the network plugin, `ClusterEgressFirewall` will be a
shared responsibility; any component of Kubernetes that provides a way
for packets to exit the cluster (eg, secondary network interfaces)
must ensure that egress firewall rules will be obeyed by exiting
traffic. Any component that has the effect of obscuring the origin of
packets (eg, proxies, load balancers, or even just masquerading) must
ensure that they cannot be used to bypass egress firewall rules. (Of
course this means that especially in the beginning, many third-party
components will not be ClusterEgressFirewall-compliant, and we'll need
to handle that with clear documentation.)

## Motivation

Cluster administrators sometimes want to block pods from accessing
particular external destinations. This turns out to be remarkably
difficult to do in many cases, because the Kubernetes ecosystem
provides so many different ways to move packets around.

For example, consider an administrator who wants to prevent pods from
connecting to the "ssh" port on nodes. Assuming that the pod network
is `172.17.0.0/16`, and the nodes are all on the subnet `10.0.1.0/24`,
the administrator might try adding a firewall rule like:

    iptables -I INPUT -s 172.17.0.0/16 -d 10.0.1.0/24 -p tcp -m tcp --dport 22 -j REJECT

But there are numerous problems with this approach:

  - In some cases, an attacker could create a Service with its
    Endpoints pointing at a node IP, and then connect to that Service
    from the pod, and the origin IP would be obscured, bypassing the
    iptables rule. With most network plugins, this wouldn't work with
    an ordinary ClusterIP service, but in some cloud environments it
    would work with a LoadBalancer service (eg, in AWS, where
    kube-proxy doesn't know the IPs of the load balancers and so can't
    write rules to short-circuit pod-to-loadbalancer traffic).

      - Even if a LoadBalancer service doesn't allow bypassing the
        iptables firewall rule _when connected to from a pod_, the
        attacker could still create a LoadBalancer service pointing to
        the node's ssh port and then connect to it _from outside the
        cluster_, which the admin presumably also didn't want.

  - An attacker that is able to create a [secondary network
    attachment] in a pod might be able to route traffic to the node
    through that interface rather than its default network interface,
    and connect to the node's IP via that unrecognized source IP.

  - The iptables rule blocks the node's primary IP, but the node may
    have additional IPs. Eg, assuming that the example above has pods
    attached to `docker0`, a pod might be able to ssh to its node by
    connecting to `172.17.0.1`. Nodes may also have additional public
    IP addresses in some cases, either created by the administrator
    (eg, a second network interface attached to a backend network), or
    created by some third-party Kubernetes component as part of its
    own functionality.

  - (And of course, if the cluster later expands to include nodes
    whose IPs aren't included in the subnet in the original firewall
    rule, the administrator needs to remember to update the rules on
    every node to restrict traffic to the new node addresses as well.)

Of course, all of these problems can be worked around, but (based on
comments seen in past discussions and feature requests) administrators
are unlikely to realize the complete scope of the problem, and so are
unlikely to take the correct steps to work around them. (And a
solution that works at one point in time might suddenly fail in the
next release, or after the addition of a new third-party component,
due to the addition of a new feature that provides a new way around
the firewalling rule.)

[secondary network attachment]: https://docs.google.com/document/d/1Ny03h6IDVy_e_vmElOqR7UdTPAG_RNydhVE1Kx54kFQ/edit

### Goals

- An administrator should be able to block access to specified
  destinations outside the cluster, in a way that prevents all
  (non-privileged, non-hostNetwork) pods in the cluster from being
  able to access them, either directly or via Services or other
  Kubernetes-controlled redirectors/proxies.

- An administrator should be able to block access to specified ports
  on nodes, in a way that prevents all (non-privileged,
  non-hostNetwork) pods in the cluster from being able to access them;
  and the rules created to implement this should automatically update
  as the set of nodes change, and should cover all IP addresses on the
  node. (That is, there should be special syntax for saying that you
  want to block access to nodes, without the administrator needing to
  enumerate node IPs.)

    - Many administrators want to block *all* pod-to-node access
      (perhaps with one or two exceptions such as DNS). It should be
      possible to do this very easily, and this should be shown as an
      example in the documentation.

- An administrator should be able to block access to the apiserver, in
  a way that prevents all (non-privileged, non-hostNetwork) pods in
  the cluster from being able to access it; and the rules created to
  implement this should automatically update as the set of masters
  change, and should cover all ways of reaching the apiserver (eg,
  `kubernetes.default` Service, its endpoints, ...). (As above, this
  means special syntax.)

- An administrator should be able to prevent users from providing
  access to restricted destinations in or around the cluster via
  externally-accessible Services (eg, ExternalIP, LoadBalancer).

- It should be possible to have multiple API objects each specifying
  firewall rules, which can be independently added and removed.

### Non-Goals

- Being able to block `hostNetwork` pods; this would necessarily
  entail also blocking the nodes themselves, which is usually
  explicitly not wanted. (eg, see the [Blocking Access to Services
  Used by the Node](#blocking-access-to-services-used-by-the-node) use
  case.)

- Being able to (fully) block privileged pods. In some cases,
  Kubernetes can implement blocking by putting rules in places that
  are inaccessible to pods (eg, in the root network namespace). But in
  other cases, traffic will go directly from the pod to the external
  network (eg, when the pod has a macvlan or SR-IOV interface), so the
  only way Kubernetes could block it would be with iptables/nft/eBPF
  rules inside the pod's network namespace. But in that case, a
  privileged pod would be able to remove those restrictions from
  itself if it wanted.

### Undecided Goals

- DNS-based rules. eg, "block access to everything except
  `api.example.com`". People definitely want this but it's hard to do
  well.

    - If we want to implement this, the right way is probably with a
      CoreDNS plugin; have a way to tell CoreDNS "I want to know what
      the IP address(es) of `api.example.com` is/are, and then if it
      changes in the future, you have to tell me, synchronously,
      *before* telling anyone else".

    - A DNS-based "deny" rule (eg, "don't allow connections to
      `api.example.com`") can never be completely secure; no matter
      what we do, it is impossible to know with certainty that we have
      discovered all possible IPs that serve a given DNS name, and a
      pod could theoretically have outside means of learning IP
      addresses that we don't know about. (Additionally, if the pod
      has access to at least one cluster-external host that the
      attacker controls, then the attacker could set that host up as a
      proxy to access the denied host anyway.)

      On the other hand, DNS-based "allow" rules on top of a larger
      "deny" substrate are secure (as long as a single IP address is
      not hosting both sites you want to allow and sites you want to
      block).

    - In some cases an L7 proxy is a better way to implement this. eg,
      an HTTP proxy can easily do DNS-based allow/deny.

- Per-Namespace firewall rules. While there are certainly good use
  cases for this, it makes things more complicated:

    - It potentially requires the network plugin to do its filtering
      in a more-complicated way. Eg, if all pods have the same egress
      firewall policies then packets passing through the service proxy
      can be filtered after passing through the proxy. But if some
      pods are allowed and others aren't, and the proxy sometimes NATs
      the packets, then the network plugin might need to instead
      filter the packets before they reach the service proxy, which
      implies that it has to be keeping track of what services have
      possibly-blocked endpoints.

    - It requires some components *other* than the network plugin to
      track the source namespace of packets when they otherwise
      wouldn't need to. (eg, an HTTP proxy that serves multiple
      namespaces might need to accept or deny connections depending on
      the namespace of the source pod). This potentially makes things
      *much* more complicated for these components.

    - Particularly because of the effect on non-network-plugin
      components, it might make sense to only allow "allow" rules
      per-Namespace, not "deny" rules. Then if some component is
      unable to implement per-Namespace rules, the effect would be
      that they block good traffic, not that they permit evil traffic.

- Per-Node restrictions might also be useful (eg, allowing pods on
  "infrastructure" nodes to access destinations that pods on ordinary
  nodes cannot). This would probably be less complicated than
  per-Namespace rules, but also less powerful.

## Proposal

FIXME

### User Stories

#### Optional Cloud Metadata Blocking

In AWS (and some other clouds), the "metadata service"
(http://169.254.169.254/) may reveal sensitive information that should
not be given to untrusted pods. Many Kubernetes installation guides
recommend blocking access to it. (The official [Securing a Cluster]
documentation suggests using `NetworkPolicy` to block access to it,
which is fine if you trust your users and are only worried about
external attackers.)

[Securing a Cluster]: https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/#restricting-cloud-metadata-api-access

OpenShift 4.x on AWS blocks access to the metadata service by default.
However, in OpenShift 3.x some users had been making use of metadata
proxies such as [kube2iam] or [kiam] to allow *some* pods to get
*some* metadata, but not all of it. This use case is now broken in
OpenShift 4.x because there is no way to disable the built-in
blocking.

Although it would be possible to just implement an OpenShift-specific
configuration option to change the OpenShift-specific behavior, this
could also be handled via `ClusterEgressFirewall`; OpenShift would
implement the block by creating an appropriate egress firewall object
at install time, and administrators could delete it if they wanted to
use a metadata proxy (or if they didn't need to worry about untrusted
pods in their cluster).

[kube2iam]: https://github.com/jtblin/kube2iam
[kiam]: https://github.com/uswitch/kiam

#### Blocking Access to Similar Overly-Trusting Services

Much like the example of the AWS metadata service, in many
environments there are services that assume that anyone capable of
talking to them is automatically trusted.

In OpenShift 4.x, [pods need to be blocked from talking to the Machine
Config Server](https://github.com/openshift/origin/pull/22821). (In
particular, if a pod can use a secondary network attachment to get an
unused IP on the node network, it may be able to trick the MCS into
believing that it is a newly-created node, and have the MCS send it a
provisioning image that may contain mildly-interesting credentials.)
OpenShift would want to include a policy enforcing this as part of the
default cluster setup.

Similarly, in ovn-kubernetes, there was a problem where nodes needed
to be able to provide information about the cluster configuration to
the OVN databases, but [pods need to be prevented from doing
so](https://github.com/ovn-org/ovn-kubernetes/pull/790). This problem
was eventually solved with TLS authentication, but if that had not
been possible then it could have been solved with
`ClusterEgressFirewall` instead; the ovn-kubernetes project could
write a resource blocking access to this server, and include it for
the administrator to create along with the other objects that must be
created in order to run ovn-kubernetes as a plugin (`ServiceAccount`,
`Deployment`, etc).

#### Blocking Access to Services Used by the Node

Nodes often need access to various services for their own purposes.
These services may be susceptible to DoS attacks (eg, a logging
server) or contain private data (eg, a package or image repository
containing proprietary software). Admins may therefore want to block
pods from accessing them.

FIXME something something NFS

#### Blocking Access to the Kubernetes apiserver

Many administrators want to explicitly block pods from reaching the
Kubernetes apiserver. There are multiple IPs that would need to be
blocked here (`kubernetes.default` Service IP, the actual host IPs
that are its Endpoints, etc). In some cases there may additional IPs
pointing to the apiserver which the admin may or may not be aware of.
(eg, in OpenShift there is a cloud load balancer IP pointing directly
to the apiservers; there would have to be some way for OpenShift to
indicate this fact so that the network plugin could restrict access to
that IP as well, or else OpenShift would have to configure the cloud
load balancer in a way that enforced relevant `ClusterEgressFirewall`
rules itself.)

### Implementation Details/Notes/Constraints

#### What Needs to be Blocked

In a completely vanilla Kubernetes system, the potential means of
communication with a blocked IP include:

- Direct connection to the blocked IP (via the pod's primary network
  connection)

- Connection via a Service IP that redirects to the blocked IP. (ie, a
  selector-less service with manually-created Endpoints).

- Connection (from inside or outside the cluster) via an ExternalIP /
  NodePort / LoadBalancer Service manually pointed at the blocked IP

- Connection (from inside or outside the cluster) via Ingress pointed
  at a Service pointed at the blocked IP

(FIXME: Is that all?)

The [Network Attachment] spec adds:

- Connection to the blocked IP via a secondary pod network interface

Network plugins may introduce additional routes. eg:

- With openshift-sdn, connection to the blocked IP from an
  egress-router pod. (This is mostly equivalent to using the Network
  Attachment spec to get a second macvlan interface in a pod, except
  that it doesn't actually involve any other CNI plugins.)

- With openshift-sdn, connection to the blocked IP from a pod in a
  Namespace with an egress IP. (This again has some similarities to
  the secondary interface use cases, but the implementation is
  different enough that it needs to be handled separately.)

- Other plugins have other features.

This suggests a few places where blocking needs to occur:

1. Pods need to be prevented from making direct connections to the
   blocked IP, from any network interface. While this could be
   implemented by each network plugin, that would imply that even
   otherwise-trivial plugins like the CNI macvlan plugin would now
   need to have a daemon process to watch for changes to the firewall
   rules and update their blocking accordingly.

   A simpler approach would be to have some process (kubelet? the
   runtime? the default network plugin?) just add iptables(/nft/ebpf)
   rules inside the network namespace of each pod, blocking the
   restricted IPs. This would then apply automatically to all
   available network interfaces.

2. The Service proxy potentially needs to write rules that reject
   connections to services that point to blocked IPs, if we can't rely
   on the network plugin to still be able to block the connections
   after they pass through the service proxy. (We also can't just have
   an admission controller forbidding the creation of Endpoints
   pointing to blocked IPs, because the set of blocked IPs can change
   over time.)

3. Any CloudProviders / Ingress / Gateways / etc that work with Endpoints
   directly rather than indirecting through Services also need to
   block connections.

4. External components would be required to provide filtering for any
   additional connection paths that they provided.

#### Syntax

Since we want to be able to have multiple independent firewall objects
working together, they need composable semantics. (In particular, if
we allow both arbitrary allow rules and arbitrary deny rules then we
also need a "priority" field or some other way of deciding how
different objects will be combined.)

For blocking access to the public internet, the only thing that makes
sense is "block everything by default, but allow X, Y, and Z". ("Block
X, Y, and Z but allow the rest of the internet" doesn't make sense
because the user could just set up a proxy at W which allows access to
X, Y, and Z.) However, for blocking access to intranet/intracloud
sites, an administrator may want either "block everything except ..."
or "allow everything except ..."

The user stories above all involve wanting to block specific sites,
with rules that are independent of each other. This implies we can't
just do "default deny plus multiple allow" as with NetworkPolicy.

If we implement DNS-based rules, then as discussed above, they only
make sense as "allow" rules on top of a larger "deny".

#### ClusterEgressFirewall Controller

The node/master service blocking use case requires keeping track of
all IPs associated with all nodes/masters. (This is not equivalent to
`node.Status.Addresses`, because it needs to necessarily include
*every* local IP on the node, while kubelet and the cloud provider may
only include the IPs they consider important/public.) This information
may be needed by multiple other components, so it makes sense to
figure it out in a central location and then make that information
available to everyone else.

One way to do this would be a ClusterEgressFirewall controller that
reads in abstract rules in a firewall's `spec` (eg, "don't allow
connections to port 22 on nodes") and writes out concrete rules in its
`status` ("don't allow connections to port 22 on 10.0.1.2, 10.0.1.3,
or 10.0.1.4").

This would also make it possible to add new features to the `spec`
section in the future without requiring changes to components that
parse the `status`. Eg, the "DNS-based restrictions" feature discussed
above could be implemented by having the firewall controller translate
from DNS addresses in the `spec` to IP addresses in the `status`.

### Risks and Mitigations

The primary risk is that users rely on the feature but it fails to
protect them.

This could happen either because of a flaw in the base implementation,
or because the user used a piece of third-party software that adds
additional attack vectors but doesn't support the egress firewall
feature.

The third-party problem would have to be solved mostly with
documentation; we would note that extensions that add network-related
functionality should not be assumed to be compatible with the egress
firewall unless they explicitly say that they are.

## Design Details

### Test Plan

### Graduation Criteria

#### Alpha -> Beta Graduation

#### Beta -> GA Graduation

### Version Skew Strategy

When adding new features that will need to work with the egress
firewall, we will need to make sure that those features work safely in
"skewed" clusters. That is, if upgrading one part of the cluster adds
a new egress venue that needs to respect the firewall, then we need to
make sure that either (a) the firewalling implementation for that
feature does not depend on any components elsewhere in the cluster
that might not have been upgraded yet, or (b) the feature fails safe
by being unavailable until the entire cluster has been upgraded.

## Implementation History

- FIXME - Initial proposal

## Alternatives / Related Work

### Calico's GlobalNetworkPolicy

Calico's `[GlobalNetworkPolicy]` is, as the name suggests, a
cluster-wide version of `NetworkPolicy` with several additional
features:

  - The resource is non-namespaced, and policies apply to all
    namespaces by default (although the policy can include a namespace
    selector).

  - Policies can be applied to either pods or nodes.

  - Policies contain an ordered list of "Allow" and "Deny" rules, and
    conflicts between policies are resolved via an `order` field that
    indicates the relative precedence of the policy.

  - There are additional match criteria, such as matching ICMP fields
    and HTTP request headers, matching port ranges, matching pods
    running under specific service accounts, or matching labels by
    prefix/suffix/substring.

As contrasted with the proposal here:

  - `GlobalNetworkPolicy` allows specifying ingress rules as well as
    egress.

  - FIXME what happens with multiple node IPs?

  - ...

[GlobalNetworkPolicy]: https://docs.projectcalico.org/v3.9/reference/resources/globalnetworkpolicy

### Cilium's CiliumNetworkPolicy

FIXME

[Cilium NetworkPolicy docs]: https://cilium.readthedocs.io/en/v1.6/policy/
[examples]: https://cilium.readthedocs.io/en/v1.6/policy/language/

### OpenShift's EgressNetworkPolicy

FIXME

### Istio(?)