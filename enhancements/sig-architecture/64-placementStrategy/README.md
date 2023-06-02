# Placement Strategy

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

The proposed work would define a new field in the Placement API (DecisionStrategy) that will let the end user able to break down the selected managedClusters to groups of clusters.

## Motivation

In large scale or production ready environment applying workloads required progressive roll out. End user uses placement API to select managed clusters then OCM applier APIs such as ManifestWorkReplicaSet, ClusterManagementAddon or Policy use the placement API to identify the selected clusters then apply the workloads. Lately many use-case ask for progressive roll out which makes the applier APIs in needs to define the roll out strategy. For example; ClusterManagementAddon has [installStrategy](https://github.com/open-cluster-management-io/api/blob/main/addon/v1alpha1/types_clustermanagementaddon.go#L60) field, Policy API consumer operator such as TALM has [clusterSelector](https://github.com/openshift-kni/cluster-group-upgrades-operator/blob/main/api/v1alpha1/clustergroupupgrade_types.go#L128) field and there is an urge in ManifestWorkReplicaSet APIs to do as well. As end user it make sense to define the roll out strategy in the placement APIs then OCM API such as ManifestWorkReplicaSet, ClusterManagementAddon and Policy use this roll out strategy to apply the workloads.

### Goals

1. Add a new field decisionStrategy to the placement APIs to let end uer define the roll out strategy for the selected clusters.

### Non-Goals

1. The new Placement APIs field decisionStrategy has no control over applying the workloads.
1. The new Placement APIs field decisionStrategy has no control over scheduling the workloads.

## Proposal

### User Stories

#### Story 1: As end user I want to define a subset of the selected clusters and label the placement decision to identify the cluster decision.
#### Story 2: As end user I want to divide the selected clusters to groups then a workloads applier APIs can use those cluster groups to roll out the workloads. 

### Placement API

PlacementStrategy's propose DecisionStrategy field to the Placement API;
1. DecisionStrategy divide the created placement decisions into groups. The DecisionStrategy's ClustersPerDecisionGroup field set the max number of clusters exist in a decision group as fixed number or percentage of the total number of selected clusters. As example; for a total 100 clusters selected, ClustersPercentagePerDecisionGroup equal to 20% will divide the placement decisions into 5 groups each group consist of 20 clusters. ClustersPerDecisionGroup default value is 100% meaning all selected clusters will be in a single group. The DecisionStrategy's DecisionGroups field is used to identify a subset of the placement decision based on cluster selector. Each DecisionGroup has its own group name and clusterSelector that will identify the placementDecision.

```go
type PlacementSpec struct {
	// ClusterSets represent the ManagedClusterSets from which the ManagedClusters are selected.
	// +optional
	ClusterSets []string `json:"clusterSets,omitempty"`

	// Predicates represent a slice of predicates to select ManagedClusters. The predicates are ORed.
	// +optional
	Predicates []ClusterPredicate `json:"predicates,omitempty"`
            ...
 
	// Decision Strategy divide the created placement decision to groups and define number of clusters per decision group.
	// +optional
	DecisionStrategy DecisionStrategy `json:"decisionStrategy,omitempty"`
}

// Subset of the created placementDecision with label.
type DecisionGroup struct {
	// group name to be added to the created placement Decisions labels
	groupName string  `json:"GroupName,omitempty"`

  // LabelSelector to select clusters subset by label.
	// +optional
	ClusterSelector metav1.LabelSelector `json:"clusterSelector,omitempty"`
}

// Group the created placementDecision into decision groups based on the number of clusters per decision group.
type DecisionStrategy struct {
	// Define subsets of the created placement decisions and add labels to the placementDecision.
	// +optional
	DecisionGroups []DecisionGroup `json:"decisionGroups,omitempty"`

	// ClusterPerDecisionGroup is a fixed number or percentage of the total selected clusters that will divide the clusters to groups each group has the max number of clusters equal to or less than the ClusterPerDecisionGroup value.
	// ex; for a total 100 clusters selected, ClusterPerDecisionGroup equal to 20% will divide the placement decision to 5 groups each group should have 20 clusters.
	// +kubebuilder:validation:Enum=None;5%;10%;15%;20%;25%;30%;40%;50%;100%
	// +kubebuilder:default:="100%"
	// +optional
	ClusterPerDecisionGroup intOrString `json:"clustersPerDecisionGroup,omitempty"`
}

```

Adding DecisionGroupStatus to the PlacementStatus API 
```go
// Present decision groups status based on the DecisionStrategy definition.
type DecisionGroupStatus struct {
	// Present the decision group index. If there is no decision strategy defined all placement decisions will be in group index 0
	// +optional
	DecisionGroupIndex int32 `json:"decisionGroupIndex"`

	// Decision group name that is defined in the DecisionStrategy's DecisionGroup.
	// +optional
	DecisionGroupName string `json:"decisionGroupName"`

	// List of placement decisions names associated with the decision group
	// +optional
	Decisions []string `json:"decisions"`

	// Total number of clusters in the decision group. Clusters count is equal or less than the max number of clusters per decision group defined in the decision strategy.
	// +kubebuilder:default:=0
	// +optional
	ClustersCount int32 `json:"clusterCount"`
}

type PlacementStatus struct {
	// List of decision groups determined by the placement and placement strategy.
	// +optional
	DecisionGroups []DecisionGroupStatus `json:"decisionGroups"`

	// NumberOfSelectedClusters represents the number of selected ManagedClusters
	// +optional
	NumberOfSelectedClusters int32 `json:"numberOfSelectedClusters"`

	// Conditions contains the different condition status for this Placement.
	// +optional
	Conditions []metav1.Condition `json:"conditions"`
}
```
#### PlacementDecision API labels 
A Placement can be linked to multiple PlacementDecisions, Adding DecisionGroupIndexLabel to determine the selected placementDecisions belong to which decision group. decisionGroupName value is the group name defined in the DecisionStrategy's DecisionGroup. The DecisionGroupIndex increase incrementally based on the ClustersPerDecisionGroup and total number of selected clusters. If there is no DecisionStrategy defined all PlacementDecisions will have DecisionGroupIndex value 0.

```go
const (
	PlacementLabel string = "cluster.open-cluster-management.io/placement"
	// decision group index.
	DecisionGroupIndexLabel string = "cluster.open-cluster-management.io/decision-group-index"
	// decision group name.
	DecisionGroupNameLabel  string = "cluster.open-cluster-management.io/decision-group-name"
)
```

## Design and Implementation Details

The placement controller will determine the decision groups based on 1) The number of total selected clusters. 2) The number or percentage defined in ClustersPerDecisionGroup. 3) The DecisionGroup defined in the DecisionStrategy. The placement controller will label the created placementDecisions with DecisionGroupIndex to relate the placementDecision with the decision groups. The placementDecisions will have decisionGroupName label that is defined in the DecisionStrategy's DecisionGroup.

### Examples

Example for a placement has DecisionStrategy defined with two decisionGroups and ClustersPerDecisionGroup equal to 150.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: ztp-placement
  namespace: ztp-acm-ns
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: common-profile
              operator: In
              values:
                - 'true'
  DecisionStrategy:
    clustersPerDecisionGroup: 150
    DecisionGroups:
    - groupName: prod-canary-west
      clusterSelector:
        matchExpressions:
          - key: prod-canary-west
            operator: Exist
    - groupName: prod-canary-east
      clusterSelector:
        matchExpressions:
          - key: prod-canary-east
            operator: Exist
```

With 320 (290 clusters + 10 prod-canary-west clusters + 10 prod-canary-east clusters) clusters selected based on the placements predicates and the defined decision Strategy there will be 4 decision groups; 
1) Decision group 1 has decisionGroupIndex=0 hold 10 clusters based on the prod-canary-west label, 2) Decision group 2 has decisionGroupIndex=1 hold 10 clusters based on the prod-canary-east label, 3) Decision group 3 has ClusterGroupIndex=2 hold 150 clusters and 4) Decision group 4 has ClusterGroupIndex=3 hold 140 clusters (the rest of total 320 clusters are selected). The order of the DecisionGroups defined under the DecisionStrategy is reflected in the DecisionGroupIndex label value.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: ztp-placement
  namespace: ztp-acm-ns
 ...
status:
  numberOfSelectedClusters: 320
  decisionGroups:
  - decisionGroupIndex: 0
    decisionGroupName: prod-canary-west
    placementDecisions:
    - ztp-placement-decision-0
    clusterCount: 10
  - decisionGroupIndex: 1
    decisionGroupName: prod-canary-east
    placementDecisions:
    - ztp-placement-decision-1
    clusterCount: 10
  - decisionGroupIndex: 2
    placementDecisions:
    - ztp-placement-decision-2
    - ztp-placement-decision-3
    clusterCount: 150
  - decisionGroupIndex: 3
    placementDecisions:
    - ztp-placement-decision-4
    - ztp-placement-decision-5
    clusterCount: 140      
  conditions:
    - lastTransitionTime: '2023-02-16T18:32:05Z'
      message: All cluster decisions scheduled
      reason: AllDecisionsScheduled
      status: 'True'
      type: PlacementSatisfied
```  
The created placementDecisions will have DecisionGroupIndex label and decisionGroupName label as below.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: PlacementDecision
metadata:
  name: ztp-placement-decision-0
  namespace: ztp-acm-ns
  labels:
    cluster.open-cluster-management.io/placement: ztp-placement
    cluster.open-cluster-management.io/decisiongroupindex: 0
    cluster.open-cluster-management.io/decisiongroupname: "prod-canary-wast"
status:
  decisions:
    - clusterName: cls001
      reason: ''
    ...
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: PlacementDecision
metadata:
  name: ztp-placement-decision-1
  namespace: ztp-acm-ns
  labels:
    cluster.open-cluster-management.io/placement: ztp-placement
    cluster.open-cluster-management.io/decisiongroupindex: 1
    cluster.open-cluster-management.io/decisiongroupname: "prod-canary-east"
status:
  decisions:
    - clusterName: cls006
      reason: ''
    ...
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: PlacementDecision
metadata:
  name: ztp-placement-decision-2
  namespace: ztp-acm-ns
  labels:
    cluster.open-cluster-management.io/placement: ztp-placement
    cluster.open-cluster-management.io/decisiongroupindex: 2
status:
  decisions:
    - clusterName: cls011
      reason: ''
    ...
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: PlacementDecision
metadata:
  name: ztp-placement-decision-3
  namespace: ztp-acm-ns
  labels:
    cluster.open-cluster-management.io/placement: ztp-placement
    cluster.open-cluster-management.io/decisiongroupindex: 2
status:
  decisions:
    - clusterName: cls011
      reason: ''
    ...
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: PlacementDecision
metadata:
  name: ztp-placement-decision-4
  namespace: ztp-acm-ns
  labels:
    cluster.open-cluster-management.io/placement: ztp-placement
    cluster.open-cluster-management.io/decisiongroupindex: 3
status:
  decisions:
    - clusterName: cls151
      reason: ''
    ...
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: PlacementDecision
metadata:
  name: ztp-placement-decision-5
  namespace: ztp-acm-ns
  labels:
    cluster.open-cluster-management.io/placement: ztp-placement
    cluster.open-cluster-management.io/decisiongroupindex: 3
status:
  decisions:
    - clusterName: cls151
      reason: ''
    ...
```

Another example for a placement doesn't have a decisionStrategy defined as below. All clusters will be set under decisionGroupIndex 0. 

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: ztp-placement
  namespace: ztp-acm-ns
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: common-profile
              operator: In
              values:
                - 'true'
status:
  numberOfSelectedClusters: 320
  decisionGroups:
  - decisionGroupIndex: 0
    placementDecisions:
    - ztp-placement-decision-0
    - ztp-placement-decision-1
    - ztp-placement-decision-2
    - ztp-placement-decision-3
    clusterCount: 320
```
Another example for a placement having a decisionStrategy define a single decision group as below. Two decision groups will be created; 1- first one have the clusters selected by prod-canary selection. 2- Second group have the rest of clusters selected by the placement predicates decision (common-profile = true). 

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: ztp-placement
  namespace: ztp-acm-ns
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: common-profile
              operator: In
              values:
                - 'true'
  DecisionStrategy:
    clustersPerDecisionGroup: 100%
    DecisionGroups:
    - groupName: prod-canary
      clusterSelector:
        matchExpressions:
          - key: prod-canary
            operator: Exist

status:
  numberOfSelectedClusters: 320
  decisionGroups:
  - decisionGroupIndex: 0
    placementDecisions:
    - ztp-placement-decision-0
    clusterCount: 20
  - decisionGroupIndex: 1
    placementDecisions:
    - ztp-placement-decision-1
    - ztp-placement-decision-2
    - ztp-placement-decision-3
    clusterCount: 300
```
Another example for a placement having decisionStrategy defined with clustersPerDecisionGroup=150 as below. With a selection out of 320 clusters there are 3 decisionGroup are created.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: ztp-placement
  namespace: ztp-acm-ns
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: common-profile
              operator: In
              values:
                - 'true'
  DecisionStrategy:
    clustersPerDecisionGroup: 150
status:
  numberOfSelectedClusters: 320
  decisionGroups:
  - decisionGroupIndex: 0
    placementDecisions:
    - ztp-placement-decision-0
    - ztp-placement-decision-1
    clusterCount: 150
  - decisionGroupIndex: 1
    placementDecisions:
    - ztp-placement-decision-2
    - ztp-placement-decision-3
    clusterCount: 150
  - decisionGroupIndex: 2
    placementDecisions:
    - ztp-placement-decision-4
    clusterCount: 20
```

The Placement Helper [library](https://github.com/open-cluster-management-io/api/blob/main/cluster/v1beta1/helpers.go) should provide functions to facilitate retrieve the decision groups.

#### Special case handling

* Clusters have been removed from decision group index 1 and decision group index 2 has been created. In order to not shift current selected clusters in the decision groups, placement controller will keep append clusters to decision group index 2 till it reach max number of clusters per decision group then append new cluster to decision group index 1 till it reach the max number of clusters per decision group.


### How Workload applier APIs will benefits from using placementStrategy

The OCM APIs such as ManifestWorkReplicaSet or Policy can define how-to roll out the workloads considering the Decision Groups defined in the placement.
For example; assuming the below RolloutStrategy struct is part of the manifestWorkReplicaSet APIs. It defines the workload rollout type (All, Progressive and ProgressivePerGroup), the decision groups to apply the workload first, timeout for waiting the workload to reach successful state and maxConcurrency of the clusters to apply the workload (if placement->DecisionStrategy->clustersPerDecisionGroup not considered).

```go
type RolloutStrategy struct {
	// Rollout types are All, Progressive and ProgressivePerGroup
	// 1) All: means apply the workload to all clusters in the decision groups at once.
	// 2) Progressive: means apply the workload to the selected clusters progressively per cluster. The max concurrency of clusters to apply the workload will be equal to the clustersPerDecisionGroup defined in the placement->DecisionStrategy APIs.
	// 3) ProgressivePerGroup: means apply the workload to decisionGroup clusters progressively per group. The workload will not be applied to the next decisionGroup unless the previous group reach the successful state or timeout.
	// +kubebuilder:validation:Enum=All;Progressive;ProgressivePerGroup
	// +kubebuilder:default:=All
	// +optional
	type string `json:"type"`

	// List of the decision groups names or indexes to apply the workload first and fail (not proceed to apply workload to other decision groups) if workload did not reach successful state or time out.
	// +optional
	MandatoryDecisionGroups []intOrString `json:"mandatoryDecisionGroups"`

	// Timeout defined in minutes for how long workload applier controller will wait till reach successful state. Only considered for Rollout Type Progressive and ProgressivePerGroup. Default is None meaning the workload apply will not proceed if did not reach the successful state. 
	// +kubebuilder:default:=None
	// +optional
	Timeout string `json:"timeout"`
	
	// MaxConcurrency is the max number of clusters to deploy workload concurrently. The default value for MaxConcurrency is determined from the clustersPerDecisionGroup defined in the placement->DecisionStrategy.
	// +optional
	MaxConcurrency intOrString `json:"maxConcurrency,omitempty"`
}
```
The Rollout Strategy types;
1) All: means apply the workload to all clusters in the decision groups at once.
2) Progressive: means apply the workload to the selected clusters progressively per cluster. The max concurrency of clusters to apply the workload will be equal to the clustersPerDecisionGroup defined in the placement->DecisionStrategy APIs.
3) ProgressivePerGroup: means apply the workload to decisionGroup clusters progressively per group. The workload will not be applied to the next decisionGroup unless the previous group reach the successful state or timeout.

The mandatoryDecisionGroups is a list of decision groups to apply the workload first. If mandatoryDecisionGroups not defined the decision group index is considered to apply the workload in groups by order.

The Timeout defined in minutes for how long workload applier controller will wait till workload reach successful state in the spoke cluster.

Let's consider the below ManifestWorkReplicaSet example has a placementRef to the above ztp-placement example AND the RolloutStrategy type is progressive with list of the decision groups names.

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestWorkReplicaSet
metadata:
  name: ztp-mwrset
  namespace: ztp-acm-ns
spec:
  placementRefs:
    - name: ztp-placement
      rolloutStrategy:
        rolloutType: Progressive
        timeout: 50min
        mandatoryDecisionGroups:
        - prod-canary-west
        - prod-canary-east
  manifestWorkTemplate:
    ...
```

The decision groups associated with ztp-placement as below

```yaml
  decisionGroups:
  - decisionGroupIndex: 0
    decisionGroupName: prod-canary-west
    placementDecisions:
    - ztp-placement-decision-0
    clusterCount: 10
  - decisionGroupIndex: 1
    decisionGroupName: prod-canary-east
    placementDecisions:
    - ztp-placement-decision-1
    clusterCount: 10
  - decisionGroupIndex: 2
    placementDecisions:
    - ztp-placement-decision-2
    - ztp-placement-decision-3
    clusterCount: 150
  - decisionGroupIndex: 3
    placementDecisions:
    - ztp-placement-decision-4
    - ztp-placement-decision-5
    clusterCount: 140 
```
The ManifestWorkReplicaSet controller apply the workload to the decision groups prod-canary-west & prod-canary-east progressively first and wait till all the clusters in those decision groups have the manifestWork in available state.
If one of the managedCluster in the decision groups prod-canary-west & prod-canary-east has the manifestWork not in available state the ManifestWorkReplicaSet controller will not proceed to apply the manifestWork to the other decision groups.
After all clusters in decision groups prod-canary-west & prod-canary-east have the manifestWork in available state, the ManifestWorkReplicaSet controller will apply the manifestWorkTemplate on the clusters in decision groups 2 and 3 progressively.

#### Special case handling for applier controller.

* A cluster has been added to decision group index 0 while the applier controller applying workloads in decision group index 2. The applier controller should apply the workload on the added cluster in the decision group 0. The applier controller should decide either continue apply the workload progressively or wait till the added cluster to the decision group 0 has the workload in its desired successful state.

* While applying the workloads on decision group index 2 the placement has been changed and all cluster groups changed. The applier controller should act accordingly and apply the workload to the added or removed clusters in decision group index 0 then decision group index 1.

### Risks and Mitigation

* Security is based on the placement resources which bind to ManagedClusterSets and the namespace where the placement CR is created.

### Test Plan

- Unit tests will cover the new changes for the APIs.
- Integration tests will cover all user stories.
- And e2e tests will cover the following cases.
  - Create/updated/delete placement with decisionStrategy.
  - Placement Decision changes while more managed clusters selected by the placement.
  - Placement Decision changes when update the decisionStrategy.

### Graduation Criteria
**Note:** *Section not required until targeted at a release.*

#### Tech Preview -> GA
**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

### Upgrade / Downgrade Strategy

Placement->decisionStrategy API will be supported on the target release of OCM (open cluster management) and it should not affect the OCM current APIs functionalities or back compatibility.

## Alternatives

1. With multiple placements, we can not define the number of clusters or percentage in each group, and dynamically generate each groups.
1. End user must label each managedCluster or set the managedClusterSet then create many placements accordingly to apply the workloads using each placement manually. 
1. Each OCM APIs such as ManifestWorkReplicaSet, ClusterManagementAddon and Policy define its own placement and roll out strategy to determine the progressive roll out.
1. Current alternative does not satisfy the requirements as explained above.
