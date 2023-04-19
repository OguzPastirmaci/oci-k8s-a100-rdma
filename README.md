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


### Pushing an Image to Oracle Cloud Infrastructure Registry
https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/registry/index.html

### Pulling Images from Registry during Deployment
[https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/oke-and-registry/index.html](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengpullingimagesfromocir.htm)

### Running NCCL test 

```yaml
apiVersion: kubeflow.org/v2beta1
kind: MPIJob
metadata:
  name: nccl-test-a100
spec:
  slotsPerWorker: 8
  runPolicy:
    cleanPodPolicy: Running
  mpiReplicaSpecs:
    Launcher:
      replicas: 1
      template:
          spec:
            initContainers:
            - name: node-ordering-by-rack
              image: oguzpastirmaci/node-ordering-by-rack:init-mpijob-v1
              volumeMounts:
              - name: node-ordering-by-rack
                mountPath: "/node-ordering-by-rack"
              - name: mpi-job-config
                mountPath: /etc/mpi
              - name: ssh-auth
                mountPath: /root/.ssh
            volumes:
            - name: node-ordering-by-rack
              emptyDir: {}    
            containers:
            - image: oguzpastirmaci/nccl-tests:cuda
              name: nccl-tests
              volumeMounts:
              - name: node-ordering-by-rack
                mountPath: "/node-ordering-by-rack"
              env:
              - name: OMPI_ALLOW_RUN_AS_ROOT
                value: "1"
              - name: OMPI_ALLOW_RUN_AS_ROOT_CONFIRM
                value: "1"           
              #command: ['sleep', '86400']
              command: ["/bin/bash", "-c"]
              args: ["mpirun \
                    --bind-to numa \
                    --hostfile /node-ordering-by-rack/ordered_hostfile \
                    --mca pml ob1 --mca btl tcp,self --mca btl_tcp_if_include eth0  --mca coll ^hcoll \
                    -x HCOLL_ENABLE_MCAST_ALL=0 \
                    -x coll_hcoll_enable=0 \
                    -x NCCL_IB_HCA=mlx5 \
                    -x NCCL_IB_GID_INDEX=3 \
                    -x NCCL_IB_QPS_PER_CONNECTION=4 \
                    -x NCCL_IB_TC=41 \
                    -x NCCL_IB_SL=0 \
                    /opt/nccl_tests/build/all_reduce_perf -b1G -e10G -i$((1024*1024*1024*9)) -g 1
                    "]

              resources:
                requests:
                  cpu: 2
                  memory: 128Mi
    Worker:
      replicas: 2
      template:
        metadata:
          annotations:
            k8s.v1.cni.cncf.io/networks: sriov-net, sriov-net, sriov-net, sriov-net, sriov-net, sriov-net, sriov-net, sriov-net, sriov-net, sriov-net, sriov-net, sriov-net, sriov-net, sriov-net, sriov-net, sriov-net
        spec:
          containers:
          - image: oguzpastirmaci/nccl-tests:cuda
            #securityContext:
              #capabilities:
                #add: [ "IPC_LOCK" ]
            name: nccl
            resources:
              requests:
                cpu: 100
                memory: 1000Gi
                nvidia.com/gpu: 8
                nvidia.com/rdma_sriov: 16
              limits:
                nvidia.com/gpu: 8
                nvidia.com/rdma_sriov: 16
            volumeMounts:
              - mountPath: /dev/shm
                name: dshm
          volumes:
            - emptyDir:
                medium: Memory
              name: dshm                
```
