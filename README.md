# oci-k8s-a100-rdma

GPU worker nodes:

OS: Ubuntu 20.04
GPU driver version: 515.86.01
CUDA version: 11.7
OFED driver version: 


### Provisioning PVCs on the OCI Block Volume Service
https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingpersistentvolumeclaim_topic-Provisioning_PVCs_on_BV.htm#Provisioning_Persistent_Volume_Claims_on_the_Block_Volume_Service

### Defining Kubernetes Services of Type LoadBalancer
https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingloadbalancer.htm

### Pushing an Image to Oracle Cloud Infrastructure Registry
https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/registry/index.html

### Requesting SR-IOV virtual functions

Each GPU node has 16 virtual functions (VF). When requesting VFs, use the `k8s.v1.cni.cncf.io/networks: sriov-net` annotation, and request `nvidia.com/rdma_sriov` in your container spec.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rdma-test-pod-1
  annotations:
    k8s.v1.cni.cncf.io/networks: sriov-net
spec:
  restartPolicy: OnFailure
  containers:
  - image: oguzpastirmaci/mofed-perftest:5.4-3.6.8.1-ubuntu20.04-amd64
    name: mofed-test-ctr
    securityContext:
      capabilities:
        add: [ "IPC_LOCK" ]
    resources: 
      requests:
        nvidia.com/rdma_sriov: 1
      limits:
        nvidia.com/rdma_sriov: 1
    command:
    - sh
    - -c
    - |
      ls -l /dev/infiniband /sys/class/net
      sleep 1000000
 ```
