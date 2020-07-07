---
title: Cost based scaling down of pods
authors:
  - "@ingvagabund"
owning-sig: sig-scheduling
participating-sigs:
  - sig-apps
reviewers:
  - TBD
approvers:
  - TBD
editor: TBD
creation-date: 2020-06-30
last-updated: yyyy-mm-dd
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
---

# Cost based scaling down of pods

To get started with this template:
1. **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking up.
  KEPs should not be checked in without a sponsoring SIG.
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

The canonical place for the latest set of instructions (and the likely source of this file) is [here](/keps/YYYYMMDD-kep-template.md).

The `Metadata` section above is intended to support the creation of tooling around the KEP process.
This will be a YAML section that is fenced as a code block.
See the KEP process for details on each of these items.

## Table of Contents

A table of contents is helpful for quickly jumping to sections of a KEP and for highlighting any additional information provided beyond the standard KEP template.

Ensure the TOC is wrapped with <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code> tags, and then generate with `hack/update-toc.sh`.

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

Cost based scaling down of pods allows to employ various scheduling strategies
to keep a cluster from diverging from an optimal distribution of resources.
Providing a solution for selecting the right victim allows to improve ability
to preserve various conditions such us balancing pods among failure domains, keeping
aligned with security requirements or respecting application policies.

Allowing controllers to be free of any scheduling strategy, yet to be aware
of impact of removing pods on the overall cluster scheduling plan, helps to reduce
a cost of re-scheduling resources.

## Motivation

Scaling down a set of pods does not always result in optimal selection of victims.
The scheduler relies on filters and scores which may distribute the pods wrt. topology
spreading and/or load balancing constraints (e.g. pods uniformly balanced among zones).
Application specific workloads may prefer to scale down short-running pods and favor long-running pods.
Selecting a victim with a trivial logic can unbalance the topology spreading
or have jobs that accumulated work to be lost in vain.
Given it's a natural property of a cluster to shift workloads in time,
decision made by a scheduler is as good as its ability to predict future demands.
The default kubernetes scheduler was constructed with a goal to provide high throughput
at the cost of being simple. Thus, it is quite easy to diverge from the scheduled plan.
In contrast, descheduler allows to help to re-balance the plan and get closer to
the scheduler constraints. Yet, it is designed to run and adjust the cluster periodically (e.g. each hour).
Therefor, unusable for scaling down purposes (which require immediate action).

In order to support more informed scaling down operation, additional decision logic
is required.

### Goals

- Controllers with scale down operation are allowed to select a victim while still respecting a scheduling plan

### Non-Goals

NONE

## Proposal

Each controller with a scale down operation has its own implementation of a victim selection logic.
The decision making logic does not take into account a scheduling plan.
Extending each such controller with additional logic to support various scheduling
constraints is impractical. In cases a proprietary solution for scaling down is required,
it's impossible. Also, controllers do not necessarily have a whole cluster overview
so its decision does not have to be optimal.
Therefor, it's more feasible to locate the logic outside of a controller.

Proposed solution is to implement an optional cost-based component that will be watching all
pods (or its subset) present in a cluster and assigning each one a cost based on a set of scheduling constraints.
The component will allow for each targeted set of pods to select a different list of scheduling
constraints. Each pod in a set will be given a cost based on how much important it is in the set.
The constraints can follow the same rules as the scheduler (through importing scheduling plugins)
or be custom made (e.g. wrt. to application or proprietary requirements).
The component will either annotate a pod or update its status.
Each controller will have a choice to either ignore the cost or take it into account
when scaling down.

This way, the logic for selecting a victim for the scaling down operation will be
separated from each controller. Allowing each consumer to provide its own
logic for assigning costs. Yet, having all controllers to consume the cost uniformly.

Given the default scheduler is not a source of truth about how a pod should be distributed
after it was scheduled, scaling down strategies can exercise completely different approaches.

Examples of scheduling constraints:
- choose pods running on a node which have a `PreferNoSchedule` taint first
- choose youngest/oldest pods first
- choose pods to respect topology skew among failure domains (e.g. availability zones)

### User Stories [optional]

#### Story 1

From [@pnovotnak](https://github.com/kubernetes/kubernetes/issues/4301#issuecomment-328685358):

```
I have a number of scientific programs that I've wrapped with code to talk
to a message broker that do not checkpoint state. The cost of deleting the resource
increases over time (some of these tasks take hours), until it completes the current unit of work.

Choosing a pod by most idle resources would also work in my case.
```

#### Story 2

From [@cpwood](https://github.com/kubernetes/kubernetes/issues/4301#issuecomment-436587548)

```
For my use case, I'd prefer Kubernetes to choose its victims from pods that are running on nodes which have a PreferNoSchedule taint.
```

### Implementation Details/Notes/Constraints [optional]

Currently, the descheduler does not allow to immediately react on changes in a cluster.
Yet, with some modification another instance of the descheduler (with different set of strategies)
might be ran in watch mode and rank each pods as it comes.
Also, once the scheduling framework gets migrated into its own repository,
scheduling plugins can be vendored as well to provide some of the core scheduling logic.

### Risks and Mitigations

It may happen the ranking component does not rank all relevant pods in time.
In that case a controller can either choose to ignore the cost. Or, it can back-off
with a configurable timeout and retry the scale down operation once all pods in
a given set are ranked.

From the security perspective a malicious code might assign pod a different cost
with a goal to remove more vital pods to harm a running application.
How much is using annotation safe? Might be better to use pod status
so only clients with pod/status update RBAC are allowed to change the cost.

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

[conformance tests]: https://github.com/kubernetes/community/blob/master/contributors/devel/conformance-tests.md

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

## Infrastructure Needed [optional]

Use this section if you need things from the project/SIG.
Examples include a new subproject, repos requested, github details.
Listing these here allows a SIG to get the process for these resources started right away.
