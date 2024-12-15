---
title: Terraform Remote State Storage with AWS S3 and DynamnoDB
description: Implement Terraform Remote State Storage with AWS S3 and DynamnoDB
---

## Step-01: Introduction
- Understand Terraform Backends
- Understand about Remote State Storage and its advantages
- This state is stored by default in a local file named "terraform.tfstate", but it can also be stored remotely, which works better in a team environment.
- Create AWS S3 bucket to store `terraform.tfstate` file and enable backend configurations in terraform settings block
- Understand about **State Locking** and its advantages
- Create DynamoDB Table and  implement State Locking by enabling the same in Terraform backend configuration

[![Image](https://stacksimplify.com/course-images/terraform-remote-state-storage-1.png "Terraform on AWS EKS")](https://stacksimplify.com/course-images/terraform-remote-state-storage-1.png)

[![Image](https://stacksimplify.com/course-images/terraform-remote-state-storage-2.png "Terraform on AWS EKS")](https://stacksimplify.com/course-images/terraform-remote-state-storage-2.png)

[![Image](https://stacksimplify.com/course-images/terraform-remote-state-storage-3.png "Terraform on AWS EKS")](https://stacksimplify.com/course-images/terraform-remote-state-storage-3.png)

[![Image](https://stacksimplify.com/course-images/terraform-remote-state-storage-4.png "Terraform on AWS EKS")](https://stacksimplify.com/course-images/terraform-remote-state-storage-4.png)

[![Image](https://stacksimplify.com/course-images/terraform-remote-state-storage-5.png "Terraform on AWS EKS")](https://stacksimplify.com/course-images/terraform-remote-state-storage-5.png)

[![Image](https://stacksimplify.com/course-images/terraform-remote-state-storage-6.png "Terraform on AWS EKS")](https://stacksimplify.com/course-images/terraform-remote-state-storage-6.png)

## Pre-requisite Step

## Step-00: Introduction 
1. For VPC switch Availability Zones from Static to Dynamic using Datasource `aws_availability_zones`
2. Create EC2 Key pair that will be used for connecting to Bastion Host and EKS Node Group EC2 VM Instances
3. EC2 Bastion Host - [Terraform Input Variables](https://www.terraform.io/docs/language/values/variables.html)
4. EC2 Bastion Host - [AWS Security Group Terraform Module](https://registry.terraform.io/modules/terraform-aws-modules/security-group/aws/latest)
5. EC2 Bastion Host - [AWS AMI Datasource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ami) (Dynamically lookup the latest Amazon2 Linux AMI)
6. EC2 Bastion Host - [AWS EC2 Instance Terraform Module](https://registry.terraform.io/modules/terraform-aws-modules/ec2-instance/aws/latest)
7. EC2 Bastion Host - [Terraform Resource AWS EC2 Elastic IP](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eip)
8. EC2 Bastion Host - [Terraform Provisioners](https://www.terraform.io/docs/language/resources/provisioners/syntax.html)
   - [File provisioner](https://www.terraform.io/docs/language/resources/provisioners/file.html)
   - [remote-exec provisioner](https://www.terraform.io/docs/language/resources/provisioners/local-exec.html)
   - [local-exec provisioner](https://www.terraform.io/docs/language/resources/provisioners/remote-exec.html)
9. EC2 Bastion Host - [Output Values](https://www.terraform.io/docs/language/values/outputs.html)
10. EC2 Bastion Host - ec2bastion.auto.tfvars
11. EKS Input Variables 
12. EKS [Local Values](https://www.terraform.io/docs/language/values/locals.html)
13. EKS Tags in VPC for Public and Private Subnets
14. Execute Terraform Commands and Test
15. Elastic IP - [depends_on Meta Argument](https://www.terraform.io/docs/language/meta-arguments/depends_on.html)

## Step-01: For VPC switch Availability Zones from Static to Dynamic
- [Datasource: aws_availability_zones](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones)
- **File Name:** `vpc-module.tf` for changes 1 and 2
```t
# Change-1: Add Datasource named aws_availability_zones
# AWS Availability Zones Datasource  
data "aws_availability_zones" "available" {
}

# Change-2: Update the same in VPC Module
  azs             = data.aws_availability_zones.available.names

# Change-3: Comment vpc_availability_zones variable in File: vpc-variables.tf
/*
variable "vpc_availability_zones" {
  description = "VPC Availability Zones"
  type = list(string)
  default = ["us-east-1a", "us-east-1b"]
}
*/

# Change-4: Comment hard-coded Availability Zones variable in File: vpc.auto.tfvars 
#vpc_availability_zones = ["us-east-1a", "us-east-1b"]  
```

## Step-02: Create EC2 Key pair and save it
- Go to Services -> EC2 -> Network & Security -> Key Pairs -> Create Key Pair
- **Name:** eks-terraform-key
- **Key Pair Type:** RSA (leave to defaults)
- **Private key file format:** .pem
- Click on **Create key pair**
- COPY the downloaded key pair to `terraform-manifests/private-key` folder
- Provide permissions as `chmod 400 keypair-name`
```t
# Provider Permissions to EC2 Key Pair
cd terraform-manifests/private-key
chmod 400 eks-terraform-key.pem
```
## Step-03: c4-01-ec2bastion-variables.tf
```t
# AWS EC2 Instance Terraform Variables

# AWS EC2 Instance Type
variable "instance_type" {
  description = "EC2 Instance Type"
  type = string
  default = "t3.micro"  
}

# AWS EC2 Instance Key Pair
variable "instance_keypair" {
  description = "AWS EC2 Key pair that need to be associated with EC2 Instance"
  type = string
  default = "eks-terraform-key"
}
```
## Step-04: c4-03-ec2bastion-securitygroups.tf
```t
# AWS EC2 Security Group Terraform Module
# Security Group for Public Bastion Host
module "public_bastion_sg" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "4.5.0"

  name = "${local.name}-public-bastion-sg"
  description = "Security Group with SSH port open for everybody (IPv4 CIDR), egress ports are all world open"
  vpc_id = module.vpc.vpc_id
  # Ingress Rules & CIDR Blocks
  ingress_rules = ["ssh-tcp"]
  ingress_cidr_blocks = ["0.0.0.0/0"]
  # Egress Rule - all-all open
  egress_rules = ["all-all"]
  tags = local.common_tags
}
```

## Step-05: c4-04-ami-datasource.tf
```t
# Get latest AMI ID for Amazon Linux2 OS
data "aws_ami" "amzlinux2" {
  most_recent = true
  owners = [ "amazon" ]
  filter {
    name = "name"
    values = [ "amzn2-ami-hvm-*-gp2" ]
  }
  filter {
    name = "root-device-type"
    values = [ "ebs" ]
  }
  filter {
    name = "virtualization-type"
    values = [ "hvm" ]
  }
  filter {
    name = "architecture"
    values = [ "x86_64" ]
  }
}
```

## Step-06: ec2bastion-instance.tf
```t
# AWS EC2 Instance Terraform Module
# Bastion Host - EC2 Instance that will be created in VPC Public Subnet
module "ec2_public" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "3.3.0"
  # insert the required variables here
  name                   = "${local.name}-BastionHost"
  ami                    = data.aws_ami.amzlinux2.id
  instance_type          = var.instance_type
  key_name               = var.instance_keypair
  #monitoring             = true
  subnet_id              = module.vpc.public_subnets[0]
  vpc_security_group_ids = [module.public_bastion_sg.security_group_id]
  tags = local.common_tags
}
```

## Step-07: ec2bastion-elasticip.tf
```t
# Create Elastic IP for Bastion Host
# Resource - depends_on Meta-Argument
resource "aws_eip" "bastion_eip" {
  depends_on = [ module.ec2_public, module.vpc ]
  instance = module.ec2_public.id
  vpc      = true
  tags = local.common_tags
}
```
## Step-08: ec2bastion-provisioners.tf
```t
# Create a Null Resource and Provisioners
resource "null_resource" "copy_ec2_keys" {
  depends_on = [module.ec2_public]
  # Connection Block for Provisioners to connect to EC2 Instance
  connection {
    type     = "ssh"
    host     = aws_eip.bastion_eip.public_ip    
    user     = "ec2-user"
    password = ""
    private_key = file("private-key/eks-terraform-key.pem")
  }  

## File Provisioner: Copies the terraform-key.pem file to /tmp/terraform-key.pem
  provisioner "file" {
    source      = "private-key/eks-terraform-key.pem"
    destination = "/tmp/eks-terraform-key.pem"
  }
## Remote Exec Provisioner: Using remote-exec provisioner fix the private key permissions on Bastion Host
  provisioner "remote-exec" {
    inline = [
      "sudo chmod 400 /tmp/eks-terraform-key.pem"
    ]
  }
## Local Exec Provisioner:  local-exec provisioner (Creation-Time Provisioner - Triggered during Create Resource)
  provisioner "local-exec" {
    command = "echo VPC created on `date` and VPC ID: ${module.vpc.vpc_id} >> creation-time-vpc-id.txt"
    working_dir = "local-exec-output-files/"
    #on_failure = continue
  }

}
```

## Step-09: ec2bastion.auto.tfvars
```t
instance_type = "t3.micro"
instance_keypair = "eks-terraform-key"
```

## Step-10: ec2bastion-outputs.tf
```t
# AWS EC2 Instance Terraform Outputs
# Public EC2 Instances - Bastion Host

## ec2_bastion_public_instance_ids
output "ec2_bastion_public_instance_ids" {
  description = "List of IDs of instances"
  value       = module.ec2_public.id
}

## ec2_bastion_public_ip
output "ec2_bastion_eip" {
  description = "Elastic IP associated to the Bastion Host"
  value       = aws_eip.bastion_eip.public_ip
}

```

## Step-11: eks-variables.tf
```t
# EKS Cluster Input Variables
variable "cluster_name" {
  description = "Name of the EKS cluster. Also used as a prefix in names of related resources."
  type        = string
  default     = "eksdemo"
}
```



## Step-12: eks.auto.tfvars
```t
cluster_name = "eksdemo1"
```

## Step-13: local-values.tf
```t
# Add additional local value
  eks_cluster_name = "${local.name}-${var.cluster_name}"  
```

## Step-14: vpc-module.tf
- Update VPC Tags to Support EKS
```t
# Create VPC Terraform Module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.11.0"
  #version = "~> 3.11"

  # VPC Basic Details
  name = local.eks_cluster_name
  cidr = var.vpc_cidr_block
  azs             = data.aws_availability_zones.available.names
  public_subnets  = var.vpc_public_subnets
  private_subnets = var.vpc_private_subnets  

  # Database Subnets
  database_subnets = var.vpc_database_subnets
  create_database_subnet_group = var.vpc_create_database_subnet_group
  create_database_subnet_route_table = var.vpc_create_database_subnet_route_table
  # create_database_internet_gateway_route = true
  # create_database_nat_gateway_route = true
  
  # NAT Gateways - Outbound Communication
  enable_nat_gateway = var.vpc_enable_nat_gateway 
  single_nat_gateway = var.vpc_single_nat_gateway

  # VPC DNS Parameters
  enable_dns_hostnames = true
  enable_dns_support   = true

  
  tags = local.common_tags
  vpc_tags = local.common_tags

  # Additional Tags to Subnets
  public_subnet_tags = {
    Type = "Public Subnets"
    "kubernetes.io/role/elb" = 1    
    "kubernetes.io/cluster/${local.eks_cluster_name}" = "shared"        
  }
  private_subnet_tags = {
    Type = "private-subnets"
    "kubernetes.io/role/internal-elb" = 1    
    "kubernetes.io/cluster/${local.eks_cluster_name}" = "shared"    
  }

  database_subnet_tags = {
    Type = "database-subnets"
  }
}
```

## Step-15: Execute Terraform Commands
```t
# Terraform Initialize
terraform init

# Terraform Validate
terraform validate

# Terraform plan
terraform plan

# Terraform Apply
terraform apply -auto-approve
```

## Step-16: Verify the following
1. Verify VPC Tags
2. Verify Bastion EC2 Instance 
3. Verify Bastion EC2 Instance Security Group
4. Connect to Bastion EC2 Instnace
```t
# Connect to Bastion EC2 Instance
ssh -i private-key/eks-terraform-key.pem ec2-user@<Elastic-IP-Bastion-Host>
sudo su -

# Verify File and Remote Exec Provisioners moved the EKS PEM file
cd /tmp
ls -lrta
Observation: We should find the file named "eks-terraform-key.pem" moved from our local desktop to Bastion EC2 Instance "/tmp" folder
```

## Step-02: Create S3 Bucket
- Go to Services -> S3 -> Create Bucket
- **Bucket name:** terraform-on-aws-eks
- **Region:** US-East (N.Virginia)
- **Bucket settings for Block Public Access:** leave to defaults
- **Bucket Versioning:** Enable
- Rest all leave to **defaults**
- Click on **Create Bucket**
- **Create Folder**
  - **Folder Name:** dev
  - Click on **Create Folder**
- **Create Folder**
  - **Folder Name:** dev/eks-cluster
  - Click on **Create Folder**  
- **Create Folder**
  - **Folder Name:** dev/app1k8s
  - Click on **Create Folder**    


## Step-03: EKS Cluster: Terraform Backend Configuration
- **File Location:** `01-ekscluster-terraform-manifests/c1-versions.tf`
- [Terraform Backend as S3](https://www.terraform.io/docs/language/settings/backends/s3.html)
- Add the below listed Terraform backend block in `Terrafrom Settings` block in `c1-versions.tf`
```t
  # Adding Backend as S3 for Remote State Storage
  backend "s3" {
    bucket = "terraform-on-aws-eks"
    key    = "dev/eks-cluster/terraform.tfstate"
    region = "us-east-1" 
 
    # For State Locking
    dynamodb_table = "dev-ekscluster"    
  }  
```

## Step-04: terraform.tfvars
- **File Location:** `01-ekscluster-terraform-manifests/terraform.tfvars`
- Update `environment` to `dev`
```t
# Generic Variables
aws_region = "us-east-1"
environment = "dev"
business_divsion = "hr"
```

## Step-05: Add State Locking Feature using DynamoDB Table
- Understand about Terraform State Locking Advantages
### Step-05-01: EKS Cluster DynamoDB Table
- Create Dynamo DB Table for EKS Cluster
  - **Table Name:** dev-ekscluster
  - **Partition key (Primary Key):** LockID (Type as String)
  - **Table settings:** Use default settings (checked)
  - Click on **Create**
### Step-05-02: App1 Kubernetes DynamoDB Table
- Create Dynamo DB Table for app1k8s
  - **Table Name:** dev-app1k8s
  - **Partition key (Primary Key):** LockID (Type as String)
  - **Table settings:** Use default settings (checked)
  - Click on **Create**


## Step-06: Create EKS Cluster: Execute Terraform Commands
```t
# Change Directory
cd 01-ekscluster-terraform-manifests

# Initialize Terraform 
terraform init
Observation: 
Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

# Terraform Validate
terraform validate

# Review the terraform plan
terraform plan 
Observation: 
1) Below messages displayed at start and end of command
Acquiring state lock. This may take a few moments...
Releasing state lock. This may take a few moments...
2) Verify DynamoDB Table -> Items tab

# Create Resources 
terraform apply -auto-approve

# Verify S3 Bucket for terraform.tfstate file
dev/eks-cluster/terraform.tfstate
Observation: 
1. Finally at this point you should see the terraform.tfstate file in s3 bucket
2. As S3 bucket version is enabled, new versions of `terraform.tfstate` file new versions will be created and tracked if any changes happens to infrastructure using Terraform Configuration Files
```

## Step-07: Kubernetes Resources: Terraform Backend Configuration
- **File Location:** `k8sresources-terraform-manifests/c1-versions.tf`
- Add the below listed Terraform backend block in `Terrafrom Settings` block in `c1-versions.tf`
```t
  # Adding Backend as S3 for Remote State Storage
  backend "s3" {
    bucket = "terraform-on-aws-eks"
    key    = "dev/app1k8s/terraform.tfstate"
    region = "us-east-1" 

    # For State Locking
    dynamodb_table = "dev-app1k8s"    
  }   
```
## Step-08: c2-remote-state-datasource.tf
- **File Location:** `k8sresources-terraform-manifests/c2-remote-state-datasource.tf`
- Update the EKS Cluster Remote State Datasource information
```t
# Terraform Remote State Datasource - Remote Backend AWS S3
data "terraform_remote_state" "eks" {
  backend = "s3"
  config = {
    bucket = "terraform-on-aws-eks"
    key    = "dev/eks-cluster/terraform.tfstate"
    region = "us-east-1" 
  }
}
```


## Step-09: Create Kubernetes Resources: Execute Terraform Commands
```t
# Change Directory
cd 02-k8sresources-terraform-manifests

# Initialize Terraform 
terraform init
Observation: 
Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

# Terraform Validate
terraform validate

# Review the terraform plan
terraform plan 
Observation: 
1) Below messages displayed at start and end of command
Acquiring state lock. This may take a few moments...
Releasing state lock. This may take a few moments...
2) Verify DynamoDB Table -> Items tab

# Create Resources 
terraform apply -auto-approve

# Verify S3 Bucket for terraform.tfstate file
dev/app1k8s/terraform.tfstate
Observation: 
1. Finally at this point you should see the terraform.tfstate file in s3 bucket
2. As S3 bucket version is enabled, new versions of `terraform.tfstate` file new versions will be created and tracked if any changes happens to infrastructure using Terraform Configuration Files
```

## Step-10: Configure kubeconfig for kubectl
```t
# Configure kubeconfig for kubectl
aws eks --region <region-code> update-kubeconfig --name <cluster_name>
aws eks --region us-east-1 update-kubeconfig --name hr-dev-eksdemo1

# List Worker Nodes
kubectl get nodes
kubectl get nodes -o wide
Observation:
1. Verify the External IP for the node

# Verify Services
kubectl get svc
```


## Step-11: Verify Kubernetes Resources
```t
# List Pods
kubectl get pods -o wide
Observation: 
1. Both app pod should be in Public Node Group 

# List Services
kubectl get svc
kubectl get svc -o wide
Observation:
1. We should see both Load Balancer Service and NodePort service created

# Access Sample Application on Browser
http://<LB-DNS-NAME>
http://abb2f2b480148414f824ed3cd843bdf0-805914492.us-east-1.elb.amazonaws.com
```


## Step-12: Verify Kubernetes Resources via AWS Management console
1. Go to Services -> EC2 -> Load Balancing -> Load Balancers
2. Verify Tabs
   - Description: Make a note of LB DNS Name
   - Instances
   - Health Checks
   - Listeners
   - Monitoring


## Step-13: Node Port Service Port - Update Node Security Group
- **Important Note:** This is not a recommended option to update the Node Security group to open ports to internet, but just for learning and testing we are doing this. 
- Go to Services -> Instances -> Find Private Node Group Instance -> Click on Security Tab
- Find the Security Group with name `eks-remoteAccess-`
- Go to the Security Group (Example Name: sg-027936abd2a182f76 - eks-remoteAccess-d6beab70-4407-dbc7-9d1f-80721415bd90)
- Add an additional Inbound Rule
   - **Type:** Custom TCP
   - **Protocol:** TCP
   - **Port range:** 31280
   - **Source:** Anywhere (0.0.0.0/0)
   - **Description:** NodePort Rule
- Click on **Save rules**

## Step-14: Verify by accessing the Sample Application using NodePort Service
```t
# List Nodes
kubectl get nodes -o wide
Observation: Make a note of the Node External IP

# List Services
kubectl get svc
Observation: Make a note of the NodePort service port "myapp1-nodeport-service" which looks as "80:31280/TCP"

# Access the Sample Application in Browser
http://<EXTERNAL-IP-OF-NODE>:<NODE-PORT>
http://54.165.248.51:31280
```

## Step-15: Remove Inbound Rule added 
- Go to Services -> Instances -> Find Private Node Group Instance -> Click on Security Tab
- Find the Security Group with name `eks-remoteAccess-`
- Go to the Security Group (Example Name: sg-027936abd2a182f76 - eks-remoteAccess-d6beab70-4407-dbc7-9d1f-80721415bd90)
- Remove the NodePort Rule which we added.

## Step-16: Clean-Up
```t
# Delete Kubernetes  Resources
cd 02-k8sresources-terraform-manifests
terraform apply -destroy -auto-approve
rm -rf .terraform*

# Verify Kubernetes Resources
kubectl get pods
kubectl get svc

# Delete EKS Cluster (Optional)
1. As we are using the EKS Cluster with Remote state storage, we can and we will reuse EKS Cluster in next sections
2. Dont delete or destroy EKS Cluster Resources

cd 01-ekscluster-terraform-manifests/
terraform apply -destroy -auto-approve
rm -rf .terraform*
```

## Additional Reference
## Step-00: Little bit theory about Terraform Backends
- Understand little bit more about Terraform Backends
- Where and when Terraform Backends are used ?
- What Terraform backends do ?
- How many types of Terraform backends exists as on today ? 

[![Image](https://stacksimplify.com/course-images/terraform-remote-state-storage-7.png "Terraform on AWS with IAC DevOps and SRE")](https://stacksimplify.com/course-images/terraform-remote-state-storage-7.png)

[![Image](https://stacksimplify.com/course-images/terraform-remote-state-storage-8.png "Terraform on AWS with IAC DevOps and SRE")](https://stacksimplify.com/course-images/terraform-remote-state-storage-8.png)

[![Image](https://stacksimplify.com/course-images/terraform-remote-state-storage-9.png "Terraform on AWS with IAC DevOps and SRE")](https://stacksimplify.com/course-images/terraform-remote-state-storage-9.png)


## References 
- [AWS S3 Backend](https://www.terraform.io/docs/language/settings/backends/s3.html)
- [Terraform Backends](https://www.terraform.io/docs/language/settings/backends/index.html)
- [Terraform State Storage](https://www.terraform.io/docs/language/state/backends.html)
- [Terraform State Locking](https://www.terraform.io/docs/language/state/locking.html)
- [Remote Backends - Enhanced](https://www.terraform.io/docs/language/settings/backends/remote.html)

