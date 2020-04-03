<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up.  KEPs should not be checked in without a sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please ensure to complete all
  fields in that template.  One of the fields asks for a link to the KEP.  You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [ ] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "title", "authors", "owning-sig",
  "status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary", and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [ ] **Create a PR for this KEP.**
  Assign it to people in the SIG that are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly.  The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

**Note:** Any PRs to move a KEP to `implementable` or significant changes once
it is marked `implementable` must be approved by each of the KEP approvers.
If any of those approvers is no longer appropriate than changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross cutting KEPs).
-->
# KEP-1664: Deprecate and Replace Node.Status.Addresses

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Problems](#problems)
    - [IPv6 and Dual-stack](#ipv6-and-dual-stack)
    - [Address Detection Behavior Across Different Cluster Types](#address-detection-behavior-across-different-cluster-types)
    - [API Cleanliness](#api-cleanliness)
    - [<code>NodeAddressType</code> Semantics](#-semantics)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [Maybe Goals](#maybe-goals)
- [Proposal](#proposal)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
    - [Current Behavior](#current-behavior)
    - [Backward Compatibility](#backward-compatibility)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Implementation](#implementation)
    - [Hostname](#hostname)
    - [IPs](#ips)
    - [<code>--node-ip</code> and Address Sorting](#-and-address-sorting)
    - [Updating Current Node.Status.Addresses Users](#updating-current-nodestatusaddresses-users)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

- [ ] Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] KEP approvers have approved the KEP status as `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

There are problems with the handling of `Node.Status.Addresses` that
currently break or complicate some IPv6 and dual-stack scenarios.

Additionally, `Node.Status.Addresses` behaves inconsistently between
bare metal, legacy cloud providers, and external cloud providers and
generally has unclear and inconsistent semantics. It also has a broken
definition, resulting in it needing to have a note attached to it in
the documentation warning users to not use it with strategic merge
patch.

To resolve these problems, this KEP proposes deprecating the existing
`Node.Status.Addresses` field and replacing it with two new fields,
`Node.Status.Hostname` and `Node.Status.IPs`, with clearer semantics
and more consistent and useful behavior.

## Motivation

### Problems

#### IPv6 and Dual-stack

Nodes implicitly have a "primary" node IP (the first `InternalIP`
address, unless there are no `InternalIP` addresses, in which case the
first `ExternalIP` address) which is used for a handful of purposes,
such as the `Pod.Status.HostIP` value, and by extension, the
`Pod.Status.PodIP` of a host-network pod. This implies that
single-stack clusters must have a primary node IP of the same family
that the pod network uses, or else pod-network pods would be unable to
communicate with host-network pods, violating [the Kubernetes network
model].

When using an external cloud provider, cloud-controller-manager is
responsible for setting `Node.Status.Addresses`, but it does not know
what IP families pods are using, so it doesn't know whether to return
an IPv4 or IPv6 address first as the primary IP. (In theory this
problem could be solved by letting it return them in any order, and
changing kubelet's definition of what the primary IP is. In practice
this would not work well since some external components also use the
same "first `InternalIP` ..." rule that kubelet uses now.)

In a dual-stack cluster, there is currently no definition of
"secondary" / "alternate-family primary" node IP, and host-network
pods currently only get a single IP in `Pod.Status.PodIPs`, which in
turn means that in any given cluster, host-network pods can either
only be the endpoints of IPv4 Services, or else only be the endpoints
of IPv6 Services.

In general, it is not clear if `Node.Status.Addresses` is
expected/allowed to contain IPv6 addresses in a single-stack IPv4
cluster, or IPv4 addresses in a single-stack IPv6 cluster. For cloud
providers that do not currently return IPv6 addresses, starting to
return them might break clients which were written without taking this
possibility into account.

[the Kubernetes network model]: https://kubernetes.io/docs/concepts/cluster-administration/networking/#the-kubernetes-network-model

#### Address Detection Behavior Across Different Cluster Types

On bare-metal, a node always has exactly one IP (even if the cluster
is supposed to be dual-stack), which is auto-detected by kubelet, but
can be overridden with `--node-ip`.

When using a legacy cloud provider, the node IPs are detected by the
cloud provider, and it may return multiple IPs of each type (eg, if
the node has multiple network interfaces, or multiple addresses on a
single interface). eg:

    [ { "type": "ExternalIP", "address": "42.42.42.42" },
      { "type": "InternalIP", "address": "10.1.2.3" },
      { "type": "InternalIP", "address": "10.4.5.6" },
      { "type": "InternalIP", "address": "10.7.8.9" } ]

In this case, the primary IP would be `10.1.2.3`. If the administrator
wants a different primary IP, they can pass `--node-ip` to choose
another one, but that will have the side effect of removing all other
IPs of the same type from the list. Eg, passing `--node-ip 10.4.5.6`,
would result in `Node.Status.Addresses` having the value:

    [ { "type": "ExternalIP", "address": "42.42.42.42" },
      { "type": "InternalIP", "address": "10.4.5.6" } ]

When using an external cloud-provider, the administrator has no
control over the node IPs; in the above example, passing `--node-ip
10.4.5.6` would have no effect on the returned list of addresses, and
the primary IP would still be `10.1.2.3`. (Although passing `--node-ip
10.11.12.13` would result in an error since that doesn't appear in the
list at all.)

#### API Cleanliness

`Node.Status.Addresses` is annotated with an incorrect strategic-merge
patch annotation, and so it is not possible to use strategic merge
patch to change that field without losing information. (See discussion
in [kubernetes #79391].) Although kubelet and cloud-controller-manager
have been fixed to work around the problem, it is still annoying to
have the incorrect annotation there, and it is possible that other
components might attempt to update the field incorrectly in the
future.

Additionally, `Node.Status.Addresses` is not currently sufficiently
checked by validation; it is possible for an external cloud provider
to declare an `InternalIP` or `ExternalIP` address that is not a
syntactically-valid IP address, meaning clients that read this field
may need to sanity-check it themselves ([kubernetes #89817]). Adding
validation now could break deployed cloud providers (which might
otherwise appear to be fully-functional, as long as the primary IP was
valid).

[kubernetes #79391]: https://github.com/kubernetes/kubernetes/pull/79391
[kubernetes #89817]: https://github.com/kubernetes/kubernetes/pull/89817

#### `NodeAddressType` Semantics

The difference between `InternalIP` and `ExternalIP` node addresses is
not well-defined except in the case of IPv4 addresses in public
clouds, and the two types are used inconsistently outside of that
environment. (eg, See [kubernetes #86918] as an example of uncertainty
about the right type to use in a particular situation.)

[kubernetes #86918]: https://github.com/kubernetes/kubernetes/pull/86918#discussion_r363997750

### Goals

- Deprecate `Node.Status.Addresses` and replace it with new fields
  that have clearer semantics, and do not have the strategic-merge and
  validation problems.

- Define clearly what "the primary Node IP(s)" is/are, and ensure that
  the bare-metal code, legacy cloud provider code, and
  cloud-controller-manager configure/set Node IPs in a consistent way
  that allows correctly configuring single stack IPv4, single-stack
  IPv6, and dual-stack clusters.

### Non-Goals

- Requiring cloud providers to implement IPv6 / dual-stack if they
  don't already.

- Updating `Pod.Status.HostIP` for dual-stack. It might be useful to
  add `HostIPs` as with `PodIPs` but that is not entirely necessary
  (we can set dual-stack `PodIPs` on host-network pods without needing
  to also have dual-stack `HostIPs`), and could be done later.

### Maybe Goals

- Being able to set multiple different kinds of IPs (eg, external vs
  internal) on bare-metal nodes ([kubernetes #42125]). This is
  potentially distinct from being able to set a dual-stack pair of
  `InternalIP` addresses.

- Being able to declare node IPs as being for specific purposes (eg,
  "data plane", "control plane")

- Improving error reporting in clusters with inconsistent IP families
  (eg, dual-stack cluster CIDR but single-stack node IPs).

[kubernetes #42125]: https://github.com/kubernetes/kubernetes/issues/42125

## Proposal

Due to the existing problems with `Node.Status.Addresses`, we will add
two new fields, `Node.Status.Hostname` (which will contain the same
value as the `NodeHostName`-type `Node.Status.Addresses` element) and
`Node.Status.IPs` (which will contain more-or-less the information
from the other `Node.Status.Addresses` elements, but slightly
reorganized and clarified).

### Notes/Constraints/Caveats

#### Current Behavior

`Node.Status.Addresses` currently always contains exactly one element
of type `NodeHostName`. Kubelet autodetects this value by calling
`os.Hostname()`, but allows the administrator to override it with the
`--hostname-override` flag. This value is then set on the
`"kubernetes.io/hostname"` annotation on the Node. The cloud provider
can choose to either use this value as `NodeHostName` or to return its
own value.

```
<<[UNRESOLVED whats-in-a-hostname ]>>
What does this get used for?

  - kube-apiserver's --kubelet-preferred-address-types starts with
    "Hostname" by default, meaning apiserver will try to DNS-resolve
    that value first when trying to reach a kubelet.

  - pkg/kubelet/certificate/kubelet.go will include the NodeHostName
    as a DNS name in the certificate request (along with any
    NodeInternalDNS or NodeExternalDNS names)

That's all I could find in k/k. It is *not* used to set
Node.ObjectMeta.Name (though in many cases they end up being the same
anyway).

GCE and AWS always return NodeAddresses where the NodeHostName value
is a duplicate of a NodeInternalDNS value, but Azure, vSphere, and
OpenStack do not (and the bare-metal case currently doesn't allow for
providing any DNS names). So deprecating NodeHostName addresses
entirely (ie, without adding Node.Status.Hostname) would potentially
lose a small amount of information.
<<[/UNRESOLVED]>>
```

The remaining `Node.Status.Addresses` values are either IP addresses or
DNS names, which can both be either "Internal" or "External".

[The documentation of node addresses] states that "the usage of these
fields varies depending on your cloud provider or bare metal
configuration" but suggests that "External" addresses are "typically the
IP address of the node that is externally routable (available from
outside the cluster)" and "Internal" addresses are "typically the IP
address of the node that is routable only within the cluster".

Beyond that there is also somewhat of a consensus that "Internal" IPs
route directly to the node, while "External" IPs may pass through some
sort of address-rewriting infrastructure (and thus be less efficient
for node-to-node communication). That is, "Internal" and "External"
are understood to map directly to the "internal"/"private" vs
"external"/"public" IPv4 addresses in most public cloud APIs.

For IPs that don't clearly obey this distinction there is not currently
consensus on how they should be labelled:

  - Clouds that support IPv6 may provide addresses which are both
    globally accessible and directly routed; calling them either
    Internal or External risks having some components misinterpret their
    semantics.

  - The vSphere provider claims that all IPs that it knows about are
    *both* "Internal" and "External".

  - In bare-metal environments, nodes have only a single IP, and that IP
    is always called "Internal".

[The documentation of node addresses]: https://kubernetes.io/docs/concepts/architecture/nodes/#addresses

#### Backward Compatibility

`Node.Status.Addresses` needs to stick around as long as `v1.Node`
does, but it can be essentially frozen with its current semantics.
Existing cloud providers may continue to set `Node.Status.Addresses`
with values that are not entirely equivalent to the `Node.Status.IPs`
values, for backward compatibility. For new cloud providers we may be
able to have `Node.Status.Addresses` be automatically generated from
`Node.Status.IPs` and `Node.Status.Hostname` by
cloud-controller-manager.

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
kubernetes ecosystem.

How will security be reviewed and by whom?

How will UX be reviewed and by whom?

Consider including folks that also work outside the SIG or subproject.
-->

```
<<[UNRESOLVED inconsistency ]>>
Security issues related to Node.Status.Addresses and Node.Status.IPs
containing different values?
<<[/UNRESOLVED]>>
```

## Design Details

### Implementation

#### Hostname

With the deprecation of `Node.Status.Addresses`, this value will be
moved to `Node.Status.Hostname`, but does not otherwise need to be
changed.

```
<<[UNRESOLVED hostname-label ]>>
Is it a bug that `Node.Status.Hostname`/`NodeHostName` and the
`"kubernetes.io/hostname"` label don't necessarily match? (NB: This
is an existing behavior.)
<<[/UNRESOLVED]>>
```

#### IPs

The other information in `Node.Status.Addresses` will be moved to a
new `Node.Status.IPs` array:

    type NodeStatus struct {
            ...

            // List of the node's IPs. The first IP in the list is the "primary" node IP
            // (eg, the hostIP value of pods on that node). If the second IP is of the
            // opposite IP address family from the first (eg, IPv6 vs IPv4), then it is
            // considered the primary IP of that address family. Other than that, the
            // order of elements carries no meaning. Note that there may be a mix of IPv4
            // and IPv6 addresses here regardless of whether the cluster supports both
            // address families.
            IPs []NodeIP `json:"ips"`

            ...
    }

(Unlike with `Addresses`, this has no declared `patchStrategy` and thus
defaults to `replace`, which is correct since it is solely maintained by
a single component (kubelet or cloud-controller-manager), which always
updates the entire field at once.)

```
<<[UNRESOLVED primary-ip ]>>
If we're tightening up the rules on how cloud providers set addresses
then it seems to make sense to just require them to put the primary IP
first (as the comment above does) rather than having some complicated
"first Internal IP unless there are no Internal IPs in which case..."
rule like we have now.

Instead of "If the second IP is of the opposite address family from
the first, then it is considered the primary IP of that address
family" we could instead do "If there are any IPs of the opposite
address family from the first IP, then the first such IP is considered
the primary IP of that address family" (which would, eg, let you put
the IPv4 External IPs before the IPv6 Internal IPs if you wanted,
etc). It seems like that just makes life more complicated for everyone
else though?
<<[/UNRESOLVED]>>
```

    // NodeIP provides information about a single Node IP address.
    type NodeIP struct {
            // IP is a node IP address
            IP string `json:"ip"`

            // Scope gives the scope of IP
            // +optional
            Scope NodeIPScope `json:"scope,omitempty"`

            // DNSName is an optional DNS name that resolves to IP
            // +optional
            DNSName string `json:"dnsName,omitempty"`
    }

    // NodeIPScope describes the scope within which a Node IP is useful.
    type NodeIPScope string

    const (
            // NodeIPUnknown indicates an IP address with unknown scope. (It is at least
            // reachable by all other nodes within the cluster, but may or may not be
            // accessible beyond that.)
            NodeIPUnknown NodeIPScope = ""

            // NodeIPInternal indicates an IP address that exists only within the scope of
            // some private network (eg, an intranet or cloud VPC). The scope is at least
            // large enough to include all other nodes within the cluster.
            NodeIPInternal NodeIPScope = "Internal"

            // NodeIPExternal indicates an IP address that exists only outside the cluster
            // (eg, traffic sent to this address from inside the cluster may leave the
            // cluster and then come back with the destination IP rewritten to an Internal
            // IP address.)
            NodeIPExternal NodeIPScope = "External"

            // NodeIPPublic indicates an IP address that is expected to be directly
            // reachable from anywhere, inside or outside the cluster.
            NodeIPPublic NodeIPScope = "Public"
    )

Rather than having separate `NodeInternalIP`/`NodeInternalDNS` and
`NodeExternalIP`/`NodeExternalDNS` entries, the DNS entries are now
included along with the matching IPs.

```
<<[UNRESOLVED dns-correlation ]>>
It seems that most clouds' APIs return IPs and DNS names via separate
API calls, so correlating them will require extra work (eg, DNS
lookups) and in some cases a cloud might currently be returning DNS
names that don't correspond to any of the IPs it returns (which would
not be possible with `NodeIPs`). Is this a problem?
<<[/UNRESOLVED]>>
```

Additionally, the `InternalIP` vs `ExternalIP` split is now clarified
and expanded to cover additional cases. For the most part, existing
cloud `InternalIP` addresses would translate to `Internal` scope and
`ExternalIP` addresses would translate to `External` scope, but IPv6
node IPs will most likely be `Public`, and bare-metal IPs will generally
be `Unknown`. (Private cloud providers (eg, vSphere and OpenStack) might
also use `Unknown` scope.)

```
<<[UNRESOLVED other-info ]>>
Do we want other information in `NodeIP`? eg, we could indicate which
network interface is associated with each IP, or more abstractly allow
there to be "tags" of some sort ("control plane", "storage
network").

Of course more fields could be added to `NodeIP` later.
<<[/UNRESOLVED]>>
```

#### `--node-ip` and Address Sorting

To allow users to override both IPv4 and IPv6 IPs in dual-stack
clusters, kubelet's `--node-ip` argument will be extended to allow a
comma-separated list of IPs.

Additionally, the `"alpha.kubernetes.io/provided-node-ip"` annotation
that kubelet uses to communicate `--node-ip` to cloud-controller-manager
will be extended/replaced to provide the extended information.

Cloud-controller-manager will be updated to sort `Node.Status.IPs` in
the same way that kubelet does when `--node-ip` is passed. (They should
probably use shared code in `pkg/util/node` to do this.)

Additionally/optionally, we could allow the user to pass a full
serialized JSON `[]NodeIP` value to `--node-ip`. In the cloud case,
this would still only be for sorting purposes, not for defining new
IPs, so the data provided would still mostly have to match the data
from the cloud provider. However, in the bare-metal case, this could
be used to clarify which IPs are internal vs external, to provide DNS
names, etc.

#### Updating Current Node.Status.Addresses Users

```
<<[UNRESOLVED current-users ]>>
See who currently uses `Node.Status.Addresses`, figure out their
behavior going forward. Eg, kube-apiserver's
`--kubelet-preferred-address-types` arg.
<<[/UNRESOLVED]>>
```

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy.  Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).  Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

This is impossible to do full e2e testing of since that would require
setting up multiple cloud environments with a mix of IPv4, IPv6, and
dual-stack. We can add more unit tests though...

TODO

### Graduation Criteria

<!--
**Note:** *Not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. The KEP
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this enhancement:
- [Maturity levels (`alpha`, `beta`, `stable`)][maturity-levels]
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc
definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, GA), regardless how the
functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

Below are some examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

#### Alpha -> Beta Graduation

- Gather feedback from developers and surveys
- Complete features A, B, C
- Tests are in Testgrid and linked in KEP

#### Beta -> GA Graduation

- N examples of real world usage
- N installs
- More rigorous forms of testing e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least 2 releases between beta and
GA/stable, since there's no opportunity for user feedback, or even bug reports,
in back-to-back releases.

#### Removing a deprecated flag

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality which deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag

**For non-optional features moving to GA, the graduation criteria must include [conformance tests].**

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md
-->

TODO

### Upgrade / Downgrade Strategy

Currently, `kubelet --node-ip` has more-or-less no effect when using an
external cloud provider. (It will require that the IP you pass is a
known node IP, but it does not do anything else.) Assuming we change it
to behave the same with external cloud providers as it does with legacy
cloud providers and bare metal (ie, letting the user choose which node
IP should be primary), then some users who were passing `--node-ip`
before would need to stop passing it when upgrading, to avoid getting
the new behavior. Perhaps this should warn and continue to use the old
behavior during alpha, then warn and use the new behavior in beta.

(Alternatively, since we need to extend `--node-ip` to support dual
stack anyway, we could deprecate `--node-ip` and replace it with
`--node-ips`, where the old `--node-ip` flag would continue to have the
same semantics on external clouds until it went away.)

Users on bare metal who upgrade and then convert their cluster from
single-stack to dual-stack might need to switch back to single-stack
first if they later needed to downgrade, due to the loss of some
bare-metal-related dual-stack functionality when downgrading.

External programs, scripts, etc that currently look at
`Node.Status.Addresses` will eventually need to be updated to look at
the new fields instead.

### Version Skew Strategy

`Node.Status.Addresses` can't actually go away until some hypothetical
future `corev2.Node`, so older components will still be able to get
all of the old information from there in a skewed cluster.

During the skew period, components would not be able to assume
unconditionally that `Node.Status.IPs` would be set, since they might
be running against an old kubelet or cloud-controller-manager. (This
includes kubelet itself, in the case of a new kubelet and an old
cloud-controller-manager.)

## Implementation History

<!--
Major milestones in the life cycle of a KEP should be tracked in this section.
Major milestones might include
- the `Summary` and `Motivation` sections being merged signaling SIG acceptance
- the `Proposal` section being merged signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

## Alternatives

Instead of deprecating `Node.Status.Addresses`, we could just clarify
its semantics instead. However, this would not fix the "API Cleanliness"
problems.

Additionally, if we want to make any changes to the semantics of the
addresses (eg, "you shouldn't declare a single IP as both internal and
external") then there would have to be some way for cloud providers to
move from the old semantics to the new ones without breaking existing
users. Adding a new field instead lets cloud providers continue to use
the wrong/old semantics in `Node.Status.Addresses`.

Some of the problems with the current system could be resolved by saying
that clients should not read `Node.Status.Addresses` directly, but
should rely on methods in apimachinery instead, where those methods have
additional intelligence. (eg, instead of assuming that the first-listed
address is the primary address, there would be a `GetNodePrimaryIP()`
method, which knew to take into account that in the external cloud
provider case, the first-listed address might be of the wrong IP
family.)
