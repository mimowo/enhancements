<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up. KEPs should not be checked in without a sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please make sure to complete all
  fields in that template. One of the fields asks for a link to the KEP. You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [ ] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "Title", "Authors", "Owning-sig",
  "Status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [ ] **Create a PR for this KEP.**
  Assign it to people in the SIG who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved. Any KEP
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One KEP corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example. If
new details emerge that belong in the KEP, edit the KEP. Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the KEP approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting KEPs).
-->
# KEP-3329: Retriable and non-retriable Pod failures for Jobs

<!--
This is the title of your KEP. Keep it short, simple, and descriptive. A good
title can help communicate what the KEP is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a KEP and for
highlighting any additional information provided beyond the standard KEP
template.

Ensure the TOC is wrapped with
  <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code>
tags, and then generate with `hack/update-toc.sh`.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests for meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This KEP extends Kubernetes to determine if a pod failure of a job is retriable
or not. Pod failures determined as non-retriable will result in early job termination.

This is needed in order to save time and computational resources wasted due
to unncessary retries of containers which are destined to fail due to
software bugs.

<!--
This section is incredibly important for producing high-quality, user-focused
documentation such as release notes or a development roadmap. It should be
possible to collect this information before implementation begins, in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself. KEP editors and SIG Docs
should help to ensure that the tone and content of the `Summary` section is
useful for a wide audience.

A good summary is probably at least a paragraph in length.

Both in this section and below, follow the guidelines of the [documentation
style guide]. In particular, wrap lines to a reasonable length, to make it
easier for reviewers to cite specific portions, and to minimize diff churn on
updates.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md
-->

## Motivation

Running a large computational workload, comprising thousands of pods on
thousands of nodes requires usage of pod restart policies in order
to account for infrastructure failures.

Currently, kubernetes Job API offers a way to account for infrastructure
failures by setting `.backoffLimit > 0`. However, this mechanism intructs the
job controller to restart all failed pods - regardless of the root cause
of the failures. Thus, in some scenarios this leads to unnecessary
restarts of many pods, resulting in a waste of time and computational
resources. What makes the restarts more expensive is the fact that the
failures may be encountered late in the execution time of a program.

Sometimes it can be determined from containers exit codes
that the root cause of a failure is in the executable and the
job is destined to fail regardless of the number of retries. However, since
the large workloads are often scheduled to run over night or over the
weekend, there is no human assistance to terminate such a job early.

The need for solving the problem has been emphasized by the kubernetes
community in the issues, see: [#17244](https://github.com/kubernetes/kubernetes/issues/17244) and [#31147](https://github.com/kubernetes/kubernetes/issues/31147).

Some third-party frameworks have implemented retry policies for their pods:
- [TensorFlow Training (TFJob)](https://www.kubeflow.org/docs/components/training/tftraining/)
- [Argo workflow](https://github.com/argoproj/argo-workflows/blob/master/examples/retry-conditional.yaml) ([example](https://github.com/argoproj/argo-workflows/blob/master/examples/retry-conditional.yaml))

Additionally, some pod failures are not linked with the container execution,
but rather with the internal kubernetes cluster management (see:
[Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)).
Such pod failures should be recongnized as infrastructure failures and should
not increment the counter for `backoffLimit`.

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this KEP.  Describe why the change is important and the benefits to users. The
motivation section can optionally provide links to [experience reports] to
demonstrate the interest in a KEP within the wider Kubernetes community.

[experience reports]: https://github.com/golang/go/wiki/ExperienceReports
-->

### Goals

- Extension of Job API with user-friendly syntax to terminate jobs based on:
  * container exit codes
  * `status.reason` field of a failed pod.
- Extensible API which will allow to implement support for other indicators
  of non-retriable jobs in the future, for example based on the termination
  logs (see: [Determine the Reason for Pod Failure](https://kubernetes.io/docs/tasks/debug/debug-application/determine-reason-pod-failure/)).

<!--
TODO: consider adding the point:
- Review of the places where the `DeleteOptions` structure is used to delete
  pods and specification of the `reason` field in these contexts.
-->
<!--
List the specific goals of the KEP. What is it trying to achieve? How will we
know that this has succeeded?
-->

### Non-Goals

- Implementation of other indicators of non-retirable jobs such as termination logs.
- Modificaiton of the semantics for job termination. In particular, allowing for
  all indexes of an indexed-job to execute when only one or a few indexes fail
  [#109712](https://github.com/kubernetes/kubernetes/issues/109712).

<!--
TODO: consider adding the point:
- Specification of the `reason` field in the `DeleteOptions` structure when
  it is used to delete other kubernetes resources than pods.
-->

<!--
What is out of scope for this KEP? Listing non-goals helps to focus discussion
and make progress.
-->

## Proposal

Extension of the Job API with a new field which allows to configure the set of
conditions which determine how a pod failure is handled. We extend the Job API
to support discrimination of pod failures based on container exit codes as well
as the pod's `status.reason` field.

<!--
The value of status.reason I found used by k8s currently is:
	// NodeUnreachablePodReason is the reason on a pod when its state cannot be confirmed as kubelet is unresponsive
	// on the node it is (was) running.
NodeUnreachablePodReason = "NodeLost" (defined on node.go)
-->
The pod's `status.reason` field is currently defined but used in few situations.
We extend its use to store the reason of a pod's failure when originated by
a kubernetes component (kube-scheduler, kubelet, pod-gc, etc.). To achieve this
we extend the Pod delete API to pass the reason for pod termination. Such
failures can typically be associated with some cluster-management actions and
should be retried.

Additionally, we review all the currently existing places were the delete API
is invoked to delete pods in order to modify the code to pass a meaningful
reason based on the invocation context.

We use the main job controller main loop to detect and categorize the pod failures
with respect to the configuration. For each failed pod, one of the following
actions is applied:
- terminate the job (`non-retriable` failure),
- ignore the failure (`retriable` failure) - restart the pod and do not increment
  the counter for `backoffLimit`,
- increment the `backoffLimit` counter and restart the pod if the limit is not
  reached (current behaviour).

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

### User Stories (Optional)

<!--
Detail the things that people will be able to do if this KEP is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

As a machine learning researcher, I run jobs comprising thousands
of long-running pods on a cluster comprising thousands of nodes. The jobs often
run at night or over weekend without any human monitoring. In order to account
for random infrastructure failures we define `.backoffLimit: 6` for the job.
However, a signifficant portion of the failures happen due to bugs in code.
Moreover, the failures may happen late during the program execution time. In
such case, restarting such a pod results in wasting a lot of computational time.

We would like to be able to automatically detect and terminate the jobs which are
failing due to the bugs in code of the executable, so that the computation resources
can be saved.

Occasionally, our executable fails, but it can be safely restarted with a good
chance of succeeding the next time. In such known retriable situations our
executable exits with a dedicated exit code in the 40-50 range. All remaining
exit codes indicate a software bug and should result in an early job termination.

The following Job configuration could be a good starting point to satisfy
my needs:

```yaml
apiVersion: v1
kind: Job
spec:
  template:
    spec:
      containers:
      - name: job-container
        image: job-image
        command: ["./program"]
  backoffLimit: 3
  backoffPolicy:
    rules:
    - containerExitCodeNotIn: 40-50
      action: Terminate
```

#### Story 2

As a service provider that offers computational resources to researchers I would like to
have a mechanism which terminates jobs for which pods are failing due to user errors,
but allows infinite retries for infrastructure errors (pod terminations by kubelet,
kube-scheduler, abnormal node termination, etc).
I do not have knowledge or influence over the executable that researchers run,
so I don't know beforehand which exit codes they might return.

The following Job configuration could be a good starting point to satisfy
my needs:

```yaml
apiVersion: v1
kind: Job
spec:
  template:
    spec:
      containers:
      - name: main-job-container
        image: job-image
        command: ["./program"]
      - name: monitoring-job-container
        image: job-monitoring
        command: ["./monitoring"]
  backoffLimit: 3
  backoffPolicy:
    rules:
    - podStatusReasonIn:
      - Preemption
      - NodePressureEviction
      action: Ignore
```

Note that, in this case the user supplies a list of pod's `status.reason` values
which are interpreted as infrastructure failures. This approach is likely to
require an iterative process to review and extend of the list. In order to
avoid the process some users may prefer to interpret any non-empty `status.reason`
as an infrastructure failure this can be achieved with the following syntax for
`backoffPolicy`:

```yaml
  backoffPolicy:
    rules:
    - podStatusReasonNotIn:
      - ''
      action: Ignore
```

### Notes/Constraints/Caveats (Optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

<!-- Increased complexity of the Job management, by interaction with some of
already exiting Job configuration options. To mitigate this, we introduce
minimal API which will allow to satisfy the main usage scenario.-->

The Pod status (which includes exit codes) could be lost if the failed pod is garbage collected.
First, it should be rather rare so the overall consequence of unnecessary pod restarts
will be limited. Second, we can prevent this by using the feature of
[job tracking with finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/).

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->

## Design Details

### Reason in DeleteOptions

The `DeleteOptions` is extended by the reason field to convey the reason of
deletion.

```golang
// DeleteOptions may be provided when deleting an API object.
type DeleteOptions struct {
  ...
	// A brief CamelCase message indicating the reason why the object is deleted (example: "Preemption").
	// + optional
	Reason *string `json:"reason,omitempty" protobuf:"bytes,6,rep,name=reason"`
}
```

The `reason` field will be used by kubernetes components (such as:
kube-scheduler, kubelet pod-gc) when they send the delete operation. During
the implementation process, we are going to review the places were the
`DeleteOptions` is used to delete pods and add the `reason` field based on the
invocation context.

Notably, the `reason` field might be useful to be supplied when deleting other
types of kubernetes resources, but supplying the field in other contexts is
beyond this KEP.

In the case of Pods, kube-apiserver will store the received `reason` in the
already existing `reason` field in the `PodStatus`:
```
// PodStatus represents information about the status of a pod. Status may trail the actual
// state of a system.
type PodStatus struct {
  ...
	// A brief CamelCase message indicating details about why the pod is in this state. e.g. 'Evicted'
	// +optional
	Reason string
  ...
}
```

### JobSpec API

We extend the Job API in order to allow to apply different actions depending
on the conditions associated with the pod failure.

```golang
// BackoffPolicyAction specifies how a Pod failure is handled.
type BackoffPolicyAction string

const (

	// This is an action which might be taken on pod failure - mark the
	// pod's job as Failed.
	TerminateBackoffAction BackoffPolicyAction = "Terminate"

	// This is an action which might be taken on pod failure - the pod will be
	// restarted and the counter for .backoffLimit will not be incremented.
	IgnoreBackoffAction BackoffPolicyAction = "Ignore"

)

// BackoffPolicyRule specifies a backoff policy rule, which comprises an action
// taken on a pod failure and the condition which needs to be satisfied to take
// the action.
type BackoffPolicyRule struct {

	// Specifies the action taken on a pod failure when the condition is satisfied.
	Action BackoffPolicyAction

	// Restricts the check for exit codes to only apply to the container with
	// the specified names.
	// +optional
	ContainerName *string

	// Specifies the rule condition by the comma-separated list of exit codes or
	// ranges of exit codes. The condition is satisfied if the actual container's
	// code is in any of the ranges.
	// +optional
	ContainerExitCodeIn *string

	// Specifies the rule condition by the comma-separated list of exit codes or
	// ranges of exit codes. The condition is satisfied if the actual container's
	// code is not in any of the the ranges.
	// +optional
	ContainerExitCodeNotIn *string

	// Specifies the rule condition by a list of values for the failed pod's
	// status reason. The condition is satisfied when the actual value of the
	// status.reason field equals any of the listed values.
	// +optional
	PodStatusReasonIn *[]string

	// Specifies the rule condition by a list of values for the failed pod's
	// status reason. The condition is satisfied when the actual value of the
	// status.reason field does not equal any of the listed values.
	// +optional
	PodStatusReasonNotIn *[]string
}

// BackoffPolicy describes the handling of failed pods. The Rules field specifies
// the list of rules. Each rule specifies an action to be taken on a pod
// failure and a condition to be checked if the rule applies.
type BackoffPolicy struct {
	Rules []BackoffPolicyRule
}

// JobSpec describes how the job execution will look like.
type JobSpec struct {
  ...
	// Specifies the policy of handling failed pods. In particular, it allows to
	// specify the set of actions and conditions which need to be
	// satisfied to take the associated action.
	// If empty, the default behaviour applies - the counter of pod failed is
	// incremented and it is checked against the backoffLimit
	// +optional
	BackoffPolicy *BackoffPolicy
  ...
```

Additionally, we validate the following constraints for each instance of `BackoffPolicyRule`:
- exactly one of the following fields is used:`ContainerExitCodeIn`,
 `ContainerExitCodeNotIn`, `PodStatusReasonIn`, `PodStatusReasonNotIn`.
- the `ContainerName` is only combined with the fields `ContainerExitCodeIn` or
 `ContainerExitCodeNotIn`.

Here is an example Job configuration which uses this API:

```yaml
apiVersion: v1
kind: Job
spec:
  template:
    spec:
      containers:
      - name: main-job-container
        image: job-image
        command: ["./program"]
      - name: monitoring-job-container
        image: job-monitoring
        command: ["./monitoring"]
  backoffLimit: 3
  backoffPolicy:
    rules:
    - containerExitCodeIn: 1-12,100-127
      containerName: main-job-container
      action: Terminate
    - podStatusReasonNotIn:
      - ''
      action: Ignore
```

<!--
This is important to clarify: What does this mean?

Would the rest of the pods be terminated? Would they allow to run? The simplest approach is to terminate all the pods, just like we do when the backoffLimit is reached. And we can introduce other options (and some specific to indexed jobs) in a follow up KEP next release.
 -->

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*
The goal is to ensure that we don't accept enhancements with inadequate testing.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

[ ] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

<!--
Based on reviewers feedback describe what additional tests need to be added prior
implementing this enhancement to ensure the enhancements have also solid foundations.
-->

##### Unit tests

<!--
In principle every added code should have complete unit test coverage, so providing
the exact set of tests will not bring additional value.
However, if complete unit test coverage is not possible, explain the reason of it
together with explanation why this is acceptable.
-->

<!--
Additionally, for Alpha try to enumerate the core package you will be touching
to implement this enhancement and provide the current unit coverage for those
in the form of:
- <package>: <date> - <current test coverage>
The data can be easily read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit

This can inform certain test coverage improvements that we want to do before
extending the production code to implement this enhancement.
-->

- `<package>`: `<date>` - `<test coverage>`

##### Integration tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

- <test>: <link to test coverage>

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html

We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->

- <test>: <link to test coverage>

### Graduation Criteria

<!--
**Note:** *Not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, [feature gate] graduations, or as
something else. The KEP should keep this high-level with a focus on what
signals will be looked at to determine graduation.

Consider the following in developing the graduation criteria for this enhancement:
- [Maturity levels (`alpha`, `beta`, `stable`)][maturity-levels]
- [Feature gate][feature gate] lifecycle
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc
definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning)
or by redefining what graduation means.

In general we try to use the same stages (alpha, beta, GA), regardless of how the
functionality is accessed.

[feature gate]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

Below are some examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

#### Alpha

- Feature implemented behind a feature flag
- Initial e2e tests completed and enabled

#### Beta

- Gather feedback from developers and surveys
- Complete features A, B, C
- Additional tests are in Testgrid and linked in KEP

#### GA

- N examples of real-world usage
- N installs
- More rigorous forms of testing—e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least two releases between beta and
GA/stable, because there's no opportunity for user feedback, or even bug reports,
in back-to-back releases.

**For non-optional features moving to GA, the graduation criteria must include
[conformance tests].**

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md

#### Deprecation

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality that deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag
-->

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to maintain previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to make use of the enhancement?
-->

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the KEP to move to `implementable` status and be included in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.
-->

### Feature Enablement and Rollback

<!--
This section must be completed when targeting alpha to a release.
-->

###### How can this feature be enabled / disabled in a live cluster?

<!--
Pick one of these and delete the rest.

Documentation is available on [feature gate lifecycle] and expectations, as
well as the [existing list] of feature gates.

[feature gate lifecycle]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[existing list]: https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/
-->

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: RetriableAndNonRetriableFailures
  - Components depending on the feature gate:
    - kube-controller-manager
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control
    plane?
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node? (Do not assume `Dynamic Kubelet Config` feature is enabled).

###### Does enabling the feature change any default behavior?

Yes. The kubernetes components (kube-scheduler and kube-controller-manager) will
extend the Pod delete requests with the `reason` field. The kube-apiserver
will store the reason in the pod's `status.reason` field.

However, the part of the feature responsible for handling of the failed pods
is opt-in with `.spec.backoffPolicy`.
<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Using the feature gate is the recommended way. When the feature is disabled
the terminated pods continue to get the `status.reason` field set. However,
the Job controller manager will handle pod failures in the default way even if
`spec.backoffPolicy` is specified.

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

Feature gates are typically disabled by setting the flag to `false` and
restarting the component. No other changes should be necessary to disable the
feature.

NOTE: Also set `disable-supported` to `true` or `false` in `kep.yaml`.
-->

###### What happens if we reenable the feature if it was previously rolled back?

The Job controller starts to handle pod failures according to the specified
`spec.backoffPolicy`.

###### Are there any tests for feature enablement/disablement?

Yes, unit and integration test for the feature enabled, disabled and transitions.

<!--
The e2e framework does not currently support enabling or disabling feature
gates. However, unit tests in each component dealing with managing data, created
with and without the feature, are necessary. At the very least, think about
conversion tests if API types are being modified.

Additionally, for features that are introducing a new API field, unit tests that
are exercising the `switch` of feature gate itself (what happens if I disable a
feature gate after having objects written with the new field) are also critical.
You can take a look at one potential example of such test in:
https://github.com/kubernetes/kubernetes/pull/97058/files#diff-7826f7adbc1996a05ab52e3f5f02429e94b68ce6bce0dc534d1be636154fded3R246-R282
-->

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some API servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

- [ ] Events
  - Event Reason:
- [ ] API .status
  - Condition name:
  - Other field:
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!--
This is your opportunity to define what "normal" quality of service looks like
for a feature.

It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question.
-->

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

No.

<!--
Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
Focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
-->

###### Will enabling / using this feature result in introducing new API types?

No.
<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.
<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

Yes. The `DeleteOptions` structure is extended by the `reason` field.
<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No.
<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

Additional CPU and memory increase in kubernetes components with regards to the
handling of the `reason` field is negligible. The additional CPU and memory
increase in kube-controller-manager related to handling of failed pods is also
negligible and only limited to these jobs which specify `spec.backoffPolicy`.

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

###### What are other known failure modes?

<!--
For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.
-->

###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
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

### Alternative 1

This alternative aims to better structure the conditions. In particular, the
`containerName` field is not only applicable to conditions based on container
exit codes.

```golang
type BackoffPolicyOnExitCodeCondition struct {
	ContainerName *string
	ValueIn       *string
	ValueNotIn    *string
}

type BackoffPolicyOnPodStatusCondition struct {
	ReasonIn      *[]string
	ReasonNotIn   *[]string
}

type BackoffPolicyCondition struct {
	OnContainerExitCode *BackoffPolicyOnExitCodeCondition
	OnPodStatus         *BackoffPolicyOnPodStatusCondition
}

type BackoffPolicyRule struct {
	Action    BackoffPolicyAction
	Condition BackoffPolicyCondition
}

type BackoffPolicy struct {
	Rules []BackoffPolicyRule
}
```

Example:
```yaml
apiVersion: v1
kind: Job
spec:
  template:
    spec:
      containers:
      - name: main-job-container
        image: job-image
        command: ["./program"]
      - name: monitoring-job-container
        image: job-monitoring
        command: ["./monitoring"]
  backoffLimit: 3
  backoffPolicy:
    rules:
    - action: Terminate
      onContainerExitCode:
        valueIn: 1-12,100-127
        containerName: main-job-container
    - action: Ignore
      onPodStatus:
        reasonIn:
        - Preemption
        - NodePressureEviction
```

### Alternative 2

Aims to introduce more re-usable and flexible API for defining the conditions
based on container exit codes or pod status. It would be based on the idea
used to specify the `matchExpressions` field (see: [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#:~:text=matchExpressions%20is%20a%20list%20of,satisfied%20in%20order%20to%20match.)).

Example:
```yaml
apiVersion: v1
kind: Job
spec:
  template:
    spec:
      containers:
      - name: main-job-container
        image: job-image
        command: ["./program"]
      - name: monitoring-job-container
        image: job-monitoring
        command: ["./monitoring"]
  backoffLimit: 3
  backoffPolicy:
    rules:
    - action: Terminate
      onContainerExitCode:
        valueExpressions:
        - {operator: NotIn, values: [40-50]}
        containerName: main-job-container
    - action: Ignore
      onPodStatus:
        reasonExpressions:
        - {operator: In, values: [Preemption, NodePressureEviction]}
```
<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->
