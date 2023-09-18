# Accelerate Application Modernization with Amazon EKS, and MongoDB Atlas

## Introduction

This is a technical repo to demonstrate the application deployment using MongoDB Atlas and AWS EKS. This tuotorial is intended for those who wants to

1. Serverless Application Deployment for Production Environment
2. Production deployment to auto scale, HA and Security
3. Agile development of application moderinzation
4. Deployment of containerized application in AWS
5. Want to try out the AWS EKS and MongoDB Atlas

## [MongoDB Atlas](https://www.mongodb.com/atlas)
MongoDB Atlas is an all purpose database having features like Document Model, Geo-spatial , Time-seires, hybrid deployment, multi cloud services. It evolved as "Developer Data Platform", intended to reduce the developers workload on development and management the database environment. It also provide a free tier to test out the application / database features.

## [AWS EKS](https://aws.amazon.com/eks/)
Amazon Elastic Kubernetes Service (Amazon EKS) is a managed Kubernetes service to run Kubernetes in the AWS cloud and on-premises data centers. In the cloud, Amazon EKS automatically manages the availability and scalability of the Kubernetes control plane nodes responsible for scheduling containers, managing application availability, storing cluster data, and other key tasks. With Amazon EKS, you can take advantage of all the performance, scale, reliability, and availability of AWS infrastructure, as well as integrations with AWS networking and security services. In this lab, you'll learn how to deploy a sample dockerized MEAN (MongoDB, Express, Angular, Node) stack application on a scalable and secure AWS infrastructure using Amazon EKS.

## Architecure Diagram
![image](https://github.com/haibzhou/aws-eks-mongodb/assets/109695471/6683c366-a59b-47bd-9ccd-047fdb3a3a9d)

# Step by Step EKS Deployment
## Step1: Set up the MongoDB Atlas cluster
MongoDB Atlas provides a free cluster setup. Pls follow the link to setup the [free cluster](https://www.mongodb.com/docs/atlas/getting-started/)

## Step2: Crerate new IAM user
Login from Isengard. Create new username and password. 
User Name: eks-user
This user must have AdministratorAccess permission. 
 
Go to IAM and select Users on left panel, then select the user name you just created. Select Security credentials. At Access keys section, click Create access key button.
 
 
 
Download the CSV file to save the access key and secret access key.

## Step3: Use CloudShell to deploy K8 VPC eks-bastion EC2 instance
Login to AWS console with username and password created in previous step.
This lab is designed to work in us-east-1 region. Make sure you move to us-east-1 region for all operations.

Open AWS Console and click the CloudShell icon. CloudShell provide an environment that allow you to run AWS CLI command.
![image](https://github.com/haibzhou/aws-eks-mongodb/assets/109695471/ce8e5a30-cdd8-438c-a457-2d6107402f8c)


Create directory for ec2-user
```
cd /home
sudo mkdir ec2-user
sudo chown cloudshell-user ec2-user
sudo chgrp cloudshell-user ec2-user
cd ec2-user
mkdir environment
cd environment
```
Install Terraform to deploy the VPC network architecture
```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum install terraform -y
```
Verify terraform is installed
```

terraform version
```
```
Terraform v1.5.6
on linux_amd64
```
We will use terraform to create the VPC, subnet, NAT gateway.

Checkout the terraform networking module from github
```
cd /home/ec2-user/environment

git clone https://github.com/haibzhou/terraform

```

```
Cloning into 'terraform'...
remote: Enumerating objects: 27, done.
remote: Counting objects: 100% (27/27), done.
remote: Compressing objects: 100% (15/15), done.
remote: Total 27 (delta 9), reused 27 (delta 9), pack-reused 0
Receiving objects: 100% (27/27), 15.46 KiB | 2.58 MiB/s, done.
Resolving deltas: 100% (9/9), done.

```

Create the necessary parameters to terraform.tfvars that we need to create the VPC networking architecture that will be used for EKS.
```
/home/ec2-user /environment $ cd terraform
cat > terraform.tfvars <<EOF
//AWS 
region      = "us-east-1"
environment = "k8s"

/* module networking */
vpc_cidr             = "10.0.0.0/16"
public_subnets_cidr  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"] //List of Public subnet cidr range
private_subnets_cidr = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"] //List of private subnet cidr range
EOF

```
Run following commands to create VPC, subnet, IGW and NAT gateway.
```
terraform init

terraform plan

terraform apply

```
Check VPC named k8-vpc is created
![image](https://github.com/haibzhou/aws-eks-mongodb/assets/109695471/147bfe1f-0a70-4eec-9880-f0ac0e19041f)

Deploy EC2 instance eks-bastion in k8s-vpc. We will use this EC2 instance for the rest of this lab.
```

export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=k8s-vpc" |jq -r .Vpcs[].VpcId )

aws ec2 create-security-group \
    --group-name eke-bastion-sg \
    --description "AWS ec2 CLI Demo SG" \
    --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=eks-bastion-sg}]' \
    --vpc-id ${VPC_ID}


export SG_ID=$(aws ec2 describe-security-groups     --filters Name=tag:Name,Values=eks-bastion-sg     --query "SecurityGroups[*].{ID:GroupId}" |jq -r .[0].ID)

aws ec2 authorize-security-group-ingress --group-id ${SG_ID} --protocol tcp --port 22 --cidr 0.0.0.0/0


export PUBLIC_SUBNETS=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1a-public-subnet" | jq -r .Subnets[].SubnetId)


aws ec2 run-instances \
    --image-id ami-04cb4ca688797756f \
    --count 1 \
    --instance-type t2.micro \
    --security-group-ids ${SG_ID} \
    --subnet-id ${PUBLIC_SUBNETS} \
    --block-device-mappings "[{\"DeviceName\":\"/dev/sdf\",\"Ebs\":{\"VolumeSize\":30,\"DeleteOnTermination\":false}}]" \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=eks-bastion}]' 'ResourceType=volume,Tags=[{Key=Name,Value=eks-baston-disk}]'


```
Go to AWS console to verify the EC2 instance is created and wait for the instance state to become Running.

# The rest of the workshop will work from the eks-bastion ec2 instance. 

## Step4: Create EKS Cluster
Open AWS console and connect to this eks-bastion EC2 instance.
Configure the AWS Access Key and Security Key on eks-bastion EC2 instance.
EKS deployment will take about 20 minutes. The ssh session will be expired if no keys are entered. We need to configure the keep alive packet to keep the session alive.

```
cat >~/.ssh/config <<EOF
 Host *
  ServerAliveInterval 50
  ServerAliveCountMax 3
EOF
```
Configure AWS credential for AWS CLI
```
aws configure
AWS Access Key ID [****************K262]: 
AWS Secret Access Key [****************C48w]: 
Default region name [us-west-2]: us-east-1
Default output format [None]:

```

Install eksctl for creating and managing your EKS cluster
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```
Verify eksctl is installed.
```
eksctl version
```
```
0.155.0
```

Prepare the cluster configuration file
First we need to grab the privete subnet id of our terraform generated VPC resources, so that we can deploy our EKS cluster to the VPC topology we just created. We should be able to achieve this by using jq command to parse the terraform output command.

```

# export VPC
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=k8s-vpc" |jq -r .Vpcs[].VpcId)

# export public subnets
export PRIVATE_SUBNETS_ID_A=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1a-private-subnet" | jq -r .Subnets[].SubnetId)
export PRIVATE_SUBNETS_ID_B=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1b-private-subnet" | jq -r .Subnets[].SubnetId)
export PRIVATE_SUBNETS_ID_C=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1c-private-subnet" | jq -r .Subnets[].SubnetId)

```

Create an EKS cluster configuration file
We shall create an EKS configuration file called 'eks-cluster.yaml` to pass the corresponding parameters to provision the cluster in the VPC we just created.

```
mkdir /home/ec2-user/environment

cat > /home/ec2-user/environment/eks-cluster.yaml <<EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: web-host-on-eks
  region: us-east-1 # Specify the aws region
  version: "1.27"
privateCluster: # Allows configuring a fully-private cluster in which no node has outbound internet access, and private access to AWS services is enabled via VPC endpoints
  enabled: false
vpc:
  id: "${VPC_ID}" # Specify the VPC_ID to the eksctl command
  subnets: # Creating the EKS master nodes to a completely private environment
    private:
      private-us-east-1a: 
        id: "${PRIVATE_SUBNETS_ID_A}"
      private-us-east-1b:
        id: "${PRIVATE_SUBNETS_ID_B}"
      private-us-east-1c:
        id: "${PRIVATE_SUBNETS_ID_C}"
managedNodeGroups: # Create a managed node group in private subnets
- name: managed
  labels:
    role: worker
  instanceType: t3.small
  minSize: 3
  desiredCapacity: 3
  maxSize: 10
  privateNetworking: true
  volumeSize: 50
  volumeType: gp2
  volumeEncrypted: true
  iam:
    withAddonPolicies: 
      autoScaler: true # enables IAM policy for cluster-autoscaler
      albIngress: true 
      cloudWatch: true 
  # securityGroups:
  #    attachIDs: ["sg-1", "sg-2"]
  ssh:
      allow: true
      publicKeyPath: ~/.ssh/id_rsa.pub
      # new feature for restricting SSH access to certain AWS security group IDs
  subnets:
    - private-us-east-1a
    - private-us-east-1b
    - private-us-east-1c
cloudWatch:
  clusterLogging:
    # enable specific types of cluster control plane logs
    enableTypes: ["all"]
    # all supported types: "api", "audit", "authenticator", "controllerManager", "scheduler"
    # supported special values: "*" and "all"
addons: # explore more on doc about EKS addons: https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html
- name: vpc-cni # no version is specified so it deploys the default version
  attachPolicyARNs:
    - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
- name: coredns
  version: latest # auto discovers the latest available
- name: kube-proxy
  version: latest
iam:
  withOIDC: true # Enable OIDC identity provider for plugins, explore more on doc: https://docs.aws.amazon.com/eks/latest/userguide/authenticate-oidc-identity-provider.html
  serviceAccounts: # create k8s service accounts(https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/) and associate with IAM policy, see more on: https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html
  - metadata:
      name: aws-load-balancer-controller # create the needed service account for aws-load-balancer-controller while provisioning ALB/ELB by k8s ingress api
      namespace: kube-system
    wellKnownPolicies:
      awsLoadBalancerController: true
  - metadata:
      name: cluster-autoscaler # create the CA needed service account and its IAM policy
      namespace: kube-system
      labels: {aws-usage: "cluster-ops"}
    wellKnownPolicies:
      autoScaler: true
EOF
```
Before running the eksctl command, we need to generate the default ssh key in your EC2 instance.
```
ssh-keygen
```
Please press enter to keep all the input value as default. Next we shall run the eksctl command to create our cluster by using the yaml configuration file we just created.
```
eksctl create cluster -f /home/ec2-user/environment/eks-cluster.yaml
```
During the waiting time, you can navigate to AWS Console, and check the CloudFormation service page, you should be able to check the status of the CloudFormation stack is in progress. The stack name should be called eksctl-web-host-on-eks-cluster. This will take about 20 minutes.

![image](https://github.com/haibzhou/aws-eks-mongodb/assets/109695471/df927aee-6f41-49f5-a3a3-10cf5fbbd168)

Access the EKS Console
After successfully creating the EKS cluster, you should be able to navigate to the EKS cluster detail  page to explore different tabs for you cluster.

![image](https://github.com/haibzhou/aws-eks-mongodb/assets/109695471/7b472273-42a7-4e09-9bdd-0a037f5dfdca)



Access your EKS cluster

Install kubectl to manage your cluster.
```
sudo bash -c "cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF"
```
```
sudo yum install -y kubectl
```
```
kubectl version –client
```
```
Client Version: v1.28.1
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```




   



