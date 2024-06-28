
## AWS EKS Cluster Deployment

The most recent version of the open-source command-line tool written in Go by the team at Weaveworks, `eksctl`, dwarfs the creation of a full-fledged EKS cluster into a one-line wonder: 

```
eksctl create cluster
```

The process takes about 15 minutes. The end result is cloud infrastructure in AWS that includes a VPC, Subnets, a Node Group of Worker Nodes (AmazonLinux EC2 instances), and the cloud resources necessary to run a managed Kubernetes Control Plane in AWS.

![aws-eks-cloud-infrastructure.png](img%2Faws-eks-cloud-infrastructure.png)

## Getting Started

This guide provides a step-by-step procedure to create a Kubernetes cluster in EKS (Elastic Kubernetes Service) using `eksctl`, and a brief verification of the AWS infrastructure established by this automated cluster deployment process. Finally, the guide suggests a cleanup step to delete cloud resources from your AWS account.

### Steps

1. Install CLI Tools
2. Create Access Key
3. Create the Cluster
4. Verify
5. Cleanup
6. Links

### 1. Install CLI Tools

The steps in this guide use 3 command-line interface tools to work with EKS: `aws`, `eksctl`, and `kubectl`.

While `eksctl` is primarily used for cluster management, `kubectl` is the tool to interact with the Kubernetes API for operational tasks.

As a command-line tool for creating and managing EKS clusters, `eksctl` is not for interacting with the resources within the cluster. Instead, it focuses on the creation, configuration, and management of EKS clusters and related AWS infrastructure.

The `aws` command is an all-in-one tool to control and manage AWS services. This guide uses it for authentication and to interface with EKS.

#### Install AWS CLI

On macOS:

Download the .pkg file and run install:

```
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
```

```
sudo installer -pkg AWSCLIV2.pkg -target /
```

On Linux x86:

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
```

```
sudo ./aws/install
```

Check version

```
aws --v
aws-cli/2.15.35 Python/3.11.8 Darwin/23.4.0 exe/x86_64 prompt/off
```

#### Install eksctl

On macOS:

```
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

On Linux:

```
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo mv /tmp/eksctl /usr/local/bin
```

Check version

```
eksctl version
0.175.0
```

#### Install kubectl

On macOS:

```
brew install kubectl
```

On Linux:

Download

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Install

```
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Check version

```
kubectl version --short
```

```
Client Version: v1.26.15-dispatcher
Kustomize Version: v4.5.7
Server Version: v1.30.0-eks-036c24b
```

### 2. Create Access Key

Create a *temporary* access key for the root user in your AWS account. In the AWS Console go to:

IAM > My security credentials > Create Access Key

- Use Case: Command Line Interface (CLI)
- Description: Access key used to interface with AWS CLI

Output:

```
Access Key: ********
Secret access key: *******
```

**Note**: Never store your access key in plain text, in a code repository, or in code.

Run the `aws configure` command to prompt for the credentials and establish an identity on the workstation that will be used to interface with AWS from the command line:

```
aws configure
```

```
AWS Access Key ID [****************2BY5]: <Access Key>
AWS Secret Access Key [****************MKCU]: <Secret access key>
Default region name [us-east-1]: 
Default output format [json]: 
```

Verify the identity is set by running the `aws sts get-caller-identity` command:

```
aws sts get-caller-identity
```

```
{
    "UserId": "<AccountID>",
    "Account": "<AccountID>",
    "Arn": "arn:aws:iam::<AccountID>:root"
}
```

The AWS Security Token Service (STS) returns a verification JSON document with the user details.

**Important**: Delete the temporary access key for the root user after completing the cluster deployment steps.

### 3. Create the Cluster

Without a config file, to create a cluster in your default region, with one managed node group containing two m5.large nodes:

```
eksctl create cluster
```

![eksctl-cluster-create.png](img%2Feksctl-cluster-create.png)

#### Check status

You can check on the status of your cluster creation with:

```
aws eks --region us-east-1 describe-cluster --name <cluster_name> --query cluster.status
```

Output

```
"CREATING"
```

#### Add new Kubectl Context

When the cluster status is “ACTIVE”, you may add a new context to your `~/.kube/config` file with the new cluster details. This enables `kubectl` to work with your cluster in EKS by connecting it to AWS.


```
aws eks --region us-east-1 describe-cluster --name <cluster_name> --query cluster.status
```

Output

```
"ACTIVE"
```

Run the `update-kubeconfig` command:

```
aws eks --region us-east-1 update-kubeconfig --name <cluster_name>
```

Output

```
Added new context <cluster_name> to /Users/alex/.kube/config
```

Confirm the active `kubectl` context in your `.kube/config` file is the new cluster created by `eksctl`:

```
kubectl config get-contexts
CURRENT   NAME                                                           CLUSTER                                       AUTHINFO                                                       NAMESPACE
          docker-desktop                                                 docker-desktop                                docker-desktop                                                 
*         iam-root-account@unique-party-1719505990.us-east-1.eksctl.io   unique-party-1719505990.us-east-1.eksctl.io   iam-root-account@unique-party-1719505990.us-east-1.eksctl.io   
 
```

### 4. Verify

We can now verify the deployment. In this section we review the following cluster resources:

* Kubernetes Services
* Worker Nodes
* Node Resources
* VPC
* Subnets
* Security Groups
* Internet Gateway
* Kubernetes Control Plane
* IAM Role

#### Get Services

The `kubectl get svc` command lists all the services that are currently defined within the active context of your `.kube/config` file.

```
kubectl get svc
```

Output

```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   1m45s
```

#### Get Nodes

To list all the nodes in the cluster:

```
kubectl get nodes -o wide
```

Output

```
NAME                             STATUS   ROLES    AGE   VERSION               INTERNAL-IP      EXTERNAL-IP     OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-192-168-28-229.ec2.internal   Ready    <none>   84m   v1.29.3-eks-ae9a62a   192.168.28.229   35.168.9.109    Amazon Linux 2   5.10.218-208.862.amzn2.x86_64   containerd://1.7.11
ip-192-168-42-138.ec2.internal   Ready    <none>   84m   v1.29.3-eks-ae9a62a   192.168.42.138   3.238.141.182   Amazon Linux 2   5.10.218-208.862.amzn2.x86_64   containerd://1.7.11
```

* Status `Ready` means they are ready to accept pods.
* Role `<none>`, indicates they are worker nodes.

#### Node Resources

To view the specs (CPU, Memory, and Storage) for the worker nodes:

```
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, capacity: .status.capacity, allocatable: .status.allocatable}'
```

Output

```
{
  "name": "ip-192-168-28-229.ec2.internal",
  "capacity": {
    "cpu": "2",
    "ephemeral-storage": "83873772Ki",
    "hugepages-1Gi": "0",
    "hugepages-2Mi": "0",
    "memory": "7824308Ki",
    "pods": "29"
  },
  "allocatable": {
    "cpu": "1930m",
    "ephemeral-storage": "76224326324",
    "hugepages-1Gi": "0",
    "hugepages-2Mi": "0",
    "memory": "7134132Ki",
    "pods": "29"
  }
}
{
  "name": "ip-192-168-42-138.ec2.internal",
  "capacity": {
    "cpu": "2",
    "ephemeral-storage": "83873772Ki",
    "hugepages-1Gi": "0",
    "hugepages-2Mi": "0",
    "memory": "7824308Ki",
    "pods": "29"
  },
  "allocatable": {
    "cpu": "1930m",
    "ephemeral-storage": "76224326324",
    "hugepages-1Gi": "0",
    "hugepages-2Mi": "0",
    "memory": "7134132Ki",
    "pods": "29"
  }
}
```

The output from the 2 commands above indicates that the worker nodes are Amazon Linux 2 EC2 instances, each with the following resources available to allocate to Pods:

- CPUs: 2
- Memory: 6.8 GB
- Storage: 70.98 GB

In the AWS Console go to:

EC2 > Instances > Instances

![aws-ec2-worker-nodes.png](img%2Faws-ec2-worker-nodes.png)

#### VPC

In the AWS Console go to:

VPC > Virtual private cloud > Your VPCs

A Virtual Private Cloud, in the context of an EKS cluster, provides the foundational networking infrastructure that allows you to isolate your cluster's networking from other resources in your AWS account. 

##### Subnets

VPC > Virtual private cloud > Your VPCs > `select vpc` > Resource Map (tab):

![aws-vpc-subnets.png](img%2Faws-vpc-subnets.png)

Within the VPC, subnets are created across multiple Availability Zones (AZs). Worker nodes are deployed into the subnets. 

##### Security Groups

In the AWS Console go to:

VPC > Security > Security groups

![aws-vpc-security-groups.png](img%2Faws-vpc-security-groups.png)

Security groups control inbound and outbound traffic to and from your EKS cluster and its resources. They act as virtual firewalls. For example, the Security group highlighted above allows nodes to communicate with each other on all ports.  

##### Internet Gateway

In the AWS Console go to:

VPC > Virtual private cloud > Internet gateways

![aws-vpc-internet-gateway.png](img%2Faws-vpc-internet-gateway.png)

An Internet Gateway (IGW) enables outbound internet access. Kubernetes clusters connect to the internet to manage the cluster (to download updates, access public APIs), and to access container registries, to download container images, for example, from Docker Hub or Amazon ECR (Elastic Container Registry). 

Many AWS services that EKS relies on are publicly accessible, such as CloudWatch for logging and monitoring, S3 for storage, and IAM for authentication and authorization. An IGW provides the internet connectivity for these services to be used by the cluster. 

##### Kubernetes Cluster

In the AWS Console go to:

EKS > Clusters > `select cluster` > Resources (tab) > Cluster > Nodes

![aws-eks-worker-nodes-joined.png](img%2Faws-eks-worker-nodes-joined.png)

Provides a partial view of:

```
kubectl get nodes -o wide
```

And verifies that the worker nodes are successfully joined to the cluster. 

##### IAM - Node Instance Role

In the AWS Console go to:

IAM > Roles > Search: "NodeInstance" > `select role` 

![aws-iam-eks-node-instance-role.png](img%2Faws-iam-eks-node-instance-role.png)

Has the following Permissions policies:

| IAM Policy                          | Description                                                                                     |
|-------------------------------------|-------------------------------------------------------------------------------------------------|
| AmazonEC2ContainerRegistryReadOnly  | Grants read-only access to Amazon ECR (Elastic Container Registry)                              |
| AmazonEKS_CNI_Policy                | Provides the necessary permissions for the Amazon VPC CNI (Container Network Interface) plugin, to manage the network interfaces and IP addresses required for Kubernetes pods |
| AmazonEKSWorkerNodePolicy           | Allows worker nodes to communicate with the EKS control plane                                   |
| AmazonSSMManagedInstanceCore        | Provides the necessary permissions for the AWS Systems Manager (SSM) to manage the instances    |


This IAM Role is mapped into a Kubernetes ConfigMap. Note that the Role ARN is referenced by a specific ConfigMap.  

In the AWS Console go to:

EKS > Clusters > `select cluster` > Config and secrets > ConfigMaps > `aws-auth`

![aws-eks-k8s-configmap.png](img%2Faws-eks-k8s-configmap.png)

The `aws-auth` ConfigMap plays a crucial role in granting necessary permissions to the worker nodes (EC2 instances) within the cluster. This ConfigMap maps an AWS IAM role to Kubernetes RBAC, allowing the nodes to interact with the EKS control plane and other AWS services, through the Permissions policies listed above. 

##### Summary

The Nodes, VPC, Subnets, Security Groups, Internet Gateway, the Kubernetes cluster itself (the Control Plane), and IAM Role, are the primary AWS infrastructure components that `eksctl` creates. 

This guide is intended to step through the process and to review the end result of cluster creation through `eksctl` without a config file. 

### 5. Cleanup

To delete the cluster, nodes, and other AWS resources created during the deployment process, and to prevent billing charges on your AWS account:

In the AWS Console go to:

1. CloudFormation > Stacks > select `Description: EKS Managed Nodes ...` > Delete
2. CloudFormation > Stacks > select `Description: EKS cluster ...` > Delete

This issues a delete operation to the cluster's CloudFormation stack. The status will change to `DELETE_IN_PROGRESS` until all the resources associated with the CloudFormation stack are removed from your AWS account. A long-running process, best to delete CloudFormation stacks from the AWS Console, to avoid timeouts from the command line. 


#### Delete Access Key

Access keys associated with the `root` user in your AWS account are way too permissive to keep around long term. Be sure to clean up the _temporary_ access key you created at the start of this procedure with:

In the AWS Console go to:

- IAM > My security credentials > Access Keys > `select key`
- Actions > Delete

### 6. Links

User Guide

https://eksctl.io/usage/creating-and-managing-clusters/

GitHub 

https://github.com/eksctl-io/eksctl
