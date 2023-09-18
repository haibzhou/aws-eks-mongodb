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


