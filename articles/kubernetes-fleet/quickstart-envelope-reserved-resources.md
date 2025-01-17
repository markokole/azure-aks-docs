---
title: "Propagate reserved resources from an Azure Kubernetes Fleet Manager hub cluster to member clusters"
description: Learn to propagate resources using ClusterResourcePlacement API without unintended side effects on the hub cluster.
ms.topic: how-to
ms.date: 11/22/2024
author: jamyct
ms.author: jamisontsai
ms.service: azure-kubernetes-fleet-manager
ms.custom:
- devx-track-azurecli
- ignite-2023
- build-2024
---

# Propagate reserved resources from an Azure Kubernetes Fleet Manager hub cluster to member clusters

This article provides an overview of how to use envelope objects to propagate reserved Kubernetes resource types from an Azure Kubernetes Fleet Manager hub cluster to member clusters.

## Envelope Object with ConfigMap

Currently, we support using a ConfigMap as an envelope object by leveraging a fleet-reserved annotation.

To designate a ConfigMap as an envelope object, ensure that it contains the following annotation:

```
metadata:
  annotations:
    kubernetes-fleet.io/envelope-configmap: "true"
```

Example ConfigMap Envelope Object

```
apiVersion: v1
kind: ConfigMap
metadata:
    name: envelope-configmap
    namespace: app
    annotations:
        kubernetes-fleet.io/envelope-configmap: "true"
data:
    resourceQuota.yaml: |
        apiVersion: v1
        kind: ResourceQuota
        metadata:
            name: mem-cpu-demo
            namespace: app
        spec:
            hard:
                requests.cpu: "1"
                requests.memory: 1Gi
                limits.cpu: "2"
                limits.memory: 2Gi
    webhook.yaml: |
        apiVersion: admissionregistration.k8s.io/v1
        kind: MutatingWebhookConfiguration
        metadata:
            creationTimestamp: null
            labels:
                azure-workload-identity.io/system: "true"
            name: azure-wi-webhook-mutating-webhook-configuration
        webhooks:
        - admissionReviewVersions:
          - v1
          - v1beta1
          clientConfig:
              service:
                  name: azure-wi-webhook-webhook-service
                  namespace: app
                  path: /mutate-v1-pod
          failurePolicy: Fail
          matchPolicy: Equivalent
          name: mutation.azure-workload-identity.io
          rules:
          - apiGroups:
              - ""
              apiVersions:
              - v1
              operations:
              - CREATE
              - UPDATE
              resources:
              - pods
          sideEffects: None
```

## Propagating an Envelope ConfigMap to member clusters

We now apply the example envelope object above on our hub cluster. Then we use a ClusterResourcePlacement object to propagate the resource from hub to a member cluster named kind-cluster-1.

Here's a sample ClusterResourcePlacement (CRP) specification.

```
spec:
    policy:
        clusterNames:
        - kind-cluster-1
        placementType: PickFixed
    resourceSelectors:
    - group: ""
        kind: Namespace
        name: app
        version: v1
    revisionHistoryLimit: 10
    strategy:
        type: RollingUpdate
```

## Retrieve the status of Envelope ConfigMap placement

Here's a sample status that shows the successful placement of an Enveloped Object.

```
status:
conditions:
- lastTransitionTime: "2023-11-30T19:54:13Z"
  message: found all the clusters needed as specified by the scheduling policy
  observedGeneration: 2
  reason: SchedulingPolicyFulfilled
  status: "True"
  type: ClusterResourcePlacementScheduled
- lastTransitionTime: "2023-11-30T19:54:18Z"
  message: All 1 cluster(s) are synchronized to the latest resources on the hub
  cluster
  observedGeneration: 2
  reason: SynchronizeSucceeded
  status: "True"
  type: ClusterResourcePlacementSynchronized
- lastTransitionTime: "2023-11-30T19:54:18Z"
  message: Successfully applied resources to 1 member clusters
  observedGeneration: 2
  reason: ApplySucceeded
  status: "True"
  type: ClusterResourcePlacementApplied
  placementStatuses:
- clusterName: kind-cluster-1
  conditions:
    - lastTransitionTime: "2023-11-30T19:54:13Z"
      message: 'Successfully scheduled resources for placement in kind-cluster-1:
      picked by scheduling policy'
      observedGeneration: 2
      reason: ScheduleSucceeded
      status: "True"
      type: ResourceScheduled
    - lastTransitionTime: "2023-11-30T19:54:18Z"
      message: Successfully Synchronized work(s) for placement
      observedGeneration: 2
      reason: WorkSynchronizeSucceeded
      status: "True"
      type: WorkSynchronized
    - lastTransitionTime: "2023-11-30T19:54:18Z"
      message: Successfully applied resources
      observedGeneration: 2
      reason: ApplySucceeded
      status: "True"
      type: ResourceApplied
      selectedResources:
- kind: Namespace
  name: app
  version: v1
- kind: ConfigMap
  name: envelope-configmap
  namespace: app
  version: v1
```

> [!NOTE]
> In the selectedResources section, we specifically display the propagated envelope object. Please note that we do not individually list all the resources contained within the envelope object in the status.

Upon inspection of the selectedResources, it indicates that the namespace app and the configmap envelope-configmap have been successfully propagated. Users can further verify the successful propagation of resources mentioned within the envelope-configmap object by ensuring that the failedPlacements section in the placementStatus for kind-cluster-1 doesn't appear in the status.

Here's a sample where the placement has failed. In this example below, within the placementStatus section for kind-cluster-1, the failedPlacements section provides details on resource that failed to apply along with information about the envelope object which contained the resource.

```
status:
conditions:
- lastTransitionTime: "2023-12-06T00:09:53Z"
  message: found all the clusters needed as specified by the scheduling policy
  observedGeneration: 2
  reason: SchedulingPolicyFulfilled
  status: "True"
  type: ClusterResourcePlacementScheduled
- lastTransitionTime: "2023-12-06T00:09:58Z"
  message: All 1 cluster(s) are synchronized to the latest resources on the hub
  cluster
  observedGeneration: 2
  reason: SynchronizeSucceeded
  status: "True"
  type: ClusterResourcePlacementSynchronized
- lastTransitionTime: "2023-12-06T00:09:58Z"
  message: Failed to apply manifests to 1 clusters, please check the `failedPlacements`
  status
  observedGeneration: 2
  reason: ApplyFailed
  status: "False"
  type: ClusterResourcePlacementApplied
  placementStatuses:
- clusterName: kind-cluster-1
  conditions:
    - lastTransitionTime: "2023-12-06T00:09:53Z"
      message: 'Successfully scheduled resources for placement in kind-cluster-1:
      picked by scheduling policy'
      observedGeneration: 2
      reason: ScheduleSucceeded
      status: "True"
      type: ResourceScheduled
    - lastTransitionTime: "2023-12-06T00:09:58Z"
      message: Successfully Synchronized work(s) for placement
      observedGeneration: 2
      reason: WorkSynchronizeSucceeded
      status: "True"
      type: WorkSynchronized
    - lastTransitionTime: "2023-12-06T00:09:58Z"
      message: Failed to apply manifests, please check the `failedPlacements` status
      observedGeneration: 2
      reason: ApplyFailed
      status: "False"
      type: ResourceApplied
      failedPlacements:
    - condition:
      lastTransitionTime: "2023-12-06T00:09:53Z"
      message: 'Failed to apply manifest: namespaces "app" not found'
      reason: AppliedManifestFailedReason
      status: "False"
      type: Applied
      envelope:
      name: envelop-configmap
      namespace: test-ns
      type: ConfigMap
      kind: ResourceQuota
      name: mem-cpu-demo
      namespace: app
      version: v1
      selectedResources:
- kind: Namespace
  name: test-ns
  version: v1
- kind: ConfigMap
  name: envelop-configmap
  namespace: test-ns
  version: v1
```
