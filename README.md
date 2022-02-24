# Node Maintenance Operator

The node-maintenance-operator is an operator generated from the [operator-sdk](https://github.com/operator-framework/operator-sdk).
The purpose of this operator is to watch for new or deleted custom resources called `NodeMaintenance` which indicate that a node in the cluster should either:
  - `NodeMaintenance` CR created: move node into maintenance, cordon the node - set it as unschedulable, and evict the pods (which can be evicted) from that node.
  - `NodeMaintenance` CR deleted: remove pod from maintenance and uncordon the node - set it as schedulable.

> *Note*:  The current behavior of the operator is to mimic `kubectl drain <node name>`.

## Build and run the operator

There are two ways to run the operator:

- Deploy the latest version, which was built from master branch, to a running OpenShift/Kubernetes cluster.
- Build and deploy from sources to a running or to be created OpenShift/Kubernetes cluster.

### Deploy the latest version

After every PR merge to master images were build and pushed to `quay.io`.
For deployment of NMO using these images you need:

- a running OpenShift cluster, or a Kubernetes cluster with OLM installed.
- `operator-sdk` binary installed, see https://sdk.operatorframework.io/docs/installation/.
- a valid `$KUBECONFIG` configured to access your cluster.

Then run `operator-sdk run bundle quay.io/medik8s/node-maintenance-operator-bundle:latest`

### Build and deploy from sources
Follow the instructions [here](https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/#3-deploy-your-operator-with-olm) for deploying the operator with OLM.
> *Note*: Webhook cannot run using `make deploy`, because the volume mount of the webserver certificate is not found.

## Setting Node Maintenance

### Set Maintenance on - Create a NodeMaintenance CR

To set maintenance on a node a `NodeMaintenance` CustomResource should be created.
The `NodeMaintenance` CR spec contains:
- nodeName: The name of the node which will be put into maintenance mode.
- reason: The reason why the node will be under maintenance.

Create the example `NodeMaintenance` CR found at `config/samples/nodemaintenance_v1beta1_nodemaintenance.yaml`:

```sh
$ cat config/samples/nodemaintenance_v1beta1_nodemaintenance.yaml
apiVersion: nodemaintenance.medik8s.io/v1beta1
kind: NodeMaintenance
metadata:
  name: nodemaintenance-sample
spec:
  nodeName: node02
  reason: "Test node maintenance"

$ kubectl apply -f config/samples/nodemaintenance_v1beta1_nodemaintenance.yaml

$ kubectl logs <nmo-pod-name>
022-02-23T07:33:58.924Z INFO controller-runtime.manager.controller.nodemaintenance Reconciling NodeMaintenance {"reconciler group": "nodemaintenance.medik8s.io", "reconciler kind": "NodeMaintenance", "name": "nodemaintenance-sample", "namespace": ""}
2022-02-23T07:33:59.266Z INFO controller-runtime.manager.controller.nodemaintenance Applying maintenance mode {"reconciler group": "nodemaintenance.medik8s.io", "reconciler kind": "NodeMaintenance", "name": "nodemaintenance-sample", "namespace": "", "node": "node02", "reason": "Test node maintenance"}
time="2022-02-24T11:58:20Z" level=info msg="Maintenance taints will be added to node node02"
time="2022-02-24T11:58:20Z" level=info msg="Applying medik8s.io/drain taint add on Node: node02"
time="2022-02-24T11:58:20Z" level=info msg="Patching taints on Node: node02"
2022-02-23T07:33:59.336Z INFO controller-runtime.manager.controller.nodemaintenance Evict all Pods from Node {"reconciler group": "nodemaintenance.medik8s.io", "reconciler kind": "NodeMaintenance", "name": "nodemaintenance-sample", "namespace": "", "nodeName": "node02"}
E0223 07:33:59.498801 1 nodemaintenance_controller.go:449] WARNING: ignoring DaemonSet-managed Pods: openshift-cluster-node-tuning-operator/tuned-jrprj, openshift-dns/dns-default-kf6jj, openshift-dns/node-resolver-72jzb, openshift-image-registry/node-ca-czgc6, openshift-ingress-canary/ingress-canary-44tgv, openshift-machine-config-operator/machine-config-daemon-csv6c, openshift-monitoring/node-exporter-rzwhz, openshift-multus/multus-additional-cni-plugins-829bh, openshift-multus/multus-qwfc9, openshift-multus/network-metrics-daemon-pxt6n, openshift-network-diagnostics/network-check-target-qqcbr, openshift-sdn/sdn-s5cqx; deleting Pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: openshift-marketplace/nmo-downstream-8-8nms7
I0223 07:33:59.500418 1 nodemaintenance_controller.go:449] evicting pod openshift-network-diagnostics/network-check-source-865d4b5578-n2cxg
I0223 07:33:59.500790 1 nodemaintenance_controller.go:449] evicting pod openshift-ingress/router-default-7548cf6fb5-rgxrq
I0223 07:33:59.500944 1 nodemaintenance_controller.go:449] evicting pod openshift-marketplace/12a4cfa0c2be01867daf1d9b7ad7c0ae7a988fd957a2ad6df0d72ff6875lhcx
I0223 07:33:59.501061 1 nodemaintenance_controller.go:449] evicting pod openshift-marketplace/nmo-downstream-8-8nms7
...
```

### Set Maintenance off - Delete the NodeMaintenance CR

To remove maintenance from a node, delete the corresponding `NodeMaintenance` (or `nm` which is a shortName) CR:

```sh
$ kubectl delete nm nodemaintenance-sample
nodemaintenance.nodemaintenance.medik8s.io "nodemaintenance-sample" deleted
$ kubectl logs <nmo-pod-name>
2022-02-24T14:27:35.332Z INFO controller-runtime.manager.controller.nodemaintenance Reconciling NodeMaintenance {"reconciler group": "nodemaintenance.medik8s.io", "reconciler kind": "NodeMaintenance", "name": "nodemaintenance-sample", "namespace": ""}
time="2022-02-24T14:27:35Z" level=info msg="Maintenance taints will be removed from node node02"
time="2022-02-24T14:27:35Z" level=info msg="Applying medik8s.io/drain taint remove on Node: node02"
...
```

## NodeMaintenance Status

The `NodeMaintenance` CR can contain the following status fields:

```yaml
$ kubectl get nm nodemaintenance-sample -o yaml
apiVersion: nodemaintenance.medik8s.io/v1beta1
kind: NodeMaintenance
metadata:
  creationTimestamp: "2022-02-24T14:37:25Z"
  finalizers:
  - foregroundDeleteNodeMaintenance
  generation: 1
  name: nodemaintenance-sample
  resourceVersion: "1267741"
  uid: 83cece87-f05c-41e8-bc22-5e6e0114f4b7
spec:
  nodeName: node02
  reason: Test node maintenance
status:
  evictionPods: 5
  pendingPods:
  - router-default-7548cf6fb5-6c6ws
  - alertmanager-main-1
  - prometheus-adapter-7b5bf59787-ccf5w
  - prometheus-k8s-1
  - thanos-querier-6dffd47d65-h4d5c
  phase: Running
  totalpods: 19
```

`phase` is the representation of the maintenance progress and can hold a string value of: Running|Succeeded.
The phase is updated for each processing attempt on the CR.

`lastError` represents the latest error if any for the latest reconciliation.

`pendingPods` PendingPods is a list of pending pods for eviction.

`totalPods` is the total number of all pods on the node from the start.

`evictionPods` is the total number of pods up for eviction from the start.

## Tests

### Run code checks and unit tests

`make check`

### Run e2e tests

1. Deploy the operator as explained above
2. run `make cluster-functest`

## Releases

### Creating a new release

For new minor releases:

  - create and push the `release-0.y` branch.
  - update OpenshiftCI with new branches!

For every major / minor / patch release:

  - create and push the `vx.y.z` tag.
  - this should trigger CI to build and push new images
    - if it fails, the manual fallback is `VERSION=x.y.z make container-build container-push`
  - make the git tag a release in the github UI.
