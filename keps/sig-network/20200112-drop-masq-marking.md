---
title: IPTables "Drop" and "Masquerade" Marking
authors:
  - "@danwinship"
owning-sig: sig-network
reviewers:
  - "@thockin"
approvers:
  - "@thockin"
editor: TBD
creation-date: 2020-01-12
last-updated: 2020-01-12
status: provisional
---

# IPTables "Drop" and "Masquerade" Marking

1. **Fill out the "overview" sections.**
  This includes the Summary and Motivation sections.
  These should be easy if you've preflighted the idea of the KEP with the appropriate SIG.
1. **Create a PR.**
  Assign it to folks in the SIG that are sponsoring this process.
1. **Create an issue in kubernetes/enhancements, if the enhancement will be targeting changes to kubernetes/kubernetes**
  When filing an enhancement tracking issue, please ensure to complete all fields in the template.
1. **Merge early.**
  Avoid getting hung up on specific details and instead aim to get the goal of the KEP merged quickly.
  The best way to do this is to just start with the "Overview" sections and fill out details incrementally in follow on PRs.
  View anything marked as a `provisional` as a working document and subject to change.
  Aim for single topic PRs to keep discussions focused.
  If you disagree with what is already in a document, open a new PR with suggested changes.

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories [optional]](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Implementation Details/Notes/Constraints [optional]](#implementation-detailsnotesconstraints-optional)
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

Kubelet and kube-proxy create iptables rules to mark incoming packets
that should be dropped, and to mark outgoing packets that should be
masqueraded (along with other rules to actually drop/masquerade them
later). However, this is currently done in a disorganized and
confusing way, and the current implementation uses a limited resource
(the packet mark) in a way that makes it difficult for other parts of
the system to share.

This KEP proposes an updated packet marking system, and a plan to
migrate from the old one to the new one.

## Motivation

### Background

IPTables only allows sending packets to the `DROP` and `MASQUERADE`
chains at certain stages in the packet handling process. In some
cases, Kubernetes creates rules that will decide that a packet needs
to be dropped or masqueraded, but that rule is running in one of the
processing stages that is not allowed to drop or masquerade the packet
directly. The solution we use is that the earlier rules mark the
packets as needing to be dropped/masqueraded, and then a later rule
sees the mark and acts on it when it is allowed to do so.

There are a few problems with the way this is currently implemented:

  1. kubelet and kube-proxy both independently create the
     masquerade-handling rules, which means that the code needs to be
     kept in sync between the two (eg, as seen in [kubernetes
     #78547]), and there could be problems during an upgrade when
     running different versions of kubelet and kube-proxy that expect
     different rules. Additionally, since the mark bit is
     configurable, this requires that the kubelet and kube-proxy
     configuration be kept in sync, or else they will end up writing
     potentially-conflicting rules.

  2. The drop-handling rules are created only by kubelet, but used
     only by kube-proxy, which creates unnecessary coupling between
     the two components. (eg, [kubernetes #82214], as-yet-unfixed
     dual-stack bug noted in [kubernetes #82125])

  3. The packet marking is currently implemented by using the iptables
     `mark` module to set a single bit in the `skb_mark` field for the
     packets in question. This means any other software on the system
     that wants to use the `skb_mark` can only use a subset of it (and
     needs to be configured to know which subset it can use). (eg,
     [openshift/origin #18121])

  4. Because of the way that `skb_mark` works in the kernel, in some
     cases, when a packet is encapsulated (eg, in a VXLAN tunnel, to
     send to another node), a mark that was intended only for the
     inner packet will also be seen on the outer packet, or vice
     versa. ([kubernetes #78948])

Though not a "problem" per se, it is also important to keep in mind
that the current `KUBE-MARK-MASQ` and `KUBE-MARK-DROP` tables are
treated like an API by some users. For example, the [CNI portmap
plugin] provides an `"externalSetMarkChain"` option that is intended
to be used with `"KUBE-MARK-MASQ"`, to make the plugin use
Kubernetes's iptables rules instead of creating its own.

[kubernetes #78547](https://github.com/kubernetes/kubernetes/issues/78547)
[kubernetes #78948](https://github.com/kubernetes/kubernetes/issues/78948)
[kubernetes #82125](https://github.com/kubernetes/kubernetes/issues/82125#issuecomment-598049271)
[kubernetes #82214](https://github.com/kubernetes/kubernetes/issues/82214)
[openshift/origin #18121](https://github.com/openshift/origin/pull/18121)
[CNI portmap plugin](https://github.com/containernetworking/plugins/tree/master/plugins/meta/portmap)

### Goals

1. Fix the mark-leaking problem with encapsulated packets.

2. Reduce coupling between kubelet and kube-proxy.

3. Free up the packet mark for use by the network plugin or other
components.

4. Provide clearer guidance about external use of Kubernetes's
iptables chains.

### Non-Goals

## Proposal

### Implementation Details/Notes/Constraints [optional]

1. We should move from using the iptables `mark` module to the
`connlabel` module; the connlabel is a value that is interpreted as
128 separate 1-bit labels, rather than as a single 32-bit integer,
making it a better fit for what we are doing. Additionally, using
`connlabel` rather than `mark` solves the mark-leaking problem with
encapsulated packets.

2. Kubelet and kube-proxy should each create their own set of marking
and handling rules, with their own `connlabel` bits, and these should
be considered private to kubelet and kube-proxy; other software that
needs similar functionality should create their own rules as well.
(Can we really say that? How do we communicate this?)

3. We should continue to create the existing `mark`-based rules for
several more releases, to support upgrades and to support external
users of those rules until they migrate away from them.





What are the caveats to the implementation?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they releate.

### Risks and Mitigations

What are the risks of this proposal and how do we mitigate.
Think broadly.
For example, consider both security and how this will impact the larger kubernetes ecosystem.

How will security be reviewed and by whom?
How will UX be reviewed and by whom?

Consider including folks that also work outside the SIG or subproject.

- `connlabel` is not as widely used as `mark` and may turn out to have
  unexpected problems.

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy.
Anything that would count as tricky in the implementation and anything particularly challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage expectations).
Please adhere to the [Kubernetes testing guidelines][testing-guidelines] when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial KEP should keep
this high-level with a focus on what signals will be looked at to determine graduation.

Consider the following in developing the graduation criteria for this enhancement:
- [Maturity levels (`alpha`, `beta`, `stable`)][maturity-levels]
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, GA), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

#### Examples

These are generalized examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

##### Alpha -> Beta Graduation

- Gather feedback from developers and surveys
- Complete features A, B, C
- Tests are in Testgrid and linked in KEP

##### Beta -> GA Graduation

- N examples of real world usage
- N installs
- More rigorous forms of testing e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least 2 releases between beta and GA/stable, since there's no opportunity for user feedback, or even bug reports, in back-to-back releases.

##### Removing a deprecated flag

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality which deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag

**For non-optional features moving to GA, the graduation criteria must include [conformance tests].**

[conformance tests]: https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing cluster required to make on upgrade in order to make use of the enhancement?

### Version Skew Strategy

If applicable, how will the component handle version skew with other components? What are the guarantees? Make sure
this is in the test plan.

Consider the following in developing a version skew strategy for this enhancement:
- Does this enhancement involve coordinating behavior in the control plane and in the kubelet? How does an n-2 kubelet without this feature available behave when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI or CNI may require updating that component before the kubelet.

## Implementation History

Major milestones in the life cycle of a KEP should be tracked in `Implementation History`.
Major milestones might include

- the `Summary` and `Motivation` sections being merged signaling SIG acceptance
- the `Proposal` section being merged signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded

## Drawbacks [optional]

Why should this KEP _not_ be implemented.

## Alternatives [optional]

Similar to the `Drawbacks` section the `Alternatives` section is used to highlight and record other possible approaches to delivering the value proposed by a KEP.
