# Getting started with EFA on EKS




## EFA Basics

Please read https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html for basic knowledge of EFA

## Kubernetes and EKS Basics

This document is assuming user has basic knowledge on [Kubernetes](https://kubernetes.io/docs/tutorials/kubernetes-basics/) and Amazon [EKS](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)

## Abstract

This document will introduce the user to create an EKS cluster with `p4d.24xlarge` backed nodegroup with EFA and GPUDirect RDMA, to run an example NCCL-Test for Multi-node NCCL Performance via EFA. This workflow will also provide a template for distributed deep learning training on EKS using EFA.


## Step 1: Create EKS cluster

Create an empty EKS cluster via `eksctl`, at the moment, `eksctl` doesnt have support to create EFA supported nodegroup. In the later steps, the nodegroup would be created then join the EKS cluster.

```
eksctl create cluster --name=${cluster_name} \
 --region=us-west-2 \
 --ssh-access --ssh-public-key ~/.ssh/id_rsa.pub \
 --without-nodegroup
```
## Step 2: Create Launch Template, EFA Security Groups and Placement Groups

Specific to executing ML workloads and other applications that take advantage of GPUDirect RDMA backed with 4x100 Gbps EFA. Key requirements for EFA networking are creating the EFA specific security groups and placements groups as well launching the nodegroup in a single Available Zone. To simplify this lift we have packaged all the requirements in a cloudformation script which will create a launch template. This launch template can be used in a managed nodegroup.

In AWS CloudFormation Console upload the cloudformation script in [cloudformation/efa-p4d-managed-nodegroup.yaml](cloudformation/efa-p4d-managed-nodegroup.yaml) apply fill in the relevant details:

- In `ClusterControlPlaneSecurityGroup`, use the control plan security group (the name contains `control plan`) which is automatically created by EKS when user created via eksctl.
- In `NodeImageIdSSMParam`, ensure it is using Amazon EKS-optimized accelerated AMI: `/aws/service/eks/optimized-ami/1.19/amazon-linux-2-gpu/recommended/image_id`.
- In `Subnetid`, it is recommended to put in a private subnet. To select a subnet where p4d instances are available in a region you can run this comman:
```
aws ec2 describe-instance-type-offerings --location-type availability-zone --filters Name=instance-type,Values=<node type> —region <region>
```

When the cloud formation is in `CREATE COMPLETE` continue to Step 3.

## Step 3: 

To complete the EKS setup we can now associate the created launch template with the EKS cluster. All aws cli command line flags can be retireved from the cloudformation `Outputs` sections

```
aws eks create-nodegroup --cluster-name <cluster_name> --nodegroup-name <node_group_name> --node-role <role_arn> --subnets <subnet_id> --launch-template id=<launch_template_id>,version=1
```

After the nodegroup is created, check the eks cluster for the nodes to join the cluster.
```
watch kubectl get nodes
```
The nodes should join after a few moments
```
NAME                                          STATUS   ROLES    AGE   VERSION
ip-172-31-65-211.us-west-2.compute.internal   Ready    <none>   65m   v1.19.6-eks-49a6c0
ip-172-31-89-239.us-west-2.compute.internal   Ready    <none>   65m   v1.19.6-eks-49a6c0
```

## Step 4: Install AWS EFA EKS Plugin, NVIDIA Device Plugin, Kubeflow MPIOperator

In order to present the underlying pod(s) with the EFA RDMA devices we are providing a device plugin you can install 
TODO: Provide a permament home EFA EKS plugin
```
kubectl apply -f manifest/efa-k8s-device-plugin.yml
```
Next apply the [NVIDIA device plugin](https://github.com/NVIDIA/k8s-device-plugin) DaemonSet to present the pods with the underyling GPU devices.
```
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.8.2/nvidia-device-plugin.yml
```
For the NCCL tests we will now apply the Kubeflow MPIOperator which will help manage the pod networking.
```
git clone https://github.com/kubeflow/mpi-operator
cd mpi-operator
kubectl create -f deploy/v1alpha2/mpi-operator.yaml
```

## Step 5: Run the Multi-node NCCL Performance Test on the EKS cluster for verifying GPUDirectRDMA/EFA

To check NCCL Performance with GPUDirectRDMA over EFA, run the standard NCCL performance test that is available on the official [NCCL-Tests Repo](https://github.com/NVIDIA/nccl-tests.git). The Dockerfile comes with this test already built for both CUDA 11.2 and the latest version of EFA/AWS-OFI-NCCL. You can similarly run Kubernetes job without EFA. Alternatively we provide a docker image available in ECR Public here:
[public.ecr.aws/w6p6i9i7/aws-efa-nccl-rdma:base-cudnn8-cuda11-ubuntu18.04](https://gallery.ecr.aws/w6p6i9i7/aws-efa-nccl-rdma)

**HugePages**
The most important modification required in a Kubernetes job for adopting EFA is configuring and managing [Huge Pages](https://kubernetes.io/docs/tasks/manage-hugepages/scheduling-hugepages/) as a schedulable resource in the cluster. Currently, nodes with EFA support pre-allocates 5128 2M Huge Pages.

The example to run 2 node NCCL Performance Test  is `examples/nccl-efa-tests.yaml`.  In the example NCCL-test job, each worker requested 8 gpus which would allow cluster to allocate two nodes in two P4d instances,  and also 256Mi hugepages-2Mi and 8000Mi.


Create NCCL-tests job via command:
```
$ kubectl create -f examples/nccl-efa-tests.yaml
mpijob.kubeflow.org/nccl-test-debug created
```
The MPIOperator creates a launcher pod and 2 workers pods (one on each node)
```
NAME                            READY   STATUS              RESTARTS   AGE
nccl-tests-efa-launcher-wzr8j   0/1     Init:0/1            0          5s
nccl-tests-efa-worker-0         0/1     ContainerCreating   0          5s
nccl-tests-efa-worker-1         0/1     ContainerCreating   0          5s
```
The whole log returned by command:

```
$ kubectl logs -f nccl-tests-efa-launcher-wzr8j
```
<details>
<summary>Click to check details logs</summary>

```
+ POD_NAME=nccl-tests-efa-worker-0
+ [ n = - ]
+ shift
+ /opt/kube/kubectl cp /opt/kube/hosts nccl-tests-efa-worker-0:/etc/hosts_of_nodes
+ POD_NAME=nccl-tests-efa-worker-1
+ [ n = - ]
+ shift
+ /opt/kube/kubectl cp /opt/kube/hosts nccl-tests-efa-worker-1:/etc/hosts_of_nodes
+ /opt/kube/kubectl exec nccl-tests-efa-worker-0 -- /bin/sh -c cat /etc/hosts_of_nodes >> /etc/hosts &&        PATH=/opt/amazon/openmpi/bin:$PATH ; export PATH ; LD_LIBRARY_PATH=/opt/amazon/openmpi/lib:$LD_LIBRARY_PATH ; export LD_LIBRARY_PATH ; DYLD_LIBRARY_PATH=/opt/amazon/openmpi/lib:$DYLD_LIBRARY_PATH ; export DYLD_LIBRARY_PATH ;   /opt/amazon/openmpi/bin/orted -mca ess "env" -mca ess_base_jobid "1279000576" -mca ess_base_vpid 1 -mca ess_base_num_procs "3" -mca orte_node_regex "nccl-tests-efa-launcher-fh[1:7]m4,nccl-tests-efa-worker-[1:0-1]@0(3)" -mca orte_hnp_uri "1279000576.0;tcp://172.31.87.173:50623" --mca pml "^cm" -mca plm "rsh" --tree-spawn -mca routed "radix" -mca orte_parent_uri "1279000576.0;tcp://172.31.87.173:50623" -mca plm_rsh_agent "/etc/mpi/kubexec.sh" -mca orte_default_hostfile "/etc/mpi/hostfile" -mca orte_tag_output "1" -mca hwloc_base_binding_policy "none" -mca rmaps_base_mapping_policy "slot" -mca rmaps_base_oversubscribe "1" -mca pmix "^s1,s2,cray,isolated"
+ /opt/kube/kubectl exec nccl-tests-efa-worker-1 -- /bin/sh -c cat /etc/hosts_of_nodes >> /etc/hosts &&        PATH=/opt/amazon/openmpi/bin:$PATH ; export PATH ; LD_LIBRARY_PATH=/opt/amazon/openmpi/lib:$LD_LIBRARY_PATH ; export LD_LIBRARY_PATH ; DYLD_LIBRARY_PATH=/opt/amazon/openmpi/lib:$DYLD_LIBRARY_PATH ; export DYLD_LIBRARY_PATH ;   /opt/amazon/openmpi/bin/orted -mca ess "env" -mca ess_base_jobid "1279000576" -mca ess_base_vpid 2 -mca ess_base_num_procs "3" -mca orte_node_regex "nccl-tests-efa-launcher-fh[1:7]m4,nccl-tests-efa-worker-[1:0-1]@0(3)" -mca orte_hnp_uri "1279000576.0;tcp://172.31.87.173:50623" --mca pml "^cm" -mca plm "rsh" --tree-spawn -mca routed "radix" -mca orte_parent_uri "1279000576.0;tcp://172.31.87.173:50623" -mca plm_rsh_agent "/etc/mpi/kubexec.sh" -mca orte_default_hostfile "/etc/mpi/hostfile" -mca orte_tag_output "1" -mca hwloc_base_binding_policy "none" -mca rmaps_base_mapping_policy "slot" -mca rmaps_base_oversubscribe "1" -mca pmix "^s1,s2,cray,isolated"
[1,0]<stdout>:# nThread 1 nGpus 1 minBytes 8 maxBytes 1073741824 step: 2(factor) warmup iters: 5 iters: 1000 validation: 1 
[1,0]<stdout>:#
[1,0]<stdout>:# Using devices
[1,0]<stdout>:#   Rank  0 Pid     26 on nccl-tests-efa-worker-0 device  0 [0x10] A100-SXM4-40GB
[1,0]<stdout>:#   Rank  1 Pid     27 on nccl-tests-efa-worker-0 device  1 [0x10] A100-SXM4-40GB
[1,0]<stdout>:#   Rank  2 Pid     28 on nccl-tests-efa-worker-0 device  2 [0x20] A100-SXM4-40GB
[1,0]<stdout>:#   Rank  3 Pid     29 on nccl-tests-efa-worker-0 device  3 [0x20] A100-SXM4-40GB
[1,0]<stdout>:#   Rank  4 Pid     30 on nccl-tests-efa-worker-0 device  4 [0x90] A100-SXM4-40GB
[1,0]<stdout>:#   Rank  5 Pid     31 on nccl-tests-efa-worker-0 device  5 [0x90] A100-SXM4-40GB
[1,0]<stdout>:#   Rank  6 Pid     33 on nccl-tests-efa-worker-0 device  6 [0xa0] A100-SXM4-40GB
[1,0]<stdout>:#   Rank  7 Pid     35 on nccl-tests-efa-worker-0 device  7 [0xa0] A100-SXM4-40GB
[1,0]<stdout>:#   Rank  8 Pid     25 on nccl-tests-efa-worker-1 device  0 [0x10] A100-SXM4-40GB
[1,0]<stdout>:#   Rank  9 Pid     26 on nccl-tests-efa-worker-1 device  1 [0x10] A100-SXM4-40GB
[1,0]<stdout>:#   Rank 10 Pid     27 on nccl-tests-efa-worker-1 device  2 [0x20] A100-SXM4-40GB
[1,0]<stdout>:#   Rank 11 Pid     28 on nccl-tests-efa-worker-1 device  3 [0x20] A100-SXM4-40GB
[1,0]<stdout>:#   Rank 12 Pid     29 on nccl-tests-efa-worker-1 device  4 [0x90] A100-SXM4-40GB
[1,0]<stdout>:#   Rank 13 Pid     30 on nccl-tests-efa-worker-1 device  5 [0x90] A100-SXM4-40GB
[1,0]<stdout>:#   Rank 14 Pid     31 on nccl-tests-efa-worker-1 device  6 [0xa0] A100-SXM4-40GB
[1,0]<stdout>:#   Rank 15 Pid     32 on nccl-tests-efa-worker-1 device  7 [0xa0] A100-SXM4-40GB
[1,0]<stdout>:nccl-tests-efa-worker-0:26:26 [0] NCCL INFO Bootstrap : Using eth0:172.31.94.17<0>
[1,0]<stdout>:nccl-tests-efa-worker-0:26:26 [0] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,0]<stdout>:nccl-tests-efa-worker-0:26:26 [0] NCCL INFO NET/OFI Selected Provider is efa
[1,0]<stdout>:nccl-tests-efa-worker-0:26:26 [0] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,0]<stdout>:nccl-tests-efa-worker-0:26:26 [0] NCCL INFO Using network AWS Libfabric
[1,0]<stdout>:NCCL version 2.8.3+cuda11.2
[1,10]<stdout>:nccl-tests-efa-worker-1:27:27 [2] NCCL INFO Bootstrap : Using eth0:172.31.77.37<0>
[1,10]<stdout>:nccl-tests-efa-worker-1:27:27 [2] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,10]<stdout>:nccl-tests-efa-worker-1:27:27 [2] NCCL INFO NET/OFI Selected Provider is efa
[1,10]<stdout>:nccl-tests-efa-worker-1:27:27 [2] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,10]<stdout>:nccl-tests-efa-worker-1:27:27 [2] NCCL INFO Using network AWS Libfabric
[1,7]<stdout>:nccl-tests-efa-worker-0:35:35 [7] NCCL INFO Bootstrap : Using eth0:172.31.94.17<0>
[1,7]<stdout>:nccl-tests-efa-worker-0:35:35 [7] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,7]<stdout>:nccl-tests-efa-worker-0:35:35 [7] NCCL INFO NET/OFI Selected Provider is efa
[1,7]<stdout>:nccl-tests-efa-worker-0:35:35 [7] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,7]<stdout>:nccl-tests-efa-worker-0:35:35 [7] NCCL INFO Using network AWS Libfabric
[1,2]<stdout>:nccl-tests-efa-worker-0:28:28 [2] NCCL INFO Bootstrap : Using eth0:172.31.94.17<0>
[1,2]<stdout>:nccl-tests-efa-worker-0:28:28 [2] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,2]<stdout>:nccl-tests-efa-worker-0:28:28 [2] NCCL INFO NET/OFI Selected Provider is efa
[1,2]<stdout>:nccl-tests-efa-worker-0:28:28 [2] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,2]<stdout>:nccl-tests-efa-worker-0:28:28 [2] NCCL INFO Using network AWS Libfabric
[1,3]<stdout>:nccl-tests-efa-worker-0:29:29 [3] NCCL INFO Bootstrap : Using eth0:172.31.94.17<0>
[1,4]<stdout>:nccl-tests-efa-worker-0:30:30 [4] NCCL INFO Bootstrap : Using eth0:172.31.94.17<0>
[1,6]<stdout>:nccl-tests-efa-worker-0:33:33 [6] NCCL INFO Bootstrap : Using eth0:172.31.94.17<0>
[1,3]<stdout>:nccl-tests-efa-worker-0:29:29 [3] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,6]<stdout>:nccl-tests-efa-worker-0:33:33 [6] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,4]<stdout>:nccl-tests-efa-worker-0:30:30 [4] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,3]<stdout>:nccl-tests-efa-worker-0:29:29 [3] NCCL INFO NET/OFI Selected Provider is efa
[1,3]<stdout>:nccl-tests-efa-worker-0:29:29 [3] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,3]<stdout>:nccl-tests-efa-worker-0:29:29 [3] NCCL INFO Using network AWS Libfabric
[1,6]<stdout>:nccl-tests-efa-worker-0:33:33 [6] NCCL INFO NET/OFI Selected Provider is efa
[1,6]<stdout>:nccl-tests-efa-worker-0:33:33 [6] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,6]<stdout>:nccl-tests-efa-worker-0:33:33 [6] NCCL INFO Using network AWS Libfabric
[1,4]<stdout>:nccl-tests-efa-worker-0:30:30 [4] NCCL INFO NET/OFI Selected Provider is efa
[1,4]<stdout>:nccl-tests-efa-worker-0:30:30 [4] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,4]<stdout>:nccl-tests-efa-worker-0:30:30 [4] NCCL INFO Using network AWS Libfabric
[1,5]<stdout>:nccl-tests-efa-worker-0:31:31 [5] NCCL INFO Bootstrap : Using eth0:172.31.94.17<0>
[1,5]<stdout>:nccl-tests-efa-worker-0:31:31 [5] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,5]<stdout>:nccl-tests-efa-worker-0:31:31 [5] NCCL INFO NET/OFI Selected Provider is efa
[1,5]<stdout>:nccl-tests-efa-worker-0:31:31 [5] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,5]<stdout>:nccl-tests-efa-worker-0:31:31 [5] NCCL INFO Using network AWS Libfabric
[1,1]<stdout>:nccl-tests-efa-worker-0:27:27 [1] NCCL INFO Bootstrap : Using eth0:172.31.94.17<0>
[1,1]<stdout>:nccl-tests-efa-worker-0:27:27 [1] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,1]<stdout>:nccl-tests-efa-worker-0:27:27 [1] NCCL INFO NET/OFI Selected Provider is efa
[1,1]<stdout>:nccl-tests-efa-worker-0:27:27 [1] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,1]<stdout>:nccl-tests-efa-worker-0:27:27 [1] NCCL INFO Using network AWS Libfabric
[1,9]<stdout>:nccl-tests-efa-worker-1:26:26 [1] NCCL INFO Bootstrap : Using eth0:172.31.77.37<0>
[1,13]<stdout>:nccl-tests-efa-worker-1:30:30 [5] NCCL INFO Bootstrap : Using eth0:172.31.77.37<0>
[1,13]<stdout>:nccl-tests-efa-worker-1:30:30 [5] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,9]<stdout>:nccl-tests-efa-worker-1:26:26 [1] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,9]<stdout>:nccl-tests-efa-worker-1:26:26 [1] NCCL INFO NET/OFI Selected Provider is efa
[1,9]<stdout>:nccl-tests-efa-worker-1:26:26 [1] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,9]<stdout>:nccl-tests-efa-worker-1:26:26 [1] NCCL INFO Using network AWS Libfabric
[1,13]<stdout>:nccl-tests-efa-worker-1:30:30 [5] NCCL INFO NET/OFI Selected Provider is efa
[1,13]<stdout>:nccl-tests-efa-worker-1:30:30 [5] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,13]<stdout>:nccl-tests-efa-worker-1:30:30 [5] NCCL INFO Using network AWS Libfabric
[1,15]<stdout>:nccl-tests-efa-worker-1:32:32 [7] NCCL INFO Bootstrap : Using eth0:172.31.77.37<0>
[1,8]<stdout>:nccl-tests-efa-worker-1:25:25 [0] NCCL INFO Bootstrap : Using eth0:172.31.77.37<0>
[1,15]<stdout>:nccl-tests-efa-worker-1:32:32 [7] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,8]<stdout>:nccl-tests-efa-worker-1:25:25 [0] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,15]<stdout>:nccl-tests-efa-worker-1:32:32 [7] NCCL INFO NET/OFI Selected Provider is efa
[1,15]<stdout>:nccl-tests-efa-worker-1:32:32 [7] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,15]<stdout>:nccl-tests-efa-worker-1:32:32 [7] NCCL INFO Using network AWS Libfabric
[1,8]<stdout>:nccl-tests-efa-worker-1:25:25 [0] NCCL INFO NET/OFI Selected Provider is efa
[1,8]<stdout>:nccl-tests-efa-worker-1:25:25 [0] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,8]<stdout>:nccl-tests-efa-worker-1:25:25 [0] NCCL INFO Using network AWS Libfabric
[1,11]<stdout>:nccl-tests-efa-worker-1:28:28 [3] NCCL INFO Bootstrap : Using eth0:172.31.77.37<0>
[1,14]<stdout>:nccl-tests-efa-worker-1:31:31 [6] NCCL INFO Bootstrap : Using eth0:172.31.77.37<0>
[1,11]<stdout>:nccl-tests-efa-worker-1:28:28 [3] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,14]<stdout>:nccl-tests-efa-worker-1:31:31 [6] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,11]<stdout>:nccl-tests-efa-worker-1:28:28 [3] NCCL INFO NET/OFI Selected Provider is efa
[1,11]<stdout>:nccl-tests-efa-worker-1:28:28 [3] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,11]<stdout>:nccl-tests-efa-worker-1:28:28 [3] NCCL INFO Using network AWS Libfabric
[1,14]<stdout>:nccl-tests-efa-worker-1:31:31 [6] NCCL INFO NET/OFI Selected Provider is efa
[1,14]<stdout>:nccl-tests-efa-worker-1:31:31 [6] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,14]<stdout>:nccl-tests-efa-worker-1:31:31 [6] NCCL INFO Using network AWS Libfabric
[1,12]<stdout>:nccl-tests-efa-worker-1:29:29 [4] NCCL INFO Bootstrap : Using eth0:172.31.77.37<0>
[1,12]<stdout>:nccl-tests-efa-worker-1:29:29 [4] NCCL INFO NET/OFI Running on P4d platform, Setting NCCL_TOPO_FILE environment variable to /opt/aws-ofi-nccl/install/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml
[1,12]<stdout>:nccl-tests-efa-worker-1:29:29 [4] NCCL INFO NET/OFI Selected Provider is efa
[1,12]<stdout>:nccl-tests-efa-worker-1:29:29 [4] NCCL INFO NET/Plugin: Failed to find ncclCollNetPlugin_v4 symbol.
[1,12]<stdout>:nccl-tests-efa-worker-1:29:29 [4] NCCL INFO Using network AWS Libfabric
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Trees [0] 3/-1/-1->2->1 [1] 3/10/-1->2->-1 [2] 3/-1/-1->2->1 [3] 3/-1/-1->2->1 [4] 3/-1/-1->2->1 [5] 3/-1/-1->2->10 [6] 3/-1/-1->2->1 [7] 3/-1/-1->2->1
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Setting affinity for GPU 2 to ff,ffff0000,00ffffff
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Trees [0] 4/-1/-1->3->2 [1] 4/-1/-1->3->2 [2] -1/-1/-1->3->2 [3] 4/-1/-1->3->2 [4] 4/-1/-1->3->2 [5] 4/-1/-1->3->2 [6] -1/-1/-1->3->2 [7] 4/-1/-1->3->2
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Setting affinity for GPU 3 to ff,ffff0000,00ffffff
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Trees [0] 5/-1/-1->4->3 [1] 5/-1/-1->4->3 [2] 5/12/-1->4->-1 [3] 5/-1/-1->4->3 [4] 5/-1/-1->4->3 [5] 5/-1/-1->4->3 [6] 5/-1/-1->4->12 [7] 5/-1/-1->4->3
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Setting affinity for GPU 4 to ffffff00,0000ffff,ff000000
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Trees [0] 6/-1/-1->5->4 [1] 6/-1/-1->5->4 [2] 6/-1/-1->5->4 [3] -1/-1/-1->5->4 [4] 6/-1/-1->5->4 [5] 6/-1/-1->5->4 [6] 6/-1/-1->5->4 [7] -1/-1/-1->5->4
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Setting affinity for GPU 5 to ffffff00,0000ffff,ff000000
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Trees [0] 7/-1/-1->6->5 [1] 7/-1/-1->6->5 [2] 7/-1/-1->6->5 [3] 7/14/-1->6->-1 [4] 7/-1/-1->6->5 [5] 7/-1/-1->6->5 [6] 7/-1/-1->6->5 [7] 7/-1/-1->6->14
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Setting affinity for GPU 6 to ffffff00,0000ffff,ff000000
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Trees [0] -1/-1/-1->7->6 [1] 0/-1/-1->7->6 [2] 0/-1/-1->7->6 [3] 0/-1/-1->7->6 [4] -1/-1/-1->7->6 [5] 0/-1/-1->7->6 [6] 0/-1/-1->7->6 [7] 0/-1/-1->7->6
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Setting affinity for GPU 7 to ffffff00,0000ffff,ff000000
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Trees [0] 9/-1/-1->8->0 [1] 9/-1/-1->8->15 [2] 9/-1/-1->8->15 [3] 9/-1/-1->8->15 [4] 9/0/-1->8->-1 [5] 9/-1/-1->8->15 [6] 9/-1/-1->8->15 [7] 9/-1/-1->8->15
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Setting affinity for GPU 0 to ff,ffff0000,00ffffff
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Trees [0] 10/-1/-1->9->8 [1] -1/-1/-1->9->8 [2] 10/-1/-1->9->8 [3] 10/-1/-1->9->8 [4] 10/-1/-1->9->8 [5] -1/-1/-1->9->8 [6] 10/-1/-1->9->8 [7] 10/-1/-1->9->8
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Setting affinity for GPU 1 to ff,ffff0000,00ffffff
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Trees [0] 11/-1/-1->10->9 [1] 11/-1/-1->10->2 [2] 11/-1/-1->10->9 [3] 11/-1/-1->10->9 [4] 11/-1/-1->10->9 [5] 11/2/-1->10->-1 [6] 11/-1/-1->10->9 [7] 11/-1/-1->10->9
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Setting affinity for GPU 2 to ff,ffff0000,00ffffff
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Trees [0] 12/-1/-1->11->10 [1] 12/-1/-1->11->10 [2] -1/-1/-1->11->10 [3] 12/-1/-1->11->10 [4] 12/-1/-1->11->10 [5] 12/-1/-1->11->10 [6] -1/-1/-1->11->10 [7] 12/-1/-1->11->10
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Setting affinity for GPU 3 to ff,ffff0000,00ffffff
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Trees [0] 13/-1/-1->12->11 [1] 13/-1/-1->12->11 [2] 13/-1/-1->12->4 [3] 13/-1/-1->12->11 [4] 13/-1/-1->12->11 [5] 13/-1/-1->12->11 [6] 13/4/-1->12->-1 [7] 13/-1/-1->12->11
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Trees [0] 14/-1/-1->13->12 [1] 14/-1/-1->13->12 [2] 14/-1/-1->13->12 [3] -1/-1/-1->13->12 [4] 14/-1/-1->13->12 [5] 14/-1/-1->13->12 [6] 14/-1/-1->13->12 [7] -1/-1/-1->13->12
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Setting affinity for GPU 5 to ffffff00,0000ffff,ff000000
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Setting affinity for GPU 4 to ffffff00,0000ffff,ff000000
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Trees [0] 2/-1/-1->1->0 [1] -1/-1/-1->1->0 [2] 2/-1/-1->1->0 [3] 2/-1/-1->1->0 [4] 2/-1/-1->1->0 [5] -1/-1/-1->1->0 [6] 2/-1/-1->1->0 [7] 2/-1/-1->1->0
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Trees [0] 15/-1/-1->14->13 [1] 15/-1/-1->14->13 [2] 15/-1/-1->14->13 [3] 15/-1/-1->14->6 [4] 15/-1/-1->14->13 [5] 15/-1/-1->14->13 [6] 15/-1/-1->14->13 [7] 15/6/-1->14->-1
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Setting affinity for GPU 6 to ffffff00,0000ffff,ff000000
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Trees [0] -1/-1/-1->15->14 [1] 8/-1/-1->15->14 [2] 8/-1/-1->15->14 [3] 8/-1/-1->15->14 [4] -1/-1/-1->15->14 [5] 8/-1/-1->15->14 [6] 8/-1/-1->15->14 [7] 8/-1/-1->15->14
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Setting affinity for GPU 7 to ffffff00,0000ffff,ff000000
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 00/08 :    0   7   6   5   4   3   2   1   8  15  14  13  12  11  10   9
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 01/08 :    0   3  10  15  14  13  12   9   8  11   2   7   6   5   4   1
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 02/08 :    0   7   6   5  12  11  10   9   8  15  14  13   4   3   2   1
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Setting affinity for GPU 1 to ff,ffff0000,00ffffff
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 03/08 :    0   5   4   7  14  11  10   9   8  13  12  15   6   3   2   1
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 04/08 :    0   7   6   5   4   3   2   1   8  15  14  13  12  11  10   9
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 05/08 :    0   3  10  15  14  13  12   9   8  11   2   7   6   5   4   1
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 06/08 :    0   7   6   5  12  11  10   9   8  15  14  13   4   3   2   1
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 07/08 :    0   5   4   7  14  11  10   9   8  13  12  15   6   3   2   1
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Trees [0] 1/8/-1->0->-1 [1] 1/-1/-1->0->7 [2] 1/-1/-1->0->7 [3] 1/-1/-1->0->7 [4] 1/-1/-1->0->8 [5] 1/-1/-1->0->7 [6] 1/-1/-1->0->7 [7] 1/-1/-1->0->7
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Setting affinity for GPU 0 to ff,ffff0000,00ffffff
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 01 : 2[201c0] -> 7[a01d0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 03 : 15[a01d0] -> 6[a01c0] [receive] via NET/AWS Libfabric/3/GDRDMA
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 03 : 4[901c0] -> 7[a01d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 00 : 1[101d0] -> 8[101c0] [send] via NET/AWS Libfabric/0/GDRDMA
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 03 : 7[a01d0] -> 14[a01c0] [receive] via NET/AWS Libfabric/3/GDRDMA
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 03 : 12[901c0] -> 15[a01d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 01 : 8[101c0] -> 11[201d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 01 : 0[101c0] -> 3[201d0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 01 : 10[201c0] -> 15[a01d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 00 : 9[101d0] -> 0[101c0] [send] via NET/AWS Libfabric/0/GDRDMA
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 07 : 12[901c0] -> 15[a01d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 05 : 8[101c0] -> 11[201d0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 05 : 2[201c0] -> 7[a01d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 04 : 1[101d0] -> 8[101c0] [send] via NET/AWS Libfabric/0/GDRDMA
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 07 : 4[901c0] -> 7[a01d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 05 : 0[101c0] -> 3[201d0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 05 : 10[201c0] -> 15[a01d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 04 : 9[101d0] -> 0[101c0] [send] via NET/AWS Libfabric/0/GDRDMA
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 02 : 13[901d0] -> 4[901c0] [receive] via NET/AWS Libfabric/2/GDRDMA
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 01 : 3[201d0] -> 10[201c0] [send] via NET/AWS Libfabric/1/GDRDMA
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 03 : 0[101c0] -> 5[901d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 02 : 5[901d0] -> 12[901c0] [receive] via NET/AWS Libfabric/2/GDRDMA
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 03 : 8[101c0] -> 13[901d0] via P2P/IPC/read
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 01 : 11[201d0] -> 2[201c0] [send] via NET/AWS Libfabric/1/GDRDMA
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 05 : 3[201d0] -> 10[201c0] [send] via NET/AWS Libfabric/1/GDRDMA
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 05 : 11[201d0] -> 2[201c0] [send] via NET/AWS Libfabric/1/GDRDMA
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 07 : 8[101c0] -> 13[901d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 07 : 0[101c0] -> 5[901d0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 01 : 11[201d0] -> 2[201c0] [receive] via NET/AWS Libfabric/1/GDRDMA
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 00 : 9[101d0] -> 0[101c0] [receive] via NET/AWS Libfabric/0/GDRDMA
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 02 : 5[901d0] -> 12[901c0] [send] via NET/AWS Libfabric/2/GDRDMA
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 06 : 5[901d0] -> 12[901c0] [send] via NET/AWS Libfabric/2/GDRDMA
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 00 : 1[101d0] -> 8[101c0] [receive] via NET/AWS Libfabric/0/GDRDMA
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 01 : 3[201d0] -> 10[201c0] [receive] via NET/AWS Libfabric/1/GDRDMA
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 02 : 13[901d0] -> 4[901c0] [send] via NET/AWS Libfabric/2/GDRDMA
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 06 : 13[901d0] -> 4[901c0] [send] via NET/AWS Libfabric/2/GDRDMA
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 03 : 7[a01d0] -> 14[a01c0] [send] via NET/AWS Libfabric/3/GDRDMA
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 07 : 7[a01d0] -> 14[a01c0] [send] via NET/AWS Libfabric/3/GDRDMA
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 03 : 15[a01d0] -> 6[a01c0] [send] via NET/AWS Libfabric/3/GDRDMA
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 07 : 15[a01d0] -> 6[a01c0] [send] via NET/AWS Libfabric/3/GDRDMA
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 07 : 15[a01d0] -> 6[a01c0] [receive] via NET/AWS Libfabric/3/GDRDMA
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 07 : 7[a01d0] -> 14[a01c0] [receive] via NET/AWS Libfabric/3/GDRDMA
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 06 : 13[901d0] -> 4[901c0] [receive] via NET/AWS Libfabric/2/GDRDMA
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 06 : 5[901d0] -> 12[901c0] [receive] via NET/AWS Libfabric/2/GDRDMA
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 05 : 3[201d0] -> 10[201c0] [receive] via NET/AWS Libfabric/1/GDRDMA
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 05 : 11[201d0] -> 2[201c0] [receive] via NET/AWS Libfabric/1/GDRDMA
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 04 : 1[101d0] -> 8[101c0] [receive] via NET/AWS Libfabric/0/GDRDMA
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 00 : 8[101c0] -> 15[a01d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 04 : 9[101d0] -> 0[101c0] [receive] via NET/AWS Libfabric/0/GDRDMA
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 02 : 8[101c0] -> 15[a01d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 04 : 8[101c0] -> 15[a01d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 00 : 0[101c0] -> 7[a01d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 02 : 0[101c0] -> 7[a01d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 06 : 8[101c0] -> 15[a01d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 04 : 0[101c0] -> 7[a01d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 06 : 0[101c0] -> 7[a01d0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 00 : 5[901d0] -> 4[901c0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 01 : 5[901d0] -> 4[901c0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 03 : 5[901d0] -> 4[901c0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 04 : 5[901d0] -> 4[901c0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 05 : 5[901d0] -> 4[901c0] via P2P/IPC/read
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 00 : 7[a01d0] -> 6[a01c0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 07 : 5[901d0] -> 4[901c0] via P2P/IPC/read
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 01 : 7[a01d0] -> 6[a01c0] via P2P/IPC/read
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 02 : 7[a01d0] -> 6[a01c0] via P2P/IPC/read
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 04 : 7[a01d0] -> 6[a01c0] via P2P/IPC/read
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 05 : 7[a01d0] -> 6[a01c0] via P2P/IPC/read
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 06 : 7[a01d0] -> 6[a01c0] via P2P/IPC/read
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 00 : 15[a01d0] -> 14[a01c0] via P2P/IPC/read
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 01 : 15[a01d0] -> 14[a01c0] via P2P/IPC/read
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 02 : 15[a01d0] -> 14[a01c0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 00 : 13[901d0] -> 12[901c0] via P2P/IPC/read
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 04 : 15[a01d0] -> 14[a01c0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 01 : 13[901d0] -> 12[901c0] via P2P/IPC/read
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 05 : 15[a01d0] -> 14[a01c0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 03 : 13[901d0] -> 12[901c0] via P2P/IPC/read
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 06 : 15[a01d0] -> 14[a01c0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 04 : 13[901d0] -> 12[901c0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 05 : 13[901d0] -> 12[901c0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 07 : 13[901d0] -> 12[901c0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 01 : 12[901c0] -> 9[101d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 05 : 12[901c0] -> 9[101d0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 01 : 4[901c0] -> 1[101d0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 05 : 4[901c0] -> 1[101d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 00 : 12[901c0] -> 11[201d0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 03 : 14[a01c0] -> 11[201d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 01 : 9[101d0] -> 8[101c0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 03 : 6[a01c0] -> 3[201d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 02 : 12[901c0] -> 11[201d0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 07 : 14[a01c0] -> 11[201d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 02 : 9[101d0] -> 8[101c0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 07 : 6[a01c0] -> 3[201d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 04 : 12[901c0] -> 11[201d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 03 : 9[101d0] -> 8[101c0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 06 : 12[901c0] -> 11[201d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 05 : 9[101d0] -> 8[101c0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 00 : 10[201c0] -> 9[101d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 06 : 9[101d0] -> 8[101c0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 00 : 4[901c0] -> 3[201d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 01 : 1[101d0] -> 0[101c0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 02 : 10[201c0] -> 9[101d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 07 : 9[101d0] -> 8[101c0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 02 : 4[901c0] -> 3[201d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 02 : 1[101d0] -> 0[101c0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 03 : 10[201c0] -> 9[101d0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 00 : 2[201c0] -> 1[101d0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 04 : 4[901c0] -> 3[201d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 03 : 1[101d0] -> 0[101c0] via P2P/IPC/read
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 00 : 11[201d0] -> 10[201c0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 04 : 10[201c0] -> 9[101d0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 06 : 4[901c0] -> 3[201d0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 02 : 2[201c0] -> 1[101d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 05 : 1[101d0] -> 0[101c0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 00 : 3[201d0] -> 2[201c0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Connected all rings
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 02 : 11[201d0] -> 10[201c0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 06 : 10[201c0] -> 9[101d0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 03 : 2[201c0] -> 1[101d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 06 : 1[101d0] -> 0[101c0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 00 : 8[101c0] -> 9[101d0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 02 : 3[201d0] -> 2[201c0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 00 : 14[a01c0] -> 13[901d0] via P2P/IPC/read
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 03 : 11[201d0] -> 10[201c0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 07 : 10[201c0] -> 9[101d0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 04 : 2[201c0] -> 1[101d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 07 : 1[101d0] -> 0[101c0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 00 : 6[a01c0] -> 5[901d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 01 : 8[101c0] -> 9[101d0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 03 : 3[201d0] -> 2[201c0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 01 : 14[a01c0] -> 13[901d0] via P2P/IPC/read
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 04 : 11[201d0] -> 10[201c0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 06 : 2[201c0] -> 1[101d0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 01 : 6[a01c0] -> 5[901d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 02 : 8[101c0] -> 9[101d0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 04 : 3[201d0] -> 2[201c0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 02 : 14[a01c0] -> 13[901d0] via P2P/IPC/read
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 06 : 11[201d0] -> 10[201c0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 07 : 2[201c0] -> 1[101d0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 02 : 6[a01c0] -> 5[901d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 03 : 8[101c0] -> 9[101d0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 04 : 14[a01c0] -> 13[901d0] via P2P/IPC/read
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 07 : 11[201d0] -> 10[201c0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 06 : 3[201d0] -> 2[201c0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Connected all rings
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 04 : 6[a01c0] -> 5[901d0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 07 : 3[201d0] -> 2[201c0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 00 : 0[101c0] -> 1[101d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 04 : 8[101c0] -> 9[101d0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 05 : 14[a01c0] -> 13[901d0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 05 : 6[a01c0] -> 5[901d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 01 : 0[101c0] -> 1[101d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 05 : 8[101c0] -> 9[101d0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 06 : 14[a01c0] -> 13[901d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Connected all rings
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 06 : 6[a01c0] -> 5[901d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 02 : 0[101c0] -> 1[101d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 06 : 8[101c0] -> 9[101d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 03 : 0[101c0] -> 1[101d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 07 : 8[101c0] -> 9[101d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Connected all rings
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Connected all rings
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Connected all rings
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Connected all rings
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Connected all rings
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Connected all rings
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Connected all rings
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Connected all rings
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 04 : 0[101c0] -> 1[101d0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Connected all rings
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 05 : 0[101c0] -> 1[101d0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Connected all rings
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Connected all rings
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 06 : 0[101c0] -> 1[101d0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Connected all rings
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Connected all rings
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 07 : 0[101c0] -> 1[101d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 00 : 12[901c0] -> 13[901d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 00 : 9[101d0] -> 10[201c0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 00 : 10[201c0] -> 11[201d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 01 : 12[901c0] -> 13[901d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 02 : 9[101d0] -> 10[201c0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 01 : 10[201c0] -> 11[201d0] via P2P/IPC/read
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 00 : 11[201d0] -> 12[901c0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 02 : 12[901c0] -> 13[901d0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 00 : 4[901c0] -> 5[901d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 03 : 9[101d0] -> 10[201c0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 00 : 14[a01c0] -> 15[a01d0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 02 : 10[201c0] -> 11[201d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 03 : 12[901c0] -> 13[901d0] via P2P/IPC/read
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 01 : 11[201d0] -> 12[901c0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 00 : 2[201c0] -> 3[201d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 04 : 9[101d0] -> 10[201c0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 01 : 4[901c0] -> 5[901d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 00 : 1[101d0] -> 2[201c0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 01 : 14[a01c0] -> 15[a01d0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 03 : 10[201c0] -> 11[201d0] via P2P/IPC/read
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 03 : 11[201d0] -> 12[901c0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 04 : 12[901c0] -> 13[901d0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 01 : 2[201c0] -> 3[201d0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 06 : 9[101d0] -> 10[201c0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 00 : 3[201d0] -> 4[901c0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 02 : 4[901c0] -> 5[901d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 02 : 1[101d0] -> 2[201c0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 00 : 6[a01c0] -> 7[a01d0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 02 : 14[a01c0] -> 15[a01d0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 00 : 13[901d0] -> 14[a01c0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 04 : 10[201c0] -> 11[201d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 05 : 12[901c0] -> 13[901d0] via P2P/IPC/read
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 04 : 11[201d0] -> 12[901c0] via P2P/IPC/read
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 07 : 9[101d0] -> 10[201c0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 01 : 3[201d0] -> 4[901c0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 02 : 2[201c0] -> 3[201d0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 03 : 4[901c0] -> 5[901d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 03 : 1[101d0] -> 2[201c0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 01 : 6[a01c0] -> 7[a01d0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 01 : 13[901d0] -> 14[a01c0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 03 : 14[a01c0] -> 15[a01d0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 05 : 10[201c0] -> 11[201d0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 03 : 2[201c0] -> 3[201d0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 03 : 3[201d0] -> 4[901c0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 04 : 4[901c0] -> 5[901d0] via P2P/IPC/read
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 05 : 11[201d0] -> 12[901c0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 06 : 12[901c0] -> 13[901d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 04 : 1[101d0] -> 2[201c0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 00 : 5[901d0] -> 6[a01c0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 02 : 6[a01c0] -> 7[a01d0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 04 : 3[201d0] -> 4[901c0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 04 : 2[201c0] -> 3[201d0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 04 : 14[a01c0] -> 15[a01d0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 02 : 13[901d0] -> 14[a01c0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 05 : 4[901c0] -> 5[901d0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 06 : 10[201c0] -> 11[201d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 06 : 1[101d0] -> 2[201c0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 07 : 12[901c0] -> 13[901d0] via P2P/IPC/read
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 07 : 11[201d0] -> 12[901c0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 03 : 6[a01c0] -> 7[a01d0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 01 : 5[901d0] -> 6[a01c0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 05 : 2[201c0] -> 3[201d0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 05 : 3[201d0] -> 4[901c0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 06 : 4[901c0] -> 5[901d0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 04 : 13[901d0] -> 14[a01c0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 05 : 14[a01c0] -> 15[a01d0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 07 : 10[201c0] -> 11[201d0] via P2P/IPC/read
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 07 : 1[101d0] -> 2[201c0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 02 : 5[901d0] -> 6[a01c0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 04 : 6[a01c0] -> 7[a01d0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 07 : 3[201d0] -> 4[901c0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 06 : 2[201c0] -> 3[201d0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 07 : 4[901c0] -> 5[901d0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 05 : 6[a01c0] -> 7[a01d0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 04 : 5[901d0] -> 6[a01c0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 07 : 2[201c0] -> 3[201d0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 06 : 14[a01c0] -> 15[a01d0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 05 : 13[901d0] -> 14[a01c0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 06 : 6[a01c0] -> 7[a01d0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 05 : 5[901d0] -> 6[a01c0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 01 : 8[101c0] -> 15[a01d0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 07 : 14[a01c0] -> 15[a01d0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 06 : 13[901d0] -> 14[a01c0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 07 : 6[a01c0] -> 7[a01d0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 06 : 5[901d0] -> 6[a01c0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 03 : 8[101c0] -> 15[a01d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 01 : 0[101c0] -> 7[a01d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 05 : 8[101c0] -> 15[a01d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 03 : 0[101c0] -> 7[a01d0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 07 : 8[101c0] -> 15[a01d0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 05 : 0[101c0] -> 7[a01d0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 01 : 2[201c0] -> 10[201c0] [receive] via NET/AWS Libfabric/1/GDRDMA
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 00 : 9[101d0] -> 8[101c0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 07 : 0[101c0] -> 7[a01d0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 05 : 2[201c0] -> 10[201c0] [receive] via NET/AWS Libfabric/1/GDRDMA
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Channel 04 : 9[101d0] -> 8[101c0] via P2P/IPC/read
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 01 : 10[201c0] -> 2[201c0] [receive] via NET/AWS Libfabric/1/GDRDMA
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 01 : 10[201c0] -> 2[201c0] [send] via NET/AWS Libfabric/1/GDRDMA
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 02 : 4[901c0] -> 12[901c0] [receive] via NET/AWS Libfabric/2/GDRDMA
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 03 : 6[a01c0] -> 14[a01c0] [receive] via NET/AWS Libfabric/3/GDRDMA
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 00 : 1[101d0] -> 0[101c0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 02 : 13[901d0] -> 12[901c0] via P2P/IPC/read
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Channel 05 : 10[201c0] -> 2[201c0] [send] via NET/AWS Libfabric/1/GDRDMA
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 06 : 4[901c0] -> 12[901c0] [receive] via NET/AWS Libfabric/2/GDRDMA
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 07 : 6[a01c0] -> 14[a01c0] [receive] via NET/AWS Libfabric/3/GDRDMA
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 05 : 10[201c0] -> 2[201c0] [receive] via NET/AWS Libfabric/1/GDRDMA
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Channel 06 : 13[901d0] -> 12[901c0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 02 : 12[901c0] -> 4[901c0] [send] via NET/AWS Libfabric/2/GDRDMA
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 03 : 14[a01c0] -> 6[a01c0] [send] via NET/AWS Libfabric/3/GDRDMA
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Channel 04 : 1[101d0] -> 0[101c0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 03 : 14[a01c0] -> 6[a01c0] [receive] via NET/AWS Libfabric/3/GDRDMA
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 01 : 2[201c0] -> 10[201c0] [send] via NET/AWS Libfabric/1/GDRDMA
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 02 : 12[901c0] -> 4[901c0] [receive] via NET/AWS Libfabric/2/GDRDMA
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 02 : 5[901d0] -> 4[901c0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 06 : 12[901c0] -> 4[901c0] [send] via NET/AWS Libfabric/2/GDRDMA
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 01 : 11[201d0] -> 10[201c0] via P2P/IPC/read
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Channel 07 : 14[a01c0] -> 6[a01c0] [send] via NET/AWS Libfabric/3/GDRDMA
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 07 : 14[a01c0] -> 6[a01c0] [receive] via NET/AWS Libfabric/3/GDRDMA
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Channel 05 : 2[201c0] -> 10[201c0] [send] via NET/AWS Libfabric/1/GDRDMA
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 06 : 12[901c0] -> 4[901c0] [receive] via NET/AWS Libfabric/2/GDRDMA
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 00 : 0[101c0] -> 8[101c0] [receive] via NET/AWS Libfabric/0/GDRDMA
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Channel 05 : 11[201d0] -> 10[201c0] via P2P/IPC/read
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 01 : 15[a01d0] -> 8[101c0] via P2P/IPC/read
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Channel 06 : 5[901d0] -> 4[901c0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 04 : 0[101c0] -> 8[101c0] [receive] via NET/AWS Libfabric/0/GDRDMA
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 02 : 15[a01d0] -> 8[101c0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 00 : 8[101c0] -> 0[101c0] [send] via NET/AWS Libfabric/0/GDRDMA
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 03 : 6[a01c0] -> 14[a01c0] [send] via NET/AWS Libfabric/3/GDRDMA
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 02 : 4[901c0] -> 12[901c0] [send] via NET/AWS Libfabric/2/GDRDMA
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 03 : 15[a01d0] -> 8[101c0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Channel 04 : 8[101c0] -> 0[101c0] [send] via NET/AWS Libfabric/0/GDRDMA
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 05 : 15[a01d0] -> 8[101c0] via P2P/IPC/read
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 06 : 15[a01d0] -> 8[101c0] via P2P/IPC/read
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 07 : 15[a01d0] -> 8[101c0] via P2P/IPC/read
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Channel 07 : 6[a01c0] -> 14[a01c0] [send] via NET/AWS Libfabric/3/GDRDMA
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 06 : 4[901c0] -> 12[901c0] [send] via NET/AWS Libfabric/2/GDRDMA
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 01 : 3[201d0] -> 2[201c0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 00 : 8[101c0] -> 0[101c0] [receive] via NET/AWS Libfabric/0/GDRDMA
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 01 : 7[a01d0] -> 0[101c0] via P2P/IPC/read
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Channel 05 : 3[201d0] -> 2[201c0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 04 : 8[101c0] -> 0[101c0] [receive] via NET/AWS Libfabric/0/GDRDMA
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 02 : 7[a01d0] -> 0[101c0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 00 : 0[101c0] -> 8[101c0] [send] via NET/AWS Libfabric/0/GDRDMA
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO Connected all trees
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,10]<stdout>:nccl-tests-efa-worker-1:27:65 [2] NCCL INFO comm 0x7f8cf8000dc0 rank 10 nranks 16 cudaDev 2 busId 201c0 - Init COMPLETE
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 03 : 7[a01d0] -> 0[101c0] via P2P/IPC/read
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Channel 04 : 0[101c0] -> 8[101c0] [send] via NET/AWS Libfabric/0/GDRDMA
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 05 : 7[a01d0] -> 0[101c0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 01 : 12[901c0] -> 11[201d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 03 : 12[901c0] -> 11[201d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 05 : 12[901c0] -> 11[201d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Channel 07 : 12[901c0] -> 11[201d0] via P2P/IPC/read
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 06 : 7[a01d0] -> 0[101c0] via P2P/IPC/read
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO Connected all trees
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,13]<stdout>:nccl-tests-efa-worker-1:30:71 [5] NCCL INFO comm 0x7fb15c000dc0 rank 13 nranks 16 cudaDev 5 busId 901d0 - Init COMPLETE
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO Connected all trees
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,2]<stdout>:nccl-tests-efa-worker-0:28:69 [2] NCCL INFO comm 0x7f5bfc000dc0 rank 2 nranks 16 cudaDev 2 busId 201c0 - Init COMPLETE
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 07 : 7[a01d0] -> 0[101c0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 01 : 4[901c0] -> 3[201d0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 03 : 4[901c0] -> 3[201d0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 05 : 4[901c0] -> 3[201d0] via P2P/IPC/read
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Channel 07 : 4[901c0] -> 3[201d0] via P2P/IPC/read
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO Connected all trees
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,12]<stdout>:nccl-tests-efa-worker-1:29:72 [4] NCCL INFO comm 0x7fc618000dc0 rank 12 nranks 16 cudaDev 4 busId 901c0 - Init COMPLETE
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO Connected all trees
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,11]<stdout>:nccl-tests-efa-worker-1:28:70 [3] NCCL INFO comm 0x7fa64c000dc0 rank 11 nranks 16 cudaDev 3 busId 201d0 - Init COMPLETE
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO Connected all trees
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,5]<stdout>:nccl-tests-efa-worker-0:31:74 [5] NCCL INFO comm 0x7f6218000dc0 rank 5 nranks 16 cudaDev 5 busId 901d0 - Init COMPLETE
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO Connected all trees
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,4]<stdout>:nccl-tests-efa-worker-0:30:70 [4] NCCL INFO comm 0x7f6d5c000dc0 rank 4 nranks 16 cudaDev 4 busId 901c0 - Init COMPLETE
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO Connected all trees
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,3]<stdout>:nccl-tests-efa-worker-0:29:71 [3] NCCL INFO comm 0x7efd04000dc0 rank 3 nranks 16 cudaDev 3 busId 201d0 - Init COMPLETE
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 03 : 15[a01d0] -> 14[a01c0] via P2P/IPC/read
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Channel 07 : 15[a01d0] -> 14[a01c0] via P2P/IPC/read
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO Connected all trees
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,8]<stdout>:nccl-tests-efa-worker-1:25:69 [0] NCCL INFO comm 0x7f9ef0000dc0 rank 8 nranks 16 cudaDev 0 busId 101c0 - Init COMPLETE
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO Connected all trees
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,9]<stdout>:nccl-tests-efa-worker-1:26:66 [1] NCCL INFO comm 0x7f3318000dc0 rank 9 nranks 16 cudaDev 1 busId 101d0 - Init COMPLETE
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO Connected all trees
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,15]<stdout>:nccl-tests-efa-worker-1:32:67 [7] NCCL INFO comm 0x7faa64000dc0 rank 15 nranks 16 cudaDev 7 busId a01d0 - Init COMPLETE
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO Connected all trees
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,14]<stdout>:nccl-tests-efa-worker-1:31:68 [6] NCCL INFO comm 0x7f4c74000dc0 rank 14 nranks 16 cudaDev 6 busId a01c0 - Init COMPLETE
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 03 : 7[a01d0] -> 6[a01c0] via P2P/IPC/read
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Channel 07 : 7[a01d0] -> 6[a01c0] via P2P/IPC/read
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO Connected all trees
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,7]<stdout>:nccl-tests-efa-worker-0:35:68 [7] NCCL INFO comm 0x7fc1dc000dc0 rank 7 nranks 16 cudaDev 7 busId a01d0 - Init COMPLETE
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO Connected all trees
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,6]<stdout>:nccl-tests-efa-worker-0:33:73 [6] NCCL INFO comm 0x7fb118000dc0 rank 6 nranks 16 cudaDev 6 busId a01c0 - Init COMPLETE
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO Connected all trees
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,0]<stdout>:nccl-tests-efa-worker-0:26:67 [0] NCCL INFO comm 0x7ff70c000dc0 rank 0 nranks 16 cudaDev 0 busId 101c0 - Init COMPLETE
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO Connected all trees
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO threadThresholds 8/8/64 | 128/8/64 | 8/8/64
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO 8 coll channels, 8 p2p channels, 1 p2p channels per peer
[1,0]<stdout>:#
[1,0]<stdout>:#                                                     out-of-place                       in-place          
[1,0]<stdout>:#       size         count    type   redop     time   algbw   busbw  error     time   algbw   busbw  error
[1,0]<stdout>:#        (B)    (elements)                     (us)  (GB/s)  (GB/s)            (us)  (GB/s)  (GB/s)       
[1,0]<stdout>:nccl-tests-efa-worker-0:26:26 [0] NCCL INFO Launch mode Parallel
[1,1]<stdout>:nccl-tests-efa-worker-0:27:72 [1] NCCL INFO comm 0x7fece4000dc0 rank 1 nranks 16 cudaDev 1 busId 101d0 - Init COMPLETE
[1,0]<stdout>:           8             2   float     sum    167.5    0.00    0.00  2e-07    167.1    0.00    0.00  1e-07
[1,0]<stdout>:          16             4   float     sum    167.3    0.00    0.00  1e-07    167.4    0.00    0.00  1e-07
[1,0]<stdout>:          32             8   float     sum    167.9    0.00    0.00  1e-07    167.5    0.00    0.00  1e-07
[1,0]<stdout>:          64            16   float     sum    167.7    0.00    0.00  1e-07    167.7    0.00    0.00  6e-08
[1,0]<stdout>:         128            32   float     sum    168.0    0.00    0.00  6e-08    167.9    0.00    0.00  6e-08
[1,0]<stdout>:         256            64   float     sum    168.6    0.00    0.00  6e-08    168.9    0.00    0.00  6e-08
[1,0]<stdout>:         512           128   float     sum    374.7    0.00    0.00  6e-08    170.1    0.00    0.01  6e-08
[1,0]<stdout>:        1024           256   float     sum    182.5    0.01    0.01  5e-07    182.3    0.01    0.01  5e-07
[1,0]<stdout>:        2048           512   float     sum    205.0    0.01    0.02  5e-07    205.0    0.01    0.02  5e-07
[1,0]<stdout>:        4096          1024   float     sum    233.3    0.02    0.03  5e-07    234.4    0.02    0.03  5e-07
[1,0]<stdout>:        8192          2048   float     sum    250.5    0.03    0.06  5e-07    249.5    0.03    0.06  5e-07
[1,0]<stdout>:       16384          4096   float     sum    254.2    0.06    0.12  5e-07    253.9    0.06    0.12  5e-07
[1,0]<stdout>:       32768          8192   float     sum    260.1    0.13    0.24  5e-07    259.7    0.13    0.24  5e-07
[1,0]<stdout>:       65536         16384   float     sum    273.9    0.24    0.45  5e-07    273.8    0.24    0.45  5e-07
[1,0]<stdout>:      131072         32768   float     sum    294.2    0.45    0.84  5e-07    294.2    0.45    0.84  5e-07
[1,0]<stdout>:      262144         65536   float     sum    304.9    0.86    1.61  5e-07    305.5    0.86    1.61  5e-07
[1,0]<stdout>:      524288        131072   float     sum    409.7    1.28    2.40  5e-07    410.3    1.28    2.40  5e-07
[1,0]<stdout>:     1048576        262144   float     sum    483.5    2.17    4.07  5e-07    483.6    2.17    4.07  5e-07
[1,0]<stdout>:     2097152        524288   float     sum    660.3    3.18    5.95  5e-07    672.4    3.12    5.85  5e-07
[1,0]<stdout>:     4194304       1048576   float     sum    817.0    5.13    9.63  5e-07    817.0    5.13    9.63  5e-07
[1,0]<stdout>:     8388608       2097152   float     sum   1228.0    6.83   12.81  5e-07   1223.6    6.86   12.85  5e-07
[1,0]<stdout>:    16777216       4194304   float     sum   1895.5    8.85   16.60  5e-07   1900.9    8.83   16.55  5e-07
[1,0]<stdout>:    33554432       8388608   float     sum   3106.8   10.80   20.25  5e-07   3104.1   10.81   20.27  5e-07
[1,0]<stdout>:    67108864      16777216   float     sum   5567.2   12.05   22.60  5e-07   5566.4   12.06   22.61  5e-07
[1,0]<stdout>:   134217728      33554432   float     sum   9388.6   14.30   26.80  5e-07   9343.3   14.37   26.93  5e-07
[1,0]<stdout>:   268435456      67108864   float     sum    16865   15.92   29.84  5e-07    16853   15.93   29.86  5e-07
[1,0]<stdout>:   536870912     134217728   float     sum    32206   16.67   31.26  5e-07    32151   16.70   31.31  5e-07
[1,0]<stdout>:  1073741824     268435456   float     sum    61556   17.44   32.71  5e-07    61303   17.52   32.84  5e-07
[1,0]<stdout>:# Out of bounds values : 0 OK
[1,0]<stdout>:# Avg bus bandwidth    : 7.80081 
```
</details>

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This project is licensed under the Apache-2.0 License.
