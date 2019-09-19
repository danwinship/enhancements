---
title: Multicast Support
authors:
  - "@danwinship"
owning-sig: sig-network
reviewers:
  - TBD
approvers:
  - "@thockin"
editor: TBD
creation-date: 2018-11-09
last-updated: 2019-02-11
status: provisional
---

# Multicast Support

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Existing Work](#existing-work)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Running WildFly (&quot;JBoss&quot;) under Kubernetes](#running-wildfly-jboss-under-kubernetes)
    - [User Story 2](#user-story-2)
  - [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
    - [Baseline Behavior: Block IP Multicast](#baseline-behavior-block-ip-multicast)
    - [&quot;kubernetes-multicast&quot; CNI Plugin](#kubernetes-multicast-cni-plugin)
    - [Plugin-specific multicast plugins](#plugin-specific-multicast-plugins)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
<!-- /toc -->

## Release Signoff Checklist

- [ ] kubernetes/enhancements issue in release milestone, which links to KEP (this should be a link to the KEP location in kubernetes/enhancements, not the initial KEP PR)
- [ ] KEP approvers have set the KEP status to `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/website]: https://github.com/kubernetes/website

## Summary

Currently, support for IP multicast traffic on the pod network is
entirely undefined, with no multicast-related behavior being either
required or forbidden, and different plugins offering different
functionality.

A [previous attempt] at a KEP for multicast support aimed to define
Multicast as a core network plugin feature, with `NetworkPolicy` being
extended to describe how multicast traffic would flow. There were
multiple problems with this approach, however.

This KEP, as with the original, aims first to clarify the default
behavior (that the pod network should not carry multicast traffic
unless configured to so). However, rather than defining multicast as
something that each plugin must implement, it suggests a way of using
the [Network Attachment Specification] to provide multicast to pods
via a separate CNI plugin. Finally, it explains how plugins that want
to provide more featureful or more optimized multicast support can do
so as a compatible expansion of the base plugin.

## Motivation

Some Kubernetes users have workloads that use multicast traffic, and
so need to ensure that multicast traffic will flow between certain
pods.

It is also important to these users to ensure that multicast traffic
*does not* flow between unexpected sets of pods, both for security
reasons (being able to ensure that only trusted pods can participate
in the multicast communication) and also to preserve the independence
of unrelated pods (being able to deploy two unrelated applications
that use the same multicast group, without those applications
interfering with each other).

Other Kubernetes users do not need to use multicast, and are more
concerned with simply ensuring that undesired multicast traffic is
blocked, to prevent a single pod from being able to easily saturate
the network.

See also:
[Multicast in Weave](https://www.weave.works/blog/multicasting-cloud-need-know/),
[Multicast in OpenShift](https://blog.openshift.com/service-discovery-openshift-using-multicast/).

### Goals

- Explicitly define the expected baseline behavior of multicast traffic in a Kubernetes cluster. (Because of existing constraints, this baseline behavior almost certainly has to be "multicast is blocked".)
- Implement a "kubernetes-multicast" CNI plugin that can be used with clusters that implement the Network Attachment Specification (eg, via Multus). This plugin will work with any primary network plugin, but may not be the most efficient approach.
- Explain how network plugins can provide their own "upward-compatible" versions of the multicast CNI plugin if they want to provide better multicast support than the default, in a compatible way.

### Non-Goals

- Requiring all plugins to support multicast.
- Forcing all plugins to support *only* a least-common-denominator form of multicast.
- Specifying anything having to do with multicast between pods and nodes, or pods and external hosts.

## Existing Work

Most plugins currently block all pod-to-pod multicast (either
intentionally, or just accidentally as a consequence of IP/MAC
filtering).

Weave treats multicast as broadcast. There has been discussion of
[implementing IGMP
snooping](https://github.com/weaveworks/weave/issues/178) so that
multicast packets would only be sent to the pods that want them, but
this is not yet implemented. There has been some discussion of
[multicast between pods and external
IPs](https://github.com/weaveworks/weave/issues/1863), which
apparently can be made to work at least in some circumstances. There
is no support for policy/isolation; all multicast packets are sent to
all pods in the cluster.

OpenShift SDN also implements multicast-as-broadcast, but on a
per-namespace basis, [with namespaces having to opt
in](https://docs.okd.io/latest/admin_guide/managing_networking.html#admin-guide-networking-multicast).
(So if you enable multicast on namespace "foo", and a pod in "foo"
sends out a multicast packet, it will be delivered to all other pods
in "foo".) As with Weave, it's not possible to use NetworkPolicy to
restrict multicast traffic, and it's also not possible to extend
multicast traffic across namespace boundaries.

## Proposal

### User Stories

#### Running WildFly ("JBoss") under Kubernetes

WildFly (formerly and sometimes still currently known as JBoss
Application Server) allows running [High
Availability](http://docs.wildfly.org/14/High_Availability_Guide.html)
Java EE applications. It uses a library called
[JGroups](http://www.jgroups.org/) to allow new and old servers to
discover each other as instances are added and removed, and to manage
communication between servers in the cluster.

Although there are several ways to configure JGroups, the default (and
generally preferred) configuration uses multicast for discovery; when
a new server instance is brought up, it joins an IP multicast group
(using either a default multicast IP or one specified in a
configuration file), and then sends a multicast message to that
address announcing its presence. The other existing servers will
respond, allowing each server to learn about each of the others
without having known about any of them in advance.

(As an alternative to using multicast for discovery, there is a
JGroups extension called
[KUBE_PING](https://github.com/jgroups-extras/jgroups-kubernetes) that
allows peers to find each other by making Kubernetes apiserver calls.
Although this provides a workaround for clusters where multicast is
not available, it's generally less preferred since it requires
additional configuration, and in particular, in clusters using RBAC it
may require creating service accounts and role bindings to grant the
WildFly pods the apiserver access they need.)

The default JGroups configuration also uses multicast for most
post-discovery peer-to-peer communication between the servers,
although it is possible to configure it to use unicast TCP instead.

#### User Story 2

TBD (something from Weave?)

### Implementation Details/Notes/Constraints

#### Baseline Behavior: Block IP Multicast

Although the Kubernetes network is normally open-by-default, except
where NetworkPolicy dictates otherwise, it seems like
closed-by-default is a better default for multicast:

1. All plugins should be capable of implementing this behavior (just
drop all traffic on the pod network addressed to 224.0.0.0/4), and
many already do. Some plugins are not capable of implementing anything
other than this behavior. (Eg, GCP just exposes the underlying network
as the pod network, and that network does not implement multicast.)

2. This is the most-commonly-implemented behavior among existing
network plugins anyway.

3. In many clusters, open-by-default multicast would be useful only as
a denial of service attack vector.

4. Even in cases where *some* pods in different namespaces on
different nodes need to communicate via multicast, it is still likely
to be the case that *most* pods will not be interested in multicast
traffic, and sending the traffic to them would be a waste of
bandwidth.

#### "kubernetes-multicast" CNI Plugin

Users who want multicast between some or all pods can use the
"kubernetes-multicast" CNI plugin to set up multicast between pods. An
administrator would create a `NetworkAttachment` resource for each
independent multicast network, and pods that needed to use multicast
would request attachment to their multicast network via the network
attachment annotation. Security would be implemented via the standard
network attachment methods (eg, namespacing the `NetworkAttachment`
resources to limit which users can request which networks).

The default plugin could work something like this:

  - In addition to providing the "kubernetes-multicast" CNI plugin,
    the administrator must also deploy a "kubernetes-multicast"
    Service to proxy multicast traffic.

  - When the CNI plugin runs, it creates some sort of IP-over-IP
    tunnel to the service IP (setting its MTU appropriately based on
    the MTU of the cluster network), and routes `224.0.0.0/4` to that
    interface.

  - When the multicast service receives tunneled multicast packets, it
    looks at the source IP to figure out which pod sent them, then
    forwards the packets to all other pods that share the same
    `NetworkAttachment` as the source pod.

  - FIXME: how does the multicast service track which pods are part of
    which `NetworkAttachment`? As described above, it basically has to
    watch all pods in all namespaces. Having the pods introduce
    themselves to the service at pod startup wouldn't work because
    then they'd lose connectivity if the service restarted. Maybe the
    CNI plugin could create some CRD object to announce its presence
    to the service and then the service would only have to watch those
    instead of watching all pods?

#### Plugin-specific multicast plugins

Some plugins may be able to implement multicast more efficiently than
the default. (eg, routing it directly over the pod network rather than
having to tunnel it, not needing a proxy service). In this case they
can provide their own implementation. This could either be a separate
CNI plugin binary, or simply an invocation of their existing CNI
binary with a different config file. The plugin documentation would
explain what needed to be done to configure `NetworkAttachment`s to
request multicast connectivity, and what other steps would be needed
(eg, if it requires running a proxy service like the default plugin
does).

From an end-user's point of view, there would be no difference between
connecting pods with a plugin-specific multicast plugin and connecting
pods using the default multicast plugin; in both cases they would
simply be using the Network Attachment spec annotation to request a
`NetworkAttachment` that the cluster administrator had created.

### Risks and Mitigations

TBD

## Design Details

### Test Plan

TBD

### Graduation Criteria

Alpha to Beta:

- The documentation indicates the expected default behavior of
  Kubernetes clusters with respect to IP multicast traffic, and how to
  configure non-default behavior.
- The networking tests validate the expected default behavior.
- Optional networking tests validate extended behavior.

Beta to GA:

- Multiple implementations, good user feedback?

### Upgrade / Downgrade Strategy

This would primarily be implemented outside of Kubernetes itself and
would therefore be independent of Kubernetes version. However, we
would like to have the ability to improve the implementation of the
default multicast plugin over time. Any such improvements would need
to be made in a way that does not break existing deployments. (eg,
users should probably be required to request a specific versioned
proxy service image rather than just pulling "`:latest`")

### Version Skew Strategy

As described in Upgrade / Downgrade Strategy

## Implementation History

TBD
