# oci-k8s-a100-rdma

#### GPU worker nodes:

- OS: Ubuntu 20.04
- GPU driver version: 515.86.01
- CUDA version: 11.7
- OFED driver version: 


### Provisioning PVCs on the OCI Block Volume Service
https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingpersistentvolumeclaim_topic-Provisioning_PVCs_on_BV.htm#Provisioning_Persistent_Volume_Claims_on_the_Block_Volume_Service

### Defining Kubernetes Services of Type LoadBalancer
When you have `type: LoadBalancer` in your service spec, a load balancer with an external IP will be created automatically.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-svc
  labels:
    app: nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: nginx
```

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

### Requesting GPUs
Each GPU node has 8 x A100 40 GB GPUs. When requesting GPUs, use `nvidia.com/gpu` in your pod spec.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nvidia-version-check
spec:
  restartPolicy: OnFailure
  containers:
  - name: nvidia-version-check
    image: nvidia/cuda:11.7.1-base-ubuntu20.04
    command: ["nvidia-smi"]
    resources:
      limits:
         nvidia.com/gpu: "1"
```

### Pushing an Image to Oracle Cloud Infrastructure Registry
https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/registry/index.html

### Pulling Images from Registry during Deployment
[https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/oke-and-registry/index.html](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengpullingimagesfromocir.htm)

### Running ib_write_bw test between pods using VFs

Deploy the following pods:

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
---
apiVersion: v1
kind: Pod
metadata:
  name: rdma-test-pod-2
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
        nvidia.com/rdma_sriov: "1"
      limits:
        nvidia.com/rdma_sriov: "1"
    command:
    - sh
    - -c
    - |
      ls -l /dev/infiniband /sys/class/net
      sleep 1000000
```


Wait until both pods are in Running state.

```
kubectl get pods -o wide

NAME              READY   STATUS    RESTARTS   AGE   IP           NODE    NOMINATED NODE   READINESS GATES
rdma-test-pod-1   1/1     Running   0          49m   10.0.0.125   gpu02   <none>           <none>
rdma-test-pod-2   1/1     Running   0          49m   10.0.0.70    gpu01   <none>           <none>
```

After the pods are running, open two terminals and exec into the pods.

```
kubectl exec -it rdma-test-pod-1 -- bash

kubectl exec -it rdma-test-pod-2 -- bash
```

In `rdma-test-pod-1`, first get the IP address of the net1 interface (net1 is the VF interface) by running `ip -f inet addr show net1`. You will see a 192.168.0.x IP.

```
ip -f inet addr show net1

36: net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 4220 qdisc mq state UP group default qlen 20000
    altname enp195s0f0v0
    inet 192.168.0.1/16 brd 192.168.255.255 scope global net1
       valid_lft forever preferred_lft forever
```

Then run `ibdev2netdev` to get the interface name:

```
ibdev2netdev

mlx5_19 port 1 ==> net1 (Up)
```

Then use the interface name in the below command (e.g. mlx5_19):

```
ib_write_bw -d mlx5_19 -a -F
```

In `rdma-test-pod-2` run `ibdev2netdev` to get the interface name, and use the interface name of `rdma-test-pod-2` and the IP of net1 of `rdma-test-pod-1`  in the following command: 

```
ib_write_bw -F -d mlx5_33 <IP of net1 in rdma-test-pod-1> -D 10 --cpu_util --report_gbits
```

You should see a result similar to below:

```
************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_3
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 CQ Moderation   : 100
 Mtu             : 4096[B]
 Link type       : Ethernet
 GID index       : 2
 Max inline data : 0[B]
 rdma_cm QPs	 : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x10107 PSN 0xb5554 RKey 0x04ad6e VAddr 0x007f8389beb000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:07:201
 remote address: LID 0000 QPN 0x10107 PSN 0x92e7fa RKey 0x047c3b VAddr 0x007fd6f366c000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:07:189
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 65536      1101500          0.00               96.25  		   0.183585
---------------------------------------------------------------------------------------
```

### Parameters to use when using NCCL as the backend
When using NCCL as the backend in your jobs, please make sure you use the following parameters for optimal performance:

```
NCCL_IB_HCA=mlx5
NCCL_IB_GID_INDEX=3
NCCL_IB_QPS_PER_CONNECTION=4
NCCL_IB_TC=41
NCCL_IB_SL=0
```
