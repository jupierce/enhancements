---
title: change-management-and-maintenance-schedules
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

# Change Management and Maintenance Schedules

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
both the control-plane and worker-nodes. 

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

#### Assisted Strategy - MachineConfigPool
Minimally, this strategy will be supported by MachineConfigPool. If and when the strategy is supported by other
change management capable resources, the configuration schema for the policy may differ as the details of
what constitutes and informs change varies between resources. 

This strategy is motivated by the desire to support the separation of control-plane and worker-node updates both
conceptually for users and in real technical terms. One way to do this for users who do not benefit from the
`MaintenanceSchedule` strategy is to ask them to initiate, pause, and resume the rollout of material
changes to their worker nodes. Contrast this with the fully self-managed state today, where worker-nodes
(normally) begin to be updated automatically and directly after the control-plane update.

Clearly, if this was the only mode of updating worker-nodes, we could never successfully disentangle the
concepts of control-plane vs worker-node updates in Standalone environments since one implies the other.

In short (details will follow in the implementation section), the assisted strategy allows users to specify the
exact rendered [`desiredConfig` the MachineConfigPool](https://github.com/openshift/machine-config-operator/blob/5112d4f8e562a2b072106f0336aeab451341d7dc/docs/MachineConfigDaemon.md#coordinating-updates) should be advertising to the MachineConfigDaemon on
nodes it is associated with. Like the `MaintenanceSchedule` strategy, it also exposes a `pausedUntil`
property.

#### Manual Strategy - MachineConfigPool
Minimally, this strategy will be supported by MachineConfigPool. If and when the strategy is supported by other
change management capable resources, the configuration schema for the policy may differ as the details of
what constitutes and informs change varies between resources. 

Like the Assisted strategy, this strategy is implemented to support the conceptual and technical separation
of control-plane and worker-nodes. The MachineConfigPool Manual strategy allows users to explicitly specify
their `desiredConfig` to be used for ignition of new and rebooting nodes. While the Manual strategy is enabled,
the MachineConfigOperator will not trigger the MachineConfigDaemon to drain or reboot nodes automatically.


### Workflow Description

#### OCM HCP Standard Change Management Scenario

1. A [Cluster Service Consumer](https://hypershift-docs.netlify.app/reference/concepts-and-personas/#personas) requests an HCP cluster via OCM.
1. To comply with their company policy, the service consumer configures a maintenance schedule through OCM. 
1. Their first preference, no updates at all, is rejected by OCM policy and they are referred to service 
   delivery documentation explaining minimum requirements. 
1. The user specifies a policy which permits changes to be initiated any time Saturday UTC on the control-plane.
1. To limit perceived risk, they try to specify a separate policy permitting worker-nodes updates only on the **first** Sunday of each month.
1. OCM rejects the configuration because, due to change management hierarchy, worker-node maintenance schedules can only be a proper subset of control-plane maintenance schedules.
1. The user changes their preference to a policy permitting worker-nodes updates only on the **first** Saturday of each month.
1. OCM accepts the configuration.
1. OCM configures the HCP (HostedCluster/NodePool) resources via the Service Delivery Hive deployment to contain a `changeManagement` stanza 
   and an active/configured `MaintenanceSchedule` strategy.
1. Hive updates the associated HCP resources.
1. Company workloads are added to the new cluster and the cluster provides value.
1. To leverage a new feature in OpenShift, the service consumer plans to update the minor version of the platform.
1. Via OCM, the service consumer requests the minor version update. They can do this at any time with confidence that the maintenance 
   schedule will be honored. They do so on Wednesday.
1. OCM (through various layers) updates the target release payload in the HCP HostedCluster and NodePool.
1. The HyperShift Operator detects the desired changes but recognizes that the `changeManagement` stanza 
    precludes the updates from being initiated.
1. Curious, the service consumer checks the projects ClusterVersion within the HostedCluster and reviews its `status` stanza. It shows that changes are pending and the time of the next window in which changes can be initiated.  
1. Separate metrics specific to change management indicate that changes are pending for both resources.
1. The non-Red Hat operations team has alerts setup to fire when changes are pending and the number of 
    seconds before the next permitted window is less than 2 days away.
1. These alerts fire after Thursday UTC 00:00 to inform the operations team that changes are about to be applied to the control-plane.
1. It is not the first week of the month, so there is no alert fired for the NodePool pending changes.
1. The operations team is comfortable with the changes being rolled out on the control-plane. 
1. On Saturday 00:00 UTC, the HyperShift operator initiates changes the control-plane update.
1. The update completes without issue.
1. Changes remain pending for the NodePool resource.
1. As the first Saturday of the month approaches, the operations alerts fire to inform the team of forthcoming changes.
1. The operations team realizes that a corporate team needs to use the cluster heavily during the weekend for a business critical deliverable.
1. The service consumer logs into OCM and adds an exclusion for the upcoming Saturday.
1. Interpreting the new exclusion, the metric for time remaining until a permitted window increases to count down to the following month's first Saturday.
1. A month passes and the pending cause the configured alerts to fire again.
1. The operations team is comfortable with the forthcoming changes.
1. The first Saturday of the month 00:00 UTC arrives. The HyperShift operator initiates the worker-node updates based on the pending changes in the cluster NodePool.
1. The HCP cluster has a large number of worker nodes and draining and rebooting them is time-consuming. 
1. At 23:59 UTC Saturday night, 80% of worker-nodes have been updated. Since the maintenance schedule still permits the initiation of material changes, another worker-node begins to be updated.
1. The update of this worker-node continues, but at 00:00 UTC Sunday, no further material changes are permitted by the change management policy and the worker-node update process is effectively paused.
1. Because not all worker-nodes have been updated, changes are still reported as pending via metrics for NodePool.  **TODO: Review with HyperShift. Pausing progress should be possible, but a metric indicating changes still pending may not since they interact only through CAPI.**
1. The HCP cluster runs with worker-nodes at mixed versions throughout the month. The N-1 skew between the old kubelet versions and control-plane is supported.
1. **TODO: Review with Service Delivery. If the user requested another minor bump to their control-plane, how does OCM prevent unsupported version skew today?**
1. On the next first Saturday, the worker-nodes updates are completed.

#### OCM Standalone Standard Change Management Scenario

1. User interactions with OCM to configure a maintenance schedule are identical to [OCM HCP Standard Change Management Scenario](#ocm-hcp-standard-change-management-scenario).
   This scenario differs after OCM accepts the maintenance schedule configuration. Control-plane updates are permitted to be initiated to any Saturday UTC.
   Worker-nodes must wait until the first Saturday of the month.
1. OCM (through various layers) configures the ClusterVersion and worker MachineConfigPool(s) (MCP) for the cluster with appropriate `changeManagement` stanzas.
1. Company workloads are added to the new cluster and the cluster provides value.
1. To leverage a new feature in OpenShift, the service consumer plans to update the minor version of the platform.
1. Via OCM, the service consumer requests the minor version update. They can do this at any time with confidence that the maintenance 
   schedule will be honored. They do so on Wednesday.
1. OCM (through various layers) updates the ClusterVersion resource on the cluster indicating the new release payload in `desiredUpdate`.
1. The Cluster Version Operator (CVO) detects that its `changeManagement` stanza does not permit the initiation of the change.
1. The CVO sets a metric indicating that changes are pending for ClusterVersion. Irrespective of pending changes, the CVO also exposes a 
   metric indicating the number of seconds until the next window in which material changes can be initiated.
1. Since MachineConfigs do not match in the desired update and the current manifests, the CVO also sets a metric indicating that MachineConfig
   changes are pending. This is done because the MachineConfigOperator (MCO) cannot anticipate the coming manifest changes and cannot,
   therefore, reflect expected changes to the worker-node MCPs. Anticipating this change ahead of time is necessary for an operation
   team to be able to set an alert with the semantics (worker-node-update changes are pending & time remaining until changes are permitted < 2d).
   The MCO will expose its own metric for changes pending when manifests are updated. But this metric will only indicate when 
   there are machines in the pool that have not achieved the desired configuration. An operations team trying to implement the 2d 
   early warning for worker-nodes must use OR on these metrics to determine whether changes are actually pending.
1. The MCO, irrespective of pending changes, exposes a metric for each MCP to indicate the number of seconds remaining until it is
   permitted to initiate changes to nodes in that MCP.
1. A privileged user on the cluster notices different options available for `changeManagement` in the ClusterVersion and MachineConfigPool
   resources. They try to set them but are prevented by either RBAC or an admission webhook (details for Service Delivery). If they wish
   to change the settings, they must update them through OCM.
1. The privileged user does an `oc describe ...` on the resources. They can see that material changes are pending in ClusterVersion for 
   the control-plane and for worker machine config. They can also see the date and time that the next material change will be permitted.
   The MCP will not show a pending change at this time, but will show the next time at which material changes will be permitted.
1. The next Saturday is _not_ the first Saturday of the month. The CVO detects that material changes are permitted at 00:00 UTC and
   begins to apply manifests. This effectively initiates the control-plane update process, which is considered a single
   material change to the cluster.
1. The control-plane update succeeds. The CVO, having reconciled its state, unsets metrics suggesting changes are pending.
1. As part of updating cluster manifests, MachineConfigs have been modified. The MachineConfigOperator (MCO) re-renders a
   configuration for worker-nodes. However, because the MCP maintenance schedule precludes initiating material changes,
   it will not begin to update Machines with that desired configuration.
1. The MCO will set a metric indicating that desired changes are pending. 
1. `oc get -o=yaml/describe` will both provide status information indicating that changes are pending for the MCP and
   the time at which the next material changes can be initiated according to the maintenance schedule.
1. On the first Saturday of the next month, 00:00 UTC, the MCO determines that material changes are permitted.
   Based on limits like maxUnavailable, the MCO begins to annotate nodes with the desiredConfiguration. The 
   MachineConfigDaemon takes over from there, draining, and rebooting nodes into the updated release.
1. There are a large number of nodes in the cluster and this process continues for more than 24 hours. On Saturday
   23:59, the MCO applies a round of desired configurations annotations to Nodes. At 00:00 on Sunday, it detects
   that material changes can no longer be initiated, and pauses its activity. Node updates that have already
   been initiated continue beyond the maintenance schedule window.
1. Since not all nodes have been updated, the MCO continues to expose a metric informing the system of
   pending changes.
1. In the subsequent days, the cluster is scaled up to handle additional workload. The new nodes receive
   the most recent, desired configuration.    
1. On the first Saturday of the next month, the MCO resumes its work. In order to ensure that forward progress is
   made for all nodes, the MCO will update nodes that have the oldest current configuration first. This ensures
   that even if the desired configuration has changed multiple times while maintenance was not permitted,
   no nodes are starved of updates. Consider the alternative where (a) worker-node updates required > 24h,
   (b) updates to nodes are performed alphabetically, and (c) MachineConfigs are frequently being changed
   during times when maintenance is not permitted. This strategy could leave nodes sorting last 
   lexicographically no opportunity to receive updates. This scenario would eventually leave those nodes
   more prone to version skew issues.
1. During this window of time, all node updates are initiated and they complete successfully. 

#### Service Delivery Emergency Patch
1. SRE determines that a significant new CVE threatens the fleet.
1. A new OpenShift release in each z-stream fixes the problem.
1. SRE plans to override customer maintenance schedules in order to rapidly remediate the problem across the fleet.
1. The new OpenShift release(s) are configured across the fleet. Clusters with permissive maintenance 
   schedules begin to apply the changes immediately.
1. Clusters with change management policies precluding updates are SRE's next focus.
1. During each region's evening hours, to limit disruption, SRE changes the `changeManagement` strategy 
   field across relevant resources to `Disabled`. Changes that were previously pending are now
   permitted to be initiated. 
1. Cluster operators who have alerts configured to fire when there is no change management policy in place 
   will do so.
1. As clusters are successfully remediated, SRE restores the `MaintenanceSchedule' strategy for its resources.
   

#### Service Delivery Immediate Remediation
1. A customer raises a ticket for a problem that is eventually determined to be caused by a worker-node system configuration.
1. SRE can address the issue with a system configuration file applied in a MachineConfig.
1. SRE creates the MachineConfig for the customer and provides the customer the option to either (a) wait until their
   configured maintenance schedule permits the material change from being initiated by the MachineConfigOperator
   or (b) having SRE override the maintenance schedule and permitting its immediate application.   
1. The customer chooses immediate application. 
1. SRE applies a change to the relevant control-plane AND worker-node resource's `changeManagement` stanza
   (both must be changed because of the change management hierarchy), setting `disabledUntil` to
   a time 48 hours in the future. The configured change management schedule is ignored for 48 as the system 
   initiates all necessary node changes.

#### Service Delivery Deferred Remediation
1. A customer raises a ticket for a problem that is eventually determined to be caused by a worker-node system configuration.
1. SRE can address the issue with a system configuration file applied in a MachineConfig.
1. SRE creates the MachineConfig for the customer and provides the customer the option to either (a) wait until their
   configured maintenance schedule permits the material change from being initiated by the MachineConfigOperator
   or (b) having SRE override the maintenance schedule and permitting its immediate application.   
1. The problem is not pervasive, so the customer chooses the deferred remediation. 
1. The change is initiated and nodes are rebooted during the next permitted maintenance window.


#### On-prem Standalone GitOps Change Management Scenario
1. An on-prem cluster is fully managed by gitops. As changes are committed to git, those changes are applied to cluster resources.
1. Configurable stanzas of the ClusterVersion and MachineConfigPool(s) resources are checked into git.
1. The cluster lifecycle administrator configures `changeManagement` in both the ClusterVersion and worker MachineConfigPool
   in git. The MaintenanceSchedule strategy is chosen. The policy permits control-plane and worker-node updates only after
   19:00 Eastern US.
1. During the working day, users may contribute and merge changes to MachineConfigs or even the `desiredUpdate` of the
   ClusterVersion. These resources will be updated in a timeline manner via GitOps.
1. Despite the resource changes, neither the CVO nor MCO will begin to initiate the material changes on the cluster.
1. Privileged users who may be curious as to the discrepancy between git and the cluster state can use `oc get -o=yaml/describe`
   on the resources. They observe that changes are pending and the time at which changes will be initiated.
1. At 19:00 Eastern, the pending changes begin to be initiated. This rollout abides by documented OpenShift constraints
   such as the MachineConfigPool `maxUnavailable` setting.
   
#### On-prem Standalone Manual Strategy Scenario
1. A small, business critical cluster is being run on-prem.
1. There are no reoccurring windows of time when the cluster lifecycle administrator can tolerate downtime.
   Instead, updates are negotiated and planned far in advance. 
1. The cluster workloads are not HA and unplanned drains are considered a business risk.
1. To prevent surprises, the cluster lifecycle administrator sets the Manual strategy on the worker MCP.
1. Given the sensitivity of the operation, the lifecycle administrator wants to manually drain and reboot
   nodes to accomplish the update.
1. The cluster lifecycle administrator sends a company-wide notice about the period during which service may be disrupted.
1. The user determines the most recent rendered worker configuration. They configure the `manual` change
   management policy to use that exact configuration as the `desiredConfig`. 
1. The MCO is thus being asked to ignite any new node or rebooted node with the desired configuration, but it
   is **not** being permitted to apply that configuration to existing nodes because it is change management, in effect,
   is paused indefinitely by the manual strategy.
1. The MCO metric for the MCP indicating the number of seconds remaining until changes can be initiated is `-1` - indicating
   that there is presently no time in the future where it will initiate material changes. The operations team
   has an alert configured if this value `!= -1`.
1. The MCO metric for the MCP indicating that changes are pending is set because not all nodes are running
   the most recently rendered configuration. This is irrespective of the `desiredConfig` in the `manual` 
   policy. Abstractly, it means, if change management were disabled, whether changes be initiated.
1. The cluster lifecycle administrator manually drains and reboots nodes in the cluster. As they come back online,
   the MachineConfigServer offers them the desiredConfig requested by the manual policy.
1. After updating all nodes, the cluster lifecycle administrator does not need make any additional 
   configuration changes. They can leave the `changeManagement` stanza in their MCP as-is.
   
#### On-prem Standalone Assisted Strategy Scenario
1. A large, business critical cluster is being run on-prem.
1. There are no reoccurring windows of time when the cluster lifecycle administrator can tolerate downtime.
   Instead, updates are negotiated and planned far in advance. 
1. The cluster workloads are not HA and unplanned drains are considered a business risk.
1. To prevent surprises, the cluster lifecycle administrator sets the Assisted strategy on the worker MCP.
1. In the `assisted` strategy change management policy, the lifecycle administrator configures `pausedUntil: true`
   and the most recently rendered worker configuration in the policy's `desiredConfig`.
1. The MCO is being asked to ignite any new node or any rebooted node with the desired configuration. However,
   becaused of `pausedUntil: true`, it is also being asked not to automatically initiate that material change
   for existing nodes.
1. The MCO metric for the MCP indicating the number of seconds remaining until changes can be initiated is `-1` - indicating
   that there is presently no time in the future where it will initiate material changes. The operations team
   has an alert configured if this value `!= -1`.
1. The MCO metric for the MCP indicating that changes are pending is set because not all nodes are running
   the most recently rendered configuration. This is irrespective of the `desiredConfig` in the `manual` 
   policy. Abstractly, it means, if change management were disabled, whether changes be initiated.
1. When the lifecycle administrator is ready to permit disruption, they set `pausedUntil: false`.
1. The MCO sets the number of seconds until changes are permitted to `0`.
1. The MCO begins to initiate worker node updates. This rollout abides by documented OpenShift constraints
   such as the MachineConfigPool `maxUnavailable` setting.
   
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

In the HCP topology, the HostedCluster and NodePool resources are enhanced to support the change management strategies
`MaintenanceSchedule` and `Disabled`. 

#### Standalone Clusters

In the Standalone topology, the ClusterVersion and MachineConfigPool resources are enhanced to support the change management strategies
`MaintenanceSchedule` and `Disabled`. The MachineConfigPool also supports the `Manual` strategy. 

While the master MCP in a Standalone cluster can be configured with a change management policy, updates to the 
control-plane will ignore this policy since it is treated as a single material change to the cluster. Specifically,
is the `osImageURL` differs between the desired and current configuration for master nodes, the change management
policy will be ignored. For other changes, such as a systemd unit added by a user, a maintenance schedule
can constrain its application.

#### Single-node Deployments or MicroShift

The ClusterVersion operator will honor the change management field just as in a standalone profile. If those profiles
have the a MachineConfigPool, material changes the node could be controlled with a change management policy
in that resource.

#### OCM Managed Instances
OpenShift Cluster Manager (OCM) should expose a user interface allowing users to manage their change management policy.
Standard Fleet clusters will expose the option to configure the MaintenanceSchedule strategy - including
only permit and exclude times.

- Service Delivery will reserve the right to disable this strategy for emergency corrective actions.
- Service Delivery should constrain permit & exclude configurations based on their internal policies. For example, customers may be forced to enable maintenance windows amount to at least 6 hours a month.

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
