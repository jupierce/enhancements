---
title: cluster-change-management-and-maintenance-schedules
authors:
  - @jupierce
reviewers: 
  - @jharrington22
  - @JoelSpeed
  - @LalatenduMohanty
approvers: 
  - @sdodson
api-approvers: 
  - @mrunalp, MCP API Changes.
  - @sjenning, HyperShift API Changes.
  - @jharrington22, SD Integration.
creation-date: 2024-02-29
last-updated: 2024-02-29

tracking-link:
  - TBD
  
---

# Cluster Change Management and Maintenance Schedules

## Summary
Implement high level APIs for change management which allow
standalone and Hosted Control Plane (HCP) clusters a measure of configurable control
over when control-plane or worker-node configuration rollouts are initiated. 
As a primary mode of configuring change management, implement an option
called Maintenance Schedules which define reoccurring windows of time (and specifically
excluded times) in which potentially disruptive changes in configuration can be initiated. 

Material changes not permitted by change management configuration are left in a 
pending state until such time as they are permitted by the configuration. 

Change management enforcement _does not_ guarantee that all initiated
material changes are completed by the close of a permitted change window (e.g. a worker-node
may still be draining or rebooting) at the close of a maintenance schedule, 
but it does prevent _additional_ material changes from being initiated. 

A "material change" may vary by cluster profile and subsystem. For example, a 
control-plane update (all components and control-plane nodes updated) is implemented as
a single material change (e.g. the close of a scheduled maintenance window
will not suspend its progress). In contrast, the rollout of worker-node updates is
more granular (you can consider it as many individual material changes) and  
the end of a permitted change window will prevent additional worker-node updates 
from being initiated.

Changes vital to the continued operation of the cluster (e.g. certificate rotation) 
are not considered material changes. Ignoring operational practicalities (e.g.
the need to fix critical bugs or update a cluster to supported software versions), 
it should be possible to safely leave changes pending indefinitely. That said,
Service Delivery and/or higher level management systems may choose to prevent
such problematic change management settings from being applied by using 
validating webhooks.

## Motivation
This enhancement is designed to improve user experience during the OpenShift 
upgrade process and other key operational moments when configuration updates
may result in material changes in cluster behavior and potential disruption 
for non-HA workloads.

The enhancement offers a direct operational tool to users while indirectly
supporting a longer term separation of control-plane and worker-node updates
for **Standalone** cluster profiles into distinct concepts and phases of managing 
an OpenShift cluster (HCP clusters already provide this distinction). The motivations
for both aspects will be covered, but a special focus will be made on the motivation
for separating Standalone control-plane and worker-node updates as, while not fully realized 
by this enhancement alone, ultimately provides additional business value helping to 
justify an investment in the new operational tool.

### Supporting the Eventual Separation of Control-Plane and Worker-Node Updates
One of the key value propositions of this proposal pre-supposes a successful
decomposition of the existing, fully self-managed, Standalone update process into two  
distinct phases as understood and controlled by the end-user: 
(1) control-plane update and (2) worker-node updates.

To some extent, Maintenance Schedules (a key supported option for change management) 
are a solution to a problem that will be created by this separation: there is a perception that it would also
double the operational burden for users updating a cluster (i.e. they have
two phases to initiate and monitor instead of just one). In short, implementing the
Maintenance Schedules concept allows users to succinctly express if and how
they wish to differentiate these phases.

Users well served by the fully self-managed update experience can disable 
change management (i.e. not set an enforced maintenance schedule), specifying
that control-plane and worker node updates can take place at
any time. Users who need more control may choose to update their control-plane
regularly (e.g. to patch CVEs) with a permissive change management configuration
for the control-plane while using a tight maintenance schedule for worker-nodes
to only update during specific, low utilization, periods.

Since separating the node update phases is such an important driver for
Maintenance Schedules, their motivations are heavily intertwined. The remainder of this 
section, therefore, delves into the motivation for this separation. 

#### The Case for Control-Plane and Worker-Node Separation
From an overall platform perspective, we believe it is important to drive a distinction
between updates of the control-plane and worker-nodes. Currently, an update is initiated
and OpenShift's ostensibly fully self-managed update mechanics take over (CVO laying
out new manifests, cluster operators rolling out new operands, etc.) culminating with 
worker-nodes being drained a rebooted by the machine-config-operator (MCO) to align
them with the version of OpenShift running on the control-plane.

This approach has proven extraordinarily successful in providing a fast and reliable 
control-plane update, but, in rare cases, the highly opinionated update process leads
to less than ideal outcomes. 

##### Node Update Separation to Address Problems in User Perception
Our success in making OpenShift control-plane updates reliable, exhaustive focus on quality aside,
is also made possible by the platform's exclusive ownership of the workloads that run on the control-plane 
nodes. Worker-nodes, on the other hand, run an endless variety of non-platform, user defined workloads - many of
which are not necessarily perfectly crafted. For example, workloads with pod disruption budgets (PDBs) that
prevent node drains and workloads which are not fundamentally HA (i.e. where draining part of the workload creates
disruption in the service it provides). 

Ultimately, we cannot solve the issue of problematic user workload configurations because
they are intentionally representable with Kubernetes APIs (e.g. it may be the user's actual intention to prevent a pod
from being drained or it may be too expensive to make a workload fully HA). When confronted with 
problematic workloads, the current, fully self-managed, OpenShift update process can appear to the end-user
to be unreliable or slow. This is because the self-managed update process takes on the end-to-end responsibility
of updating the control-plane and worker-nodes. Given the automated and somewhat opaque nature of this
update, it is reasonable for users to expect that the process is hands-off and will complete in a timely 
manner regardless of their workloads.  

When this expectation is violated because of problematic user workloads, the update process is 
often called into question. For example, if an update appears stalled after 12 hours, a
user is likely to have a poor perception of the platform and open a support ticket before
successfully diagnosing an underlying undrainable workload.

By separating control-plane and worker-node updates into two distinct phases for an operator to consider,
we can more clearly communicate (1) the reliability and speeed of OpenShift control-plane updates and
(2) the shared responsibility, along with the end user, of successfully updating worker-nodes. 

As an analogy, when you are late to work because of delays in a subway system, you blame the subway system.
They own the infrastructure and schedules and have every reason to provide reliable and predictable transport.
If, instead, you are late to work because you step into a fully automated car that gets stuck in traffic, you blame the
traffic. The fully self-managed update process suggests to the end user that it is a subway -- subtly insulating
them from the fact that they might well hit traffic (problematic user workloads). Separating the update journey into
two parts - a subway portion (the control-plane) and a self-driving car portion (worker-nodes), we can quickly build the
user's intuition about their responsibilities in the latter part of the journey. For example, leaving earlier to
avoid traffic or staying at a hotel after the subway to optimize their departure for the car ride.

##### Node Update Separation to Improve Risk Mitigation Strategies
With any cluster update, there is risk --  software is changing and even subtle differences in behavior can cause
issues given an unlucky combination of factors. Teams responsible for cluster operations are familiar with these
risks and owe it to their stakeholders to minimize them where possible.

The current, fully self-managed, update process makes one obvious risk mitigation strategy
a relatively advanced strategy to employ: only updating the control-plane and leaving worker-nodes as-is.
It is possible by pausing machine config pools, but this is certainly not an intuitive step for users. Farther back 
in OpenShift 4's history, the strategy was not even safe to perform since it could lead to worker-node 
certificates to expiring. 

By separating the control-plane and worker-node updates into two separate steps, we provide a clear
and intuitive method of deferring worker-node updates: not initiating them. Leaving this to the user's
discretion, within safe skew-bounds, gives them the flexibility to make the right choices for their
unique circumstances.

#### Enhancing Operational Control
The preceding section delved deeply into a motivation for Change Management / Maintenance Schedules based on our desire to 
separate control-plane and worker-node updates without increasing operational burden on end-users. However,
Change Management, by providing control over exactly when updates & material changes to nodes in
the cluster can be initiated, provide value irrespective of this strategic direction. The benefit of
controlling exactly when changes are applied to critical systems is universally appreciated in enterprise 
software. 

Since these are such well established principles, I will summarize the motivation as helping
OpenShift meet industry standard expectations with respect to limiting potentially disruptive change 
outside well planned time windows. 

It could be argued that rigorous and time sensitive management of OpenShift cluster API resources could prevent
unplanned material changes, but Change Management / Maintenance Schedules introduce higher level, platform native, and more 
intuitive guard rails. For example, consider the common pattern of a gitops configured OpenShift cluster.
If a user wants to introduce a change to a MachineConfig, it is simple to merge a change to the
resource without appreciating the fact that it will trigger a rolling reboot of nodes in the cluster.

Trying to merge this change at a particular time of day and/or trying to pause and unpause a 
MachineConfigPool to limit the impact of that merge to a particular time window requires
significant forethought by the user. Even with that forethought, if an enterprise wants 
changes to only be applied during weekends, additional custom mechanics would need
to be employed to ensure the change merged during the weekend without needing someone present.

Contrast this complexity with the user setting a Change Management / Maintenance Schedule on the cluster. The user
is then free to merge configuration changes and gitops can apply those changes to OpenShift
resources, but material change to the cluster will not be initiated until a time permitted
by the Maintenance Schedule. Users do not require special insight into the implications of
configuring platform resources as the top-level Maintenance Schedule control will help ensure
that potentially disruptive changes are limited to well known time windows.

#### Reducing Service Delivery Operational Tooling
Service Delivery, as part of our OpenShift Dedicated, ROSA and other offerings is keenly aware of
the issues motivating the Change Management / Maintenance Schedule concept. This is evidenced by their design
and implementation of tooling to fill the gaps in the platform the preceding sections
suggest exist.

Specifically, Service Delivery has developed UXs outside the platform which allow customers 
to define a preferred maintenance window. For example, when requesting an update, the user 
can specify the desired start time. This is honored by Service Delivery tooling (unless
there are reasons to supersede the customer's preference).

By acknowledging the need for scheduled maintenance in the platform, we reduce the need for Service
Delivery to develop and maintain custom tooling to manage the platform while 
simultaneously reducing simplifying management for all customer facing similar challenges.

### User Stories
For readability, "cluster lifecycle administrator" is used repeatedly in the user stories. This 
term can apply to different roles depending on the cluster environment and profile. In general,
it is the person or team making most material changes to the cluster - including planning and
choosing when to enact phases of the OpenShift platform update.

For HCP, the role is called the [Cluster Service Consumer](https://hypershift-docs.netlify.app/reference/concepts-and-personas/#personas). For
Standalone clusters, this role would normally be filled by one or more `system:admin` users. There
may be several layers of abstraction between the cluster lifecycle administrator and changes being
actuated on the cluster (e.g. gitops, OCM, Hive, etc.), but the role will still be concerned with limiting
risks and disruption when rolling out changes to their environments.

> "As a cluster lifecycle administrator, I want to ensure any material changes to my cluster 
> (control-plane or worker-nodes) are only initiated during well known windows of low service 
> utilization to reduce the impact of any service disruption."

> "As a cluster lifecycle administrator, I want to ensure any material changes to my 
> control-plane are only initiated during well known windows of low service utilization to 
> reduce the impact of any service disruption."

> "As a cluster lifecycle administrator, I want to ensure that no material changes to my 
> cluster occurs during a known date range even if it falls within our
> normal maintenance schedule due to an anticipated atypical usage (e.g. Black Friday)."

> "As a cluster lifecycle administrator, I want to suspend additional material changes from 
> taking place when it is no longer practical to monitor for service disruptions. For example,
> if a worker-node update is proving to be problematic during a valid maintenance window, I would
> like to be able to pause that change manually so that the team will not have to work on the weekend."

> "As a cluster lifecycle administrator, I need to stop all material changes on my cluster
> quickly and indefinitely until I can understand a potential issue. I not want to consider dates or
> timezones in this delay as they are not known and irrelevant to my immediate concern."

> "As a cluster lifecycle administrator, I want to ensure any material changes to my 
> control-plane are only initiated during well known windows of low service utilization to 
> reduce the impact of any service disruption. Furthermore, I want to ensure that material
> changes to my worker-nodes occur on a less frequent cadence because I know my workloads
> are not HA."

> "As an SRE, tasked with performing non-emergency corrective action, I want 
> to be able to apply a desired configuration (e.g. PID limit change) and have that change roll out 
> in a minimally disruptive way subject to the customer's configured maintenance schedule."

> "As an SRE, tasked with performing emergency corrective action, I want to be able to 
> quickly disable a configured maintenance schedule, apply necessary changes, have them roll out immediately, 
> and restore the maintenance schedule to its previous configuration."

> "As a leader within the Service Delivery organization, tasked with performing emergency corrective action
> across our fleet, I want to be able to bypass and then restore customer maintenance schedules
> with minimal technical overhead."

> "As a cluster lifecycle administrator who is well served by a fully managed update without maintenance windows, 
> I want to be minimally inconvenienced by the introduction of maintenance schedules."

> "As a cluster lifecycle administrator who is not well served by a fully managed update and needs exacting 
> control over when material changes occur on my cluster where opportunities do NOT arise at reoccurring intervals,
> I want to employ a change management strategy that defers material changes until I perform a manual action."

> "As a cluster lifecycle administrator, I want to easily determine the next time at which maintenance operations
> will be permitted to be initiated, based on the configured maintenance schedule, by looking at the 
> status of relevant API resources or metrics."

> "As a cluster lifecycle administrator, I want to easily determine whether there are material changes pending for
> my cluster, awaiting a permitted window based on the configured maintenance schedule, by looking at the 
> status of relevant API resources or metrics."

> "As a cluster lifecycle administrator, I want to easily determine whether a maintenance schedule is currently being
> enforced on my cluster by looking at the status of relevant API resources or metrics."

> "As a cluster lifecycle administrator, I want to be able to alert my operations team when changes are pending,
> when and the number of seconds to the next permitted window approaches, or when a maintenance window is not being
> enforced on my cluster."

> "As a cluster lifecycle administrator, I want to be able to diagnose why pending changes have not been applied
> if I expected them to be."

> "As a cluster administrator or privileged user familiar with OpenShift prior to the introduction of change management, 
> I want it to be clear when I am looking at the desired versus actual state of the system. For example, if I can see 
> the state of the clusterversion or a machineconfigpool, it should be straightforward to understand why I am 
> observing differences in the state of those resources compared to the state of the system."

### Goals

1. Indirectly support the strategic separation of control-plane and worker-node update phases for Standalone clusters by supplying a change control mechanism that will allow both control-plane and worker-node updates to proceed at predictable times without doubling operational overhead.
2. Directly support the strategic separation of control-plane and worker-node update phases by implementing a "manual" change management strategy where users who value the full control of the separation can manually actuate changes to them independently.
3. Empower OpenShift cluster lifecycle administrators with tools that simplify implementing industry standard notions of maintenance windows.
4. Provide Service Delivery a platform native feature which will reduce the amount of custom tooling necessary to provide maintenance windows for customers.
5. Deliver a consistent change management experience across all platforms and profiles (e.g. Standalone, ROSA, HCP).
6. Enable SRE to, when appropriate, make configuration changes on a customer cluster and have that change actually take effect only when permitted by the customer's change management preferences.
7. Do not subvert expectations of customers well served by the existing fully self-managed cluster update.
8. Ensure the architectural space for enabling different change management strategies in the future. 

### Non-Goals

1. Allowing control-plane upgrades to be paused midway through an update. Control-plane updates are relatively rapid and pausing will introduce unnecessary complexity and risk. 
2. Requiring the use of maintenance schedules for OpenShift upgrades (the changes should be compatible with various upgrade methodologies – including being manually triggered).
3. Allowing Standalone worker-nodes to upgrade to a different payload version than the control-plane (this is supported in HCP, but is not a goal for standalone).
4. Exposing maintenance schedule controls from the oc CLI. This may be a future goal but is not required by this enhancement.
5. Providing strict promises around the exact timing of upgrade processes. Maintenance schedules will be honored to a reasonable extent (e.g. upgrade actions will only be initiated during a window), but long running operations may exceed the configured end of a maintenance schedule.
6. Implementing logic to defend against impractical maintenance schedules (e.g. if a customer configures a 1 second maintenance schedule every year). Service Delivery may want to implement such logic to ensure upgrade progress can be made.

## Proposal

### Change Management Overview
Add a `changeManagement` stanza to several resources in the OpenShift ecosystem:
- HCP's `HostedCluster`. Honored by HyperShift Operator and supported by underlying CAPI primitives.
- HCP's `NodePool`. Honored by HyperShift Operator and supported by underlying CAPI primitives.
- Standalone's `ClusterVersion`. Honored by Cluster Version Operator.
- Standalone's `MachineConfigPool`. Honored by Machine Config Operator.

The implementation of `changeManagement` will vary by profile
and resource, however, they will share a core schema and provide a reasonably consistent user
experience across profiles. 

The schema will provide options for controlling exactly when changes to API resources on the 
cluster can initiate material changes to the cluster. Changes that are not allowed to be
initiated due to a change management control will be called "pending". Subsystems responsible
for initiating pending changes will await a permitted window according to the change's
relevant `changeManagement` configuration(s).

### Change Management Status
Change Management information will also be reflected in resource status. Each resource 
which contains the stanza in its `spec` will expose its current impact in its `status`. 
Common user interfaces for aggregating and displaying progress of these underlying resources
should be updated to proxy that status information to the end users.

### Change Management Metrics
Cluster wide change management information will be made available through cluster metrics. Each resource
containing the stanza should expose the following metrics:
- The number of seconds until the next known permitted change window. 0 if changes can currently be initiated. -1 if changes are paused indefinitely. -2 if no permitted window can be computed.
- Whether any change management strategy is enabled.
- Which change management strategy is enabled.
- If changes are pending due to change management controls.

### Change Management Hierarchy
Material changes to worker-nodes are constrained by change management policies in their associated resource AND 
at the control-plane resource. For example, in a standalone profile, if a MachineConfigPool's change management
policy apparently permitted material change in isolation, if material changes were not permitted by a change management
policy in the ClusterVersion resource, changes for the MachineConfigPool will not be initiated.

The design choice is informed by a thought experiment: As a cluster lifecycle administrator for a Standalone cluster,
who wants to achieve the simple goal of ensuring no material changes take place outside of a well defined 
maintenance schedule, do you want to have to the challenge of keeping every MachineConfigPool's 
`changeManagement` stanza in perfect synchronization with the ClusterVersion's? What if a new MCP is created 
without your knowledge?
 
The hierarchical approach allows a single master change management policy to be in place across 
bother the control-plane and worker-nodes. 

Conversely, material changes CAN take place on the control-plane when permitted by its associated 
change management policy even while material changes are not being permitted by worker-nodes 
policies.

### Change Management Strategies
Each resource bearing the `changeManagement` stanza must support two or more change management strategies.
Each strategy may require an additional configuration element within the stanza. For example:
```yaml
spec:
  changeManagement:
    strategy: "MaintenanceSchedule"
    maintenanceSchedule:
      ..options to configure a policy for the maintenance schedule..
```

All change management implementations must support `Disabled` and `MaintenanceSchedule`. Abstracting 
change management into strategies allows for simplified future expansion or deprecation of strategies. 
Tactically, `strategy: Disabled` provides a convenient syntax for bypassing configured 
change management policy without permanently deleting its configuration.

For example, if SRE needs to apply emergency corrective action on a cluster with a `MaintenanceSchedule` change
management strategy configured, they can simply set `strategy: Disabled` without having to delete the existing
`maintenanceSchedule` stanza which configures the previous strategy. Once the correct action has been completed,
SRE simply restores `strategy: MaintenanceSchedule` and the previous configuration begins to be enforced.

Configurations for different management strategies can be recorded in the `changeManagement` stanza, but
only one strategy can be selected as the active strategy at a given time.

#### Maintenance Schedule Strategy
The maintenance schedule strategy is supported by all resources which support change management. The strategy
allows a cluster lifecycle administrator to express one of the following semantically:

| pausedUntil    | permit | exclude | outcome                                                                                                                                                                                  |
|----------------|--------|---------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `null`/`false` | `null` | `null`  | Material changes are permitted at any time.                                                                                                                                              |
| `true`         | *      | *       | Material changes are paused indefinitely.                                                                                                                                                |
| `null`/`false` | set    | `null`  | Material changes are permitted in specified reocurring windows.                                                                                                                          |
| `null`/`false` | set    | set     | Material changes are permitted in specified reocurring windows except for specific excluded dates.                                                                                       |
| `null`/`false` | `null` | set     | Material changes are permitted except during excluded dates.                                                                                                                             |
| date           | *      | *       | Honor permit and exclude values, but only after the specified date. For example, permit: `null` and exclude: `null` implies any material changes are permitted after the specified date. |

#### Disabled Strategy
This strategy indicates that no change management strategy is being enforced by the resource. This does not always
mean that material change is permitted due to change management hierarchies. For example, a MachineConfigPool
with `strategy: Disabled` would still be subject to a `strategy: MaintenanceWindow` in the ClusterVersion resource.

#### Manual Strategy
Minimally, this strategy will be supported by MachineConfigPool. If and when the strategy is supported by other
change management capable resources, the configuration schema for the policy may differ as the details of
what constitutes and informs change varies between resources. 

#### Manual Strategy - MachineConfigPool
This strategy is motivated by the desire to support the separation of control-plane and worker-node updates both
conceptually for users and in real technical terms. One way to do this for users who do not benefit from the
`MaintenanceSchedule` strategy is to ask them to manually initiate, pause, and resume the rollout of material
changes to their worker nodes. Contrast this with the fully self-managed state today, where worker-nodes
(normally) begin to be updated automatically and directly after the control-plane update.

Clearly, if this was the only mode of updating worker-nodes, we could never successfully disentangle the
concepts of control-plane vs worker-node updates in Standalone environments since one implies the other.

In short (details will follow in the implementation section), the manual strategy allows users to specify the
exact rendered [`desiredConfig` the MachineConfigPool](https://github.com/openshift/machine-config-operator/blob/5112d4f8e562a2b072106f0336aeab451341d7dc/docs/MachineConfigDaemon.md#coordinating-updates) should be advertising to the MachineConfigDaemon on
nodes it is associated with. Like the `MaintenanceSchedule` strategy, it also exposes a `pausedUntil`
property.

### Workflow Description

Explain how the user will use the feature. Be detailed and explicit.
Describe all of the actors, their roles, and the APIs or interfaces
involved. Define a starting state and then list the steps that the
user would need to go through to trigger the feature described in the
enhancement. Optionally add a
[mermaid](https://github.com/mermaid-js/mermaid#readme) sequence
diagram.

Use sub-sections to explain variations, such as for error handling,
failure recovery, or alternative outcomes.

For example:

**cluster creator** is a human user responsible for deploying a
cluster.

**application administrator** is a human user responsible for
deploying an application in a cluster.

1. The cluster creator sits down at their keyboard...
2. ...
3. The cluster creator sees that their cluster is ready to receive
   applications, and gives the application administrator their
   credentials.

See
https://github.com/openshift/enhancements/blob/master/enhancements/workload-partitioning/management-workload-partitioning.md#high-level-end-to-end-workflow
and https://github.com/openshift/enhancements/blob/master/enhancements/agent-installer/automated-workflow-for-agent-based-installer.md for more detailed examples.

### API Extensions

API Extensions are CRDs, admission and conversion webhooks, aggregated API servers,
and finalizers, i.e. those mechanisms that change the OCP API surface and behaviour.

- Name the API extensions this enhancement adds or modifies.
- Does this enhancement modify the behaviour of existing resources, especially those owned
  by other parties than the authoring team (including upstream resources), and, if yes, how?
  Please add those other parties as reviewers to the enhancement.

  Examples:
  - Adds a finalizer to namespaces. Namespace cannot be deleted without our controller running.
  - Restricts the label format for objects to X.
  - Defaults field Y on object kind Z.

Fill in the operational impact of these API Extensions in the "Operational Aspects
of API Extensions" section.

### Topology Considerations

#### Hypershift / Hosted Control Planes

Are there any unique considerations for making this change work with
Hypershift?

See https://github.com/openshift/enhancements/blob/e044f84e9b2bafa600e6c24e35d226463c2308a5/enhancements/multi-arch/heterogeneous-architecture-clusters.md?plain=1#L282

How does it affect any of the components running in the
management cluster? How does it affect any components running split
between the management cluster and guest cluster?

#### Standalone Clusters

Is the change relevant for standalone clusters?

#### Single-node Deployments or MicroShift

How does this proposal affect the resource consumption of a
single-node OpenShift deployment (SNO), CPU and memory?

How does this proposal affect MicroShift? For example, if the proposal
adds configuration options through API resources, should any of those
behaviors also be exposed to MicroShift admins through the
configuration file for MicroShift?

### Implementation Details/Notes/Constraints

What are some important details that didn't come across above in the
**Proposal**? Go in to as much detail as necessary here. This might be
a good place to talk about core concepts and how they relate. While it is useful
to go into the details of the code changes required, it is not necessary to show
how the code will be rewritten in the enhancement.

### Risks and Mitigations

What are the risks of this proposal and how do we mitigate. Think broadly. For
example, consider both security and how this will impact the larger OKD
ecosystem.

How will security be reviewed and by whom?

How will UX be reviewed and by whom?

Consider including folks that also work outside your immediate sub-project.

### Drawbacks

The idea is to find the best form of an argument why this enhancement should
_not_ be implemented.

What trade-offs (technical/efficiency cost, user experience, flexibility,
supportability, etc) must be made in order to implement this? What are the reasons
we might not want to undertake this proposal, and how do we overcome them?

Does this proposal implement a behavior that's new/unique/novel? Is it poorly
aligned with existing user expectations?  Will it be a significant maintenance
burden?  Is it likely to be superceded by something else in the near future?

## Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance,
 > 1. This requires exposing previously private resources which contain sensitive
  information.  Can we do this?

## Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?
- What additional testing is necessary to support managed OpenShift service-based offerings?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

## Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- Maturity levels
  - [`alpha`, `beta`, `stable` in upstream Kubernetes][maturity-levels]
  - `Dev Preview`, `Tech Preview`, `GA` in OpenShift
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, GA), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

**If this is a user facing change requiring new or updated documentation in [openshift-docs](https://github.com/openshift/openshift-docs/),
please be sure to include in the graduation criteria.**

**Examples**: These are generalized examples to consider, in addition
to the aforementioned [maturity levels][maturity-levels].

### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers
- Enumerate service level indicators (SLIs), expose SLIs as metrics
- Write symptoms-based alerts for the component(s)

### Tech Preview -> GA

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default
- Backhaul SLI telemetry
- Document SLOs for the component
- Conduct load testing
- User facing documentation created in [openshift-docs](https://github.com/openshift/openshift-docs/)

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

## Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

Upgrade expectations:
- Each component should remain available for user requests and
  workloads during upgrades. Ensure the components leverage best practices in handling [voluntary
  disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/). Any exception to
  this should be identified and discussed here.
- Micro version upgrades - users should be able to skip forward versions within a
  minor release stream without being required to pass through intermediate
  versions - i.e. `x.y.N->x.y.N+2` should work without requiring `x.y.N->x.y.N+1`
  as an intermediate step.
- Minor version upgrades - you only need to support `x.N->x.N+1` upgrade
  steps. So, for example, it is acceptable to require a user running 4.3 to
  upgrade to 4.5 with a `4.3->4.4` step followed by a `4.4->4.5` step.
- While an upgrade is in progress, new component versions should
  continue to operate correctly in concert with older component
  versions (aka "version skew"). For example, if a node is down, and
  an operator is rolling out a daemonset, the old and new daemonset
  pods must continue to work correctly even while the cluster remains
  in this partially upgraded state for some time.

Downgrade expectations:
- If an `N->N+1` upgrade fails mid-way through, or if the `N+1` cluster is
  misbehaving, it should be possible for the user to rollback to `N`. It is
  acceptable to require some documented manual steps in order to fully restore
  the downgraded cluster to its previous state. Examples of acceptable steps
  include:
  - Deleting any CVO-managed resources added by the new version. The
    CVO does not currently delete resources that no longer exist in
    the target version.

## Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet.

## Operational Aspects of API Extensions

Describe the impact of API extensions (mentioned in the proposal section, i.e. CRDs,
admission and conversion webhooks, aggregated API servers, finalizers) here in detail,
especially how they impact the OCP system architecture and operational aspects.

- For conversion/admission webhooks and aggregated apiservers: what are the SLIs (Service Level
  Indicators) an administrator or support can use to determine the health of the API extensions

  Examples (metrics, alerts, operator conditions)
  - authentication-operator condition `APIServerDegraded=False`
  - authentication-operator condition `APIServerAvailable=True`
  - openshift-authentication/oauth-apiserver deployment and pods health

- What impact do these API extensions have on existing SLIs (e.g. scalability, API throughput,
  API availability)

  Examples:
  - Adds 1s to every pod update in the system, slowing down pod scheduling by 5s on average.
  - Fails creation of ConfigMap in the system when the webhook is not available.
  - Adds a dependency on the SDN service network for all resources, risking API availability in case
    of SDN issues.
  - Expected use-cases require less than 1000 instances of the CRD, not impacting
    general API throughput.

- How is the impact on existing SLIs to be measured and when (e.g. every release by QE, or
  automatically in CI) and by whom (e.g. perf team; name the responsible person and let them review
  this enhancement)

- Describe the possible failure modes of the API extensions.
- Describe how a failure or behaviour of the extension will impact the overall cluster health
  (e.g. which kube-controller-manager functionality will stop working), especially regarding
  stability, availability, performance and security.
- Describe which OCP teams are likely to be called upon in case of escalation with one of the failure modes
  and add them as reviewers to this enhancement.

## Support Procedures

Describe how to
- detect the failure modes in a support situation, describe possible symptoms (events, metrics,
  alerts, which log output in which component)

  Examples:
  - If the webhook is not running, kube-apiserver logs will show errors like "failed to call admission webhook xyz".
  - Operator X will degrade with message "Failed to launch webhook server" and reason "WehhookServerFailed".
  - The metric `webhook_admission_duration_seconds("openpolicyagent-admission", "mutating", "put", "false")`
    will show >1s latency and alert `WebhookAdmissionLatencyHigh` will fire.

- disable the API extension (e.g. remove MutatingWebhookConfiguration `xyz`, remove APIService `foo`)

  - What consequences does it have on the cluster health?

    Examples:
    - Garbage collection in kube-controller-manager will stop working.
    - Quota will be wrongly computed.
    - Disabling/removing the CRD is not possible without removing the CR instances. Customer will lose data.
      Disabling the conversion webhook will break garbage collection.

  - What consequences does it have on existing, running workloads?

    Examples:
    - New namespaces won't get the finalizer "xyz" and hence might leak resource X
      when deleted.
    - SDN pod-to-pod routing will stop updating, potentially breaking pod-to-pod
      communication after some minutes.

  - What consequences does it have for newly created workloads?

    Examples:
    - New pods in namespace with Istio support will not get sidecars injected, breaking
      their networking.

- Does functionality fail gracefully and will work resume when re-enabled without risking
  consistency?

  Examples:
  - The mutating admission webhook "xyz" has FailPolicy=Ignore and hence
    will not block the creation or updates on objects when it fails. When the
    webhook comes back online, there is a controller reconciling all objects, applying
    labels that were not applied during admission webhook downtime.
  - Namespaces deletion will not delete all objects in etcd, leading to zombie
    objects when another namespace with the same name is created.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used
to highlight and record other possible approaches to delivering the
value proposed by an enhancement, including especially information
about why the alternative was not selected.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.
