# KEP-2307: Job tracking without lingering Pods

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [New API calls](#new-api-calls)
    - [Bigger Job status](#bigger-job-status)
    - [Unprotected Job status endpoint](#unprotected-job-status-endpoint)
- [Design Details](#design-details)
  - [API changes](#api-changes)
  - [Algorithm](#algorithm)
    - [Simplified algorithm for Indexed Jobs](#simplified-algorithm-for-indexed-jobs)
  - [Deleted Pods](#deleted-pods)
  - [Deleted Jobs](#deleted-jobs)
  - [Pod adoption](#pod-adoption)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Alpha -&gt; Beta Graduation](#alpha---beta-graduation)
    - [Beta -&gt; GA Graduation](#beta---ga-graduation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
    - [How can this feature be enabled / disabled in a live cluster?](#how-can-this-feature-be-enabled--disabled-in-a-live-cluster)
    - [Does enabling the feature change any default behavior?](#does-enabling-the-feature-change-any-default-behavior)
    - [Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?](#can-the-feature-be-disabled-once-it-has-been-enabled-ie-can-we-roll-back-the-enablement)
    - [What happens if we reenable the feature if it was previously rolled back?**](#what-happens-if-we-reenable-the-feature-if-it-was-previously-rolled-back)
    - [Are there any tests for feature enablement/disablement?**](#are-there-any-tests-for-feature-enablementdisablement)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
    - [How can a rollout fail? Can it impact already running workloads?](#how-can-a-rollout-fail-can-it-impact-already-running-workloads)
    - [What specific metrics should inform a rollback?](#what-specific-metrics-should-inform-a-rollback)
    - [Were upgrade and rollback tested? Was the upgrade-&gt;downgrade-&gt;upgrade path tested?](#were-upgrade-and-rollback-tested-was-the-upgrade-downgrade-upgrade-path-tested)
    - [Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?](#is-the-rollout-accompanied-by-any-deprecations-andor-removals-of-features-apis-fields-of-api-types-flags-etc)
  - [Monitoring Requirements](#monitoring-requirements)
    - [How can an operator determine if the feature is in use by workloads?](#how-can-an-operator-determine-if-the-feature-is-in-use-by-workloads)
    - [What are the reasonable SLOs (Service Level Objectives) for the enhancement?](#what-are-the-reasonable-slos-service-level-objectives-for-the-enhancement)
    - [What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?](#what-are-the-slis-service-level-indicators-an-operator-can-use-to-determine-the-health-of-the-service)
    - [Are there any missing metrics that would be useful to have to improve observability of this feature?](#are-there-any-missing-metrics-that-would-be-useful-to-have-to-improve-observability-of-this-feature)
  - [Dependencies](#dependencies)
    - [Does this feature depend on any specific services running in the cluster?](#does-this-feature-depend-on-any-specific-services-running-in-the-cluster)
  - [Scalability](#scalability)
    - [Will enabling / using this feature result in any new API calls?](#will-enabling--using-this-feature-result-in-any-new-api-calls)
    - [Will enabling / using this feature result in introducing new API types?](#will-enabling--using-this-feature-result-in-introducing-new-api-types)
    - [Will enabling / using this feature result in any new calls to the cloud provider?](#will-enabling--using-this-feature-result-in-any-new-calls-to-the-cloud-provider)
    - [Will enabling / using this feature result in increasing size or count of the existing API objects?](#will-enabling--using-this-feature-result-in-increasing-size-or-count-of-the-existing-api-objects)
    - [Will enabling / using this feature result in increasing time taken by any operations covered by <a href="https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos">existing SLIs/SLOs</a>?](#will-enabling--using-this-feature-result-in-increasing-time-taken-by-any-operations-covered-by-existing-slisslos)
    - [Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?](#will-enabling--using-this-feature-result-in-non-negligible-increase-of-resource-usage-cpu-ram-disk-io--in-any-components)
  - [Troubleshooting](#troubleshooting)
    - [How does this feature react if the API server and/or etcd is unavailable?](#how-does-this-feature-react-if-the-api-server-andor-etcd-is-unavailable)
    - [What are other known failure modes?](#what-are-other-known-failure-modes)
    - [What steps should be taken if SLOs are not being met to determine the problem?](#what-steps-should-be-taken-if-slos-are-not-being-met-to-determine-the-problem)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [x] (R) KEP approvers have approved the KEP status as `implementable`
- [x] (R) Design details are appropriately documented
- [x] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [x] (R) Graduation criteria is in place
- [x] (R) Production readiness review completed
- [x] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

The current Job controller currently relies on completed Pods to not be removed
in order to track the Job completion status. This proposal presents an
alternative implementation for Job tracking that does not have this dependency.

## Motivation

The current approach of relying on the Pods existence is problematic for Jobs
that require a big number of completions or for clusters with too many Jobs
running at the same time. The finished Pods cannot be removed until the entire
Job completes, even if the Pods failed.

Furthermore, once the number of finished Pods reaches a threshold, the Pod
garbage collection controller starts removing Pods. Today, the Job controller
relies on the garbage collector having a big threshold.

### Goals

- Perform Job completion and failures tracking without relying on lingering
  Pods.

### Non-Goals

- Remove Pods once they have been accounted for.

## Proposal

The Job controller creates Pods with a finalizer to prevent finished Pods from
being removed by the garbage collector. The Job controller removes the finalizer
from the finished Pods once it has accounted for them. In subsequent Job syncs,
the controller ignores finished Pods that don't have the finalizer.

### Notes/Constraints/Caveats (Optional)

Due to the lack of support of atomic changes across kubernetes objects, an
intermediate state is necessary. Before removing the Pod finalizers,
the controller adds finished Pods to a list in the Job status. After removing
the Pod finalizers, the controller clears the list and updates the counters.

### Risks and Mitigations

#### New API calls

The new algorithm introduces new API calls:
- one per Pod lifecycle to remove finalizers
- one for each Job sync, to track the intermediate state.

Note that there no new calls in the Pod creation path.

On the other hand, to update the Job status once a Pod finishes, we need a total
2 new API calls compared to the legacy algorithm. If more than one Pod finish at
a given time, we add `n + 1` API calls, where `n` is the number finished Pods.

Consider a Job with multiple Pods. With a 50 QPS limit in the job controller
client, the controller should be able to process between 2000 to 3000 Pods.

The increase in API calls is justified for the following reasons:
- The legacy tracking cannot handle a big number of terminated Pods, across
  any number of Jobs, at a given point.
- In the entirety of its lifecycle, a Pod requires at least 8 API calls,
  including status updates and events.
  
However, in order to prevent Jobs with big number of Pods from starving Jobs
with fewer Pods, the Job controller might skip status updates until enough
Pods have accumulated or enough time has passed. See [Algorithm](#algorithm)
below for more details.
   
#### Bigger Job status

Job status can temporarily grow if too many Pods finish at the same time or
if the Job controller is down for some time. In this case, we do partial
status updates. See [Algorithm](#algorithm) below.

#### Unprotected Job status endpoint

Changes in the status not produced by the Job controller in
kube-controller-manager could affect the Job tracking. Cluster administrators
should make sure to protect the Job status endpoint via RBAC.

## Design Details

### API changes

The Job status gets a new struct to hold the uncounted Pods before they are
added to the counters.

```golang
type JobStatus struct {
    Succeeded int32
    Failed    int32
    ...

    // UncountedTerminatedPods holds UIDs of Pods that have finished but
    // haven't been accounted in the counters.
    // Old jobs might not be tracked using this field, in which case this
    // field remains null.
    // +optional
    UncountedTerminatedPods *UncountedTerminatedPods
}

// UncountedTerminatedPods holds UIDs of Pods that have finished but haven't
// been accounted in Job status counters.
type UncountedTerminatedPods struct {
    // Succeeded holds UIDs of succeeded Pods.
    Succeeded []types.UID
    // Failed holds UIDs of failed Pods.
    Failed    []types.UID
}
```

Note: the final name of the field `uncountedTerminatedPods` will be decided
during API review.

### Algorithm

The following algorithm updates the status counters without relying on finished
Pods to be present indefinitely. The algorithm assumes that the Job controller
could be stopped at any point and executed again from the first step without
losing information. Generally, all the steps happen in a single Job sync
cycle.

0. kube-apiserver adds the `batch.kubernetes.io/job-completion` annotation
   to newly created Jobs. This annotation allows the distinction of new Jobs
   from Jobs that are already tracked with the legacy algorithm.
1. The Job controller calculates the number of succeeded Pods as the sum of:
   - `.status.succeeded`,
   - the size of `job.status.uncountedTerminatedPods.succeeded` and
   - the number of finished Pods that are not in `job.status.uncountedTerminatedPods.succeeded`
     and have a finalizer.
     
   The Job controller calculates the number of failed Pods similarly, and the
   number of active Pods as Pods that don't have a Failed or Succeeded condition
   and have a finalizer.

   This number informs the creation of missing Pods to reach `.spec.completions`.
   The controller creates Pods for a Job with the finalizer
   `batch.kubernetes.io/job-completion`.
2. The Job controller adds Pod UIDs to the `.status.uncountedTerminatedPods.succeeded`
   and `.status.uncountedTerminatedPods.failed` lists if the Pod:
    - has the `batch.kubernetes.io/job-completion` finalizer, and
    - the Pod is on Succeeded or Failed phase, respectively.
    The controller sends a status update.
3. The Job controller removes the `batch.kubernetes.io/job-completion` finalizer
   from all Pods on Succeeded or Failed phase that were added to the lists in
   `.status.uncountedTerminatedPods` in the previous step.
4. The Job controller counts the Pods in the `.status.uncountedTerminatedPods` lists
   that:
   - have no finalizer, or
   - were removed from the system.
   The counts increment the `.status.failed` and `.status.succeeded` and clears
   counted Pods from `.status.uncountedTerminatedPods` lists. The controller
   sends a status update.

Steps 2 to 4 might deal with a potentially big number of Pods. Thus, status
updates can potentially stress the kube-apiserver. For this reason, the Job
controller repeats steps 2 to 4, capping the number of Pods each time, until
the controller processes all the Pods. The number of Pods is caped by:
- time: in each Job sync, the job controller removes all the Pods' finalizers
  it can in a unit of time in the order of tens of seconds. If there are
  pending Pods to count, the controller puts back the Job into the work queue.
  This allows to throttle the number of Job status updates and avoid starving
  smaller jobs.
- count: Preventing big writes to etcd. We limit the number of UIDs to the order
  of hundreds, keeping the size of the slice under 20kb.

If any Pod finalizer removal fails in step 3, the controller manager still
executes step 4 with the Pods that succeeded.

Steps 2 to 4 might be skipped in the scenario where a status update happened
too recently and the number of uncounted Pods is a small percentage of
parallelism.

Note that the `.status.uncountedTerminatedPods` struct allows to uniquely
identify finished Pods to avoid over counting.

#### Simplified algorithm for Indexed Jobs

Pods in Indexed Jobs have a unique identifier: the completion index. Even if
more than one Pod gets created for the same index, only one of them counts
towards completions. The completed indexes are available in
`.status.completedIndexes` in a compressed format.

When tracking Indexed Jobs, the Job controller can use
`.status.completedIndexes` in place of
`.status.uncountedTerminatedPods.succeeded` in step 2 and completely skip step 4
if there are no failed terminated pods in the same sync cycle. This saves one
API call for a Job status update.

### Deleted Pods
   
In the case where a user or another controller removes a Pod, which sets a
deletion timestamp, the Job controller treats it the same as any other Pod.
Since deleted Pods with finalizers get inevitably marked as Failed, the
Job controller already counts them as such and removes their finalizers.
This is different from the legacy tracking, where the Job controller does not
account for deleted Pods. This is a limitation that this KEP also wants to
solve.

One edge case is when there is a Node failure. If the Node is down long enough,
its Pods become orphan, and the garbage collector deletes them. Some of these
deleted Pods could not have finished, but the algorithm described above treats
them as failed.

On the other hand, if the Job controller deletes the Pod (when the user
decreases parallelism or suspends the Job, for example), the controller removes
the finalizer before deleting it. Thus, these deletions don't count towards the
failures.

### Deleted Jobs

When a user or another controller deletes a Job, the cascading makes sure that
each Pods gets a deletion timestamp. The job controller captures this Pod
update event, adding the orphan Pod (Pod for which the Job controller doesn't
exist) to a separate work queue. A single worker scans this work queue to
remove the finalizer from the Pod.
   
### Pod adoption

If a Job with `.status.uncountedTerminatedPods != nil` can adopt a Pod
(according to the existing adoption criteria), this Pod might not have a
finalizer.

The job controller adds the finalizer in the same patch request that modifies
the owner reference.

### Test Plan

- Unit tests:
  - Job sync with feature gate enabled.
  - Removal of finalizers when feature gate is disabled.
  - Tracking of terminating Pods.
- Integration tests:
  - Job tracking with feature enabled.
  - Tracking of terminating Pods.
  - Transition from feature enabled to disabled and enabled again.
  - Clean up finalizers of Orphan Pods.
  - Tracking Jobs with big number of Pods, making sure the status is eventually
    consistent.
- E2E test:
  - Job tracking with feature enabled.

### Graduation Criteria

#### Alpha

- Implementation:
  - Job tracking without lingering Pods
  - Removal of finalizer when feature gate is disabled.
  - Support for [Indexed Jobs](https://git.k8s.io/enhancements/keps/sig-apps/2214-indexed-job)
- Tests: unit, integration.

#### Alpha -> Beta Graduation

- Pod processing throughput per minute (mix of creating and counting finished Pods),
  assuming an average Job .spec.parallelism=10.
  - Up to 2500 Pods (~3000 queries) for a 50 QPS client limit for the job controller.
  - Up to 5000 (~6000 queries) Pods for a 100 QPS client limit for the job controller.
- Ensure that tracking Jobs with big number of Pods doesn't cause starvation of
  smaller jobs.
- Metrics for latency, counting updates and errors.
- Job E2E tests are in Testgrid with the feature enabled and linked in KEP

#### Beta -> GA Graduation

- Remove legacy tracking and the use of `batch.kubernetes.io/job-completion` as
  an annotation. This is possible assuming:
  - After the feature was enabled for some time, the Job controller is already
    tracking most Jobs using finalizers.
  - For the remaining Jobs, the Job controller already accounted most of the
    finished Pods in the status. The controller adds a tracking finalizer to
    any running Pod that doesn't have it. The controller might leave some
    finished Pods unaccounted, if they finish before the controller has a chance
    to add a finalizer. This is acceptable as it's no worse that the current
    behavior were the controller doesn't account for Pods removed by garbage
    collection or other means.
- Job E2E tests graduate to conformance.
- Job tracking scales to 10^5 completions per Job processed within an order of
  minutes.

### Upgrade / Downgrade Strategy

When the feature `JobTrackingWithFinalizers` is enabled for the first
time, the cluster can have Jobs whose Pods don't have the
`batch.kubernetes.io/job-completion` finalizer. It would be hard to add the
finalizer to all Pods while preventing race conditions. That is, at the time
of migration to the new tracking, a Pod could not have the finalizer for two
reasons: it wasn't migrated yet, or it was already counted.

The job controller uses the existence of the Job annotation
`batch.kubernetes.io/job-completion` to determine if it should use tracking with
finalizers. If the annotation is not present, and the Job is not yet completed,
the job controllers tracks Pods using the legacy tracking (with lingering Pods).

The kube-apiserver sets the `batch.kubernetes.io/job-completion` annotation to
newly created Jobs when the feature gate `JobTrackingWithFinalizers` is enabled.
This annotation cannot be added in a Job update.

When the feature is disabled after being enabled for some time, the next time
the Job controller syncs a Job:
1. It removes finalizers from the Pods owned by it and the annotation from the
   Job.
2. Sets `.status.uncountedTerminatedPods` to nil.

After this point, the Job will no longer be tracked using finalizers, even if
the feature gate is re-enabled.

### Version Skew Strategy

No implications to node runtime.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

#### How can this feature be enabled / disabled in a live cluster?
  - [x] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name: JobTrackingWithFinalizers
    - Components depending on the feature gate:
      - kube-apiserver
      - kube-controller-manager
  - [ ] Other
    - Describe the mechanism:
    - Will enabling / disabling the feature require downtime of the control
      plane?
    - Will enabling / disabling the feature require downtime or reprovisioning
      of a node? (Do not assume `Dynamic Kubelet Config` feature is enabled).

#### Does enabling the feature change any default behavior?

  Yes.
  
  - Removing terminated Pods doesn't affect Job status.
  - Pods removed by the user or other controllers count towards failures or
    completions.

#### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?
  
  Yes.
  The job controller removes finalizers in this case.
  Since some succeeded Pods might have been removed, the job controller will
  create new Pods to fulfill completions. But this is no different from
  existing behavior with the legacy tracking.

#### What happens if we reenable the feature if it was previously rolled back?**

  Existing Jobs are tracked with legacy Pods.

#### Are there any tests for feature enablement/disablement?**

  Yes, we have [integration tests](https://github.com/kubernetes/kubernetes/blob/7a0638da76cb9843def65708b661d2c6aa58ed5a/test/integration/job/job_test.go)
  for feature enabled, disabled and transitions.

### Rollout, Upgrade and Rollback Planning

#### How can a rollout fail? Can it impact already running workloads?
  
  The change doesn't affect running Pods. If the component restarts
  mid-rollout into an older version, the Job controller switches to tracking
  Jobs without using finalizers.

#### What specific metrics should inform a rollback?

  - An increase in `job_sync_duration_seconds`. Users should expect a higher
    duration than previous versions of the job controller due to the new API
    calls.
  - Stale `job_sync_total` or `job_finished_total`.

#### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?
  
  Integration tests cover feature gate disablement and re-enablement.

  A manual upgrade->downgrade->upgrade flow will be executed prior to Beta
  graduation to ensure that a running Job falls back to tracking without
  finalizers. The KEP will be updated with the findings of the test.

#### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?
  
  No.

### Monitoring Requirements

#### How can an operator determine if the feature is in use by workloads?
  
  - The metric `job_pod_finished` (with a label result=failed/completed)
    increments when the job controller removes a Pod out of
    `.status.uncountedTerminatedPods` to increase the failed/completed counters.
  - Administrators can check for the existence of Job objects with the annotation
    `batch.kubernetes.io/job-completion` or Pods with the finalizer
    `batch.kubernetes.io/job-completion`.

###### How can someone using this feature know that it is working for their instance?

- [ ] Events
  - Event Reason:
- [x] API .status
  - Condition name:
  - Other field: `Job.status.uncountedTerminatedPods` is not null.
- [x] Other (treat as last resort)
  - Details: The `Job` has the annotation `batch.kubernetes.io/job-completion`

#### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

- 99% percentile over day for Job syncs is <= 15s for a client-side 50 QPS
  limit.
    
#### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [x] Metrics
    - Metric name: `job_sync_duration_seconds`
    - [Optional] Aggregation method:
    - Components exposing the metric: `kube-controller-manager`

#### Are there any missing metrics that would be useful to have to improve observability of this feature?
  
  - A label in `job_sync_total` for the type of Job tracking. This label would
    have to be removed when we graduate the feature to GA, adding operational
    burden.

### Dependencies

#### Does this feature depend on any specific services running in the cluster?
  
  No.

### Scalability

#### Will enabling / using this feature result in any new API calls?

  - PATCH Pods, to remove finalizers.
    - estimated throughput: one per Pod created by the Job controller, when Pod
      finishes or is removed.
    - originating component: kube-controller-manager
  - PUT Job status, to keep track of uncounted Pods.
    - estimated throughput: at least one per Job sync. The job controller
      throttles additional calls at 1 per a few seconds (precise throughput TBD
      from experiments).
    - originating component: kube-controller-manager.

#### Will enabling / using this feature result in introducing new API types?

  No.

#### Will enabling / using this feature result in any new calls to the cloud provider?

  No.

#### Will enabling / using this feature result in increasing size or count of the existing API objects?

  - Pod
    - Estimated increase: new finalizer of 33 bytes.
  - Job
    - Estimated increase: new finalizer of 33 bytes.
  - Job status
    - Estimated increase: new array temporarily containing terminated Pod UIDs.
      The job controller caps the size of the array to less than 20kb.

#### Will enabling / using this feature result in increasing time taken by any operations covered by [existing SLIs/SLOs]?

  Users should expect an increase in the `job_sync_duration_seconds` metric.
  There is no existing SLO for Jobs.

#### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

  Additional memory to hold terminated Pods and the status of removing their
  finalizers.

### Troubleshooting

#### How does this feature react if the API server and/or etcd is unavailable?

  It wouldn't make progress, but the controllers re-queues the Job syncs.
  This is no different from existing behavior.

#### What are other known failure modes?
  
  TBD from user feedback. No know failures modes so far.
  
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

#### What steps should be taken if SLOs are not being met to determine the problem?

1. Check reachability between kube-controller-manager and apiserver.
1. If the `job_sync_duration_seconds` above the suggested SLO, check for the number
   of requests in apiserver coming from the kube-system/job-controller service
   account. Consider increasing the number of inflight requests for
   apiserver or tuning [API priority and fairness](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/)
   to give more priority for the job-controller requests.
1. If the steps above are insufficient or if the `job_sync_total` metric is stale,
   even though there are Jobs progressing in the cluster, disable the `JobTrackingWithFinalizers`
   feature gate from apiserver and kube-controller-manager and [report an issue](https://github.com/kubernetes/kubernetes/issues).

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

- 2021-02-08: First proposal.
- 2021-04-20: Target 1.22 for Alpha.
- 2021-07-09: Alpha implementation merged
- 2021-08-18: PRR completed and graduation to beta proposed.

## Drawbacks

- Extra API calls and temporarily bigger Job status. However, without them
  it's impossible to ever scale the Job controller to deal with greater amount
  of Jobs or Jobs with greater amount of Pods.

## Alternatives

- Keep a list of created Pod UIDs, clearing them when they have been accounted.
  This has the benefit of requiring less Job status updates. On the other hand,
  the size of the updates is unbounded.
