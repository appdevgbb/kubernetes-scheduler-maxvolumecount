# Understanding node autoscaler and maxAttachLimit on Kubernetes

This is an exploratory repo with information on how node autoscaler can be triggered based on the number of volumes reaching a limit on an AKS node.

## What we know

1. Nodes will have a limit on how many volumes they can attach before hitting a limit. We are assuming here that we have a 1:1:1 ratio between, PVC:PV:Disk. For example, a `Standard_DS2_v2` will support up to [8](https://learn.microsoft.com/en-us/azure/virtual-machines/dv2-dsv2-series) volumes/disks. 
2. When scheduling a pod to a node that that is at it's maximum number of volumes (disks), the `default-scheduler` will not be able to find a place to run that pod and it will raise a `FailedScheduling` warning. The typical warning message looks like this:  0/1 nodes are available: 1 node(s) exceed max volume count. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.
3. When condition (3) happens and the cluster has autoscaling enabled, the `cluster-autoscaler` will add a new node as described [here](https://github.com/kubernetes/kubernetes/blob/6330b2722559c49e76daf347240ba569023257e4/pkg/scheduler/framework/plugins/nodevolumelimits/csi.go#L157).

![image](https://github.com/appdevgbb/kubernetes-scheduler-maxvolumecount/assets/3240777/a9b1ef39-0ce3-426c-8893-8a33283fc705)


## Recreating this condition
1. Create a deployment file

```bash
apiVersion: v1  
kind: PersistentVolumeClaim  
metadata:  
  name: azure-managed-disk-TOKEN
spec:  
  accessModes:  
  - ReadWriteOnce  
  storageClassName: managed-premium  
  resources:  
    requests:  
      storage: 5Gi  
  
---  
  
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: azure-disk-deployment-TOKEN  
spec:  
  replicas: 1  
  selector:  
    matchLabels:  
      app: azure-disk-app  
  template:  
    metadata:  
      labels:  
        app: azure-disk-app  
    spec:  
      containers:  
      - name: azure-disk-app-container-TOKEN
        image: nginx  
        ports:  
        - containerPort: 80  
        volumeMounts:  
        - mountPath: "/mnt/azure"  
          name: volume-TOKEN
      volumes:  
      - name: volume-TOKEN
        persistentVolumeClaim:  
          claimName: azure-managed-disk-TOKEN
```

2. Run the following one-liner to create 8 disks.

```bash
for i in `seq 1  8`; do sed "s/TOKEN/$i/g" deployment.yaml | kubectl apply -f - ; done
```

3. Once kube-scheduler can't find a placement for a pod, it will issue a `FailedScheduling` error message:
```yaml
- apiVersion: v1  
  count: 1    
  eventTime: null
  firstTimestamp: "2023-10-25T23:11:02Z"
  involvedObject:
    apiVersion: v1
    kind: Pod
    name: azure-disk-deployment-7-6df87c6d74-jw7dw               
    namespace: default
    resourceVersion: "4108869"
    uid: 1f016116-89a2-4930-b574-123b036787a1
  kind: Event
  lastTimestamp: "2023-10-25T23:11:02Z"
  message: '0/1 nodes are available: 1 node(s) exceed max volume count. preemption:
    0/1 nodes are available: 1 No preemption victims found for incoming pod..'
  metadata:
    creationTimestamp: "2023-10-25T23:11:02Z"
    name: azure-disk-deployment-7-6df87c6d74-jw7dw.17917c890d4dcf60
    namespace: default 
    resourceVersion: "4108870"
    uid: 681a7e25-2177-4f82-aef9-f4fe987a344c
  reason: FailedScheduling
  reportingComponent: ""
  reportingInstance: ""
  source:
    component: default-scheduler
  type: Warning
```

## Timeline of events

| Time | Severity | Type | Object | Message |  
| --- | --- | --- | --- | --- |  
| 13m | Normal | WaitForPodScheduled | persistentvolumeclaim/azure-managed-disk-7 | waiting for pod azure-disk-deployment-7-6df87c6d74-jw7dw to be scheduled |  
| 14m | Normal | SuccessfulAttachVolume | pod/azure-disk-deployment-6-84d57ccd54-lhx2j | AttachVolume.Attach succeeded for volume "pvc-2d87d0f1-240b-4971-b0e3-7a766e0c7f2e" |  
| 14m | Normal | SuccessfulAttachVolume | pod/azure-disk-deployment-4-85b466694-4hcbw | AttachVolume.Attach succeeded for volume "pvc-c522bb55-2af4-4979-ac2f-acad2aa0fcc5" |  
| ... | ... | ... | ... | ... |  
| 14m | Normal | SuccessfulAttachVolume | pod/azure-disk-deployment-3-5f668c9f94-ff6bw | AttachVolume.Attach succeeded for volume "pvc-904d6d49-6f40-4257-bd85-b53dd2e2b7a3" |  
| 14m | Normal | TriggeredScaleUp | pod/azure-disk-deployment-7-6df87c6d74-jw7dw | pod triggered scale-up: [{aks-nodepool1-50557415-vmss 1->2 (max: 3)}] |  
| 14m | Normal | SuccessfulAttachVolume | pod/azure-disk-deployment-2-866f7b4974-dpszd | AttachVolume.Attach succeeded for volume "pvc-535fa3b9-9743-45a9-a9c9-abae3f2e933b" |  
| 14m | Normal | SuccessfulAttachVolume | pod/azure-disk-deployment-5-7dff99f658-6s449 | AttachVolume.Attach succeeded for volume "pvc-c7ec8210-2f7b-458c-8e0e-fc8f39731a3f" |  
| 14m | Normal | Pulling | pod/azure-disk-deployment-5-7dff99f658-6s449 | Pulling image "nginx" |  
| 14m | Normal | Pulling | pod/azure-disk-deployment-2-866f7b4974-dpszd | Pulling image "nginx" |  
| ... | ... | ... | ... | ... |  
| 14m | Normal | Starting | node/aks-nodepool1-50557415-vmss000004 | Starting kubelet. |  
| 14m | Warning | InvalidDiskCapacity | node/aks-nodepool1-50557415-vmss000004 | invalid capacity 0 on image filesystem |  
| 14m | Normal | NodeReady | node/aks-nodepool1-50557415-vmss000004 | Node aks-nodepool1-50557415-vmss000004 status is now: NodeReady |  
| 14m | Warning | FailedScheduling | pod/azure-disk-deployment-7-6df87c6d74-jw7dw | 0/2 nodes are available: 1 node(s) exceed max volume count, 1 node(s) had untolerated taint {node.cloudprovider.kubernetes.io/uninitialized: true}. preemption: 0/2 nodes are available: 1 No preemption victims found for incoming pod, 1 Preemption is not helpful for scheduling.. |  
| 14m | Normal | NodeHasSufficientMemory | node/aks-nodepool1-50557415-vmss000004 | Node aks-nodepool1-50557415-vmss000004 status is now: NodeHasSufficientMemory |  
| 14m | Normal | NodeHasNoDiskPressure | node/aks-nodepool1-50557415-vmss000004 | Node aks-nodepool1-50557415-vmss000004 status is now: NodeHasNoDiskPressure |  
| 14m | Normal | NodeHasSufficientPID | node/aks-nodepool1-50557415-vmss000004 | Node aks-nodepool1-50557415-vmss000004 status is now: NodeHasSufficientPID |  
| 14m | Normal | NodeAllocatableEnforced | node/aks-nodepool1-50557415-vmss000004 | Updated Node Allocatable limit across pods |  
| 14m | Normal | RegisteredNode | node/aks-nodepool1-50557415-vmss000004 | Node aks-nodepool1-50557415-vmss000004 event: Registered Node aks-nodepool1-50557415-vmss000004 in Controller |  
| 14m | Warning | ContainerdStart | node/aks-nodepool1-50557415-vmss000004 | Starting containerd container runtime... |  
| 13m | Normal | Starting | node/aks-nodepool1-50557415-vmss000004 |  |  
| 13m | Normal | Provisioning | persistentvolumeclaim/azure-managed-disk-7 | External provisioner is provisioning volume for claim "default/azure-managed-disk-7" |  
| 13m | Normal | ExternalProvisioning | persistentvolumeclaim/azure-managed-disk-7 | waiting for a volume to be created, either by external provisioner "disk.csi.azure.com" or manually created


## References:

1. [https://learn.microsoft.com/en-us/answers/questions/669556/max-data-disks-attached-to-a-worker-node-in-azure](https://learn.microsoft.com/en-us/answers/questions/669556/max-data-disks-attached-to-a-worker-node-in-azure)
