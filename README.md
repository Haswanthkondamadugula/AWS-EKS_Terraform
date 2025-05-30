# 🚀 **Provision Amazon EKS Cluster on AWS using Terraform**  

![image](https://github.com/user-attachments/assets/cc781501-696a-410b-be4c-6563673f7bfa)
 

## 🔥 **Introduction**  

### 🟢 What is Amazon EKS?  

Amazon Elastic Kubernetes Service (**Amazon EKS**) is a **fully managed** Kubernetes service that simplifies deploying, managing, and scaling containerized applications on AWS.  

### 🟢 What is Terraform?  

Terraform is an **open-source Infrastructure as Code (IaC) tool** that enables declarative provisioning and management of cloud infrastructure.  

---

## ✅ **Prerequisites**  

Before proceeding, ensure you have the following:  

- **AWS Account** (Free Tier Available)  
- **AWS CLI** ([Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))  
- **Terraform** ([Install Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli))  
- **Kubectl** ([Install Kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html))  
- **VS Code** (Recommended IDE)  

---

## 📂 **Project Structure**  

```bash
📦 eks-cluster-terraform
├── 📜 provider.tf               # AWS Provider Configuration
├── 📜 eks-backend-terra.tf       # S3 Backend for Terraform State
├── 📜 vpc.tf                     # VPC Configuration
├── 📜 subnets.tf                 # Public Subnets
├── 📜 internetgw.tf              # Internet Gateway
├── 📜 route.tf                   # Route Table
├── 📜 sg.tf                      # Security Groups
├── 📜 iam_role.tf                # IAM Roles & Policies
├── 📜 eks_cluster.tf             # EKS Cluster Configuration
├── 📜 eks_node_group.tf          # EKS Worker Nodes
└── 📜 README.md                  # Project Documentation
```  

---

## 📚 **References**  

- **[Terraform Registry](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs)**  
- **[AWS CLI Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)**  
- **[EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)**  

🔗 _This project follows AWS best practices for EKS deployment using Terraform._  

---

_**Warning!!! You might incur costs in your AWS account by doing this**_

## 📌 **Getting Started**

Head to your AWS account and navigate to the S3 section. This is where we will store our **tf.state file** as it is considered best practice to keep our state file in a Remote Location. The primary purpose of Terraform state is to store bindings between objects in a remote system and resources declared in your configuration and I am going to keep mine in Amazon S3.

Create an S3 Bucket. You can choose whatever name suits your purpose, I am naming mine `terra-eks-backend`. You can provision your bucket in any region that suits your purpose but I am provisioning mine in the `us-east-1`.

![aws-eks](https://imgur.com/r1hBcxO.png)

It is highly recommended that you enable [Bucket Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/manage-versioning-examples.html) on the S3 bucket to allow for state recovery in the case of accidental deletions and human error.

![aws-eks](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*r_Qbo1YHAPMfcjjFyQJOAw.png)

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*CtasAJc5N0WaX-P1y3CDnQ.png)

To make my `tf.state file` consistent I am going to enable State-Locking. File locking is a data management feature that allows only one user or process access to a file at any given time. It restricts other users from changing the file while it is being used by another user or process. In other to achieve this I need to create a Dynamo DB Table so let's head to Dynamo DB in our AWS account and create a Dynamo DB table

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*--_Q2-Jr_OLP4kGCjvapbA.png)

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*rw6mSFdFnIM-LBWdNFxkLQ.png)

You can choose any name for your DynamoDB Table but it is important to note that the partition key is case sensitive. I named the Partition Key LOCKID because this will enable Dynamo DB to lock and release the file. Scroll down the page and click on Create Table

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*QJZKwa4UpaqGzkl5Y5NE2w.png)

### **CREATE TERRAFORM FILES**

**eks-backend-terra.tf**

```hashicorp
terraform {
  backend "s3" {
    bucket         = "your-s3-bucket-name"
    key            = "terraform/state/eks-cluster.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "your-dynamodb-table-name"
  }
}
```

**Provider.tf**

Name of provider AWS

Our source will be defined as hashicorp/aws. This is a short abbreviation for ```registry.terraform.io/hashicorp/aws```

The version is set to ~>5.0

Region is us-east-1

```hashicorp
terraform {
  required_version = ">= 1.3.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

**Vpc.tf**

```hashicorp
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "pc-vpc"
    Environment = "dev"
  }
}
```

The vpc.tf contains codes to create a new Vpc. The CIDR block is 10.0.0.0/16 and I have tagged the name of the VPC as PC-VPC

**subnets.tf**

```hashicorp
resource "aws_subnet" "public_1" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name        = "public-sub-1"
    Environment = "dev"
  }
}

resource "aws_subnet" "public_2" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "us-east-1b"
  map_public_ip_on_launch = true

  tags = {
    Name        = "public-sub-2"
    Environment = "dev"
  }
}

resource "aws_subnet" "private_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.3.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name        = "private-sub-1"
    Environment = "dev"
  }
}

resource "aws_subnet" "private_2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.4.0/24"
  availability_zone = "us-east-1b"

  tags = {
    Name        = "private-sub-2"
    Environment = "dev"
  }
}
```

EKS requires a minimum of two subnets to function so this is creating two public subnets in two availability zones namely us-east-1a and us-east-1b respectively. I have also set _map public IP on launch = true_. This will assign our subnets with public IPV4 addresses. I have given the CIDR range of 10.0.1.0/24 to public-1 and 10.0.2.0/24 to public-2.

**Internetgw.tf**

```hashicorp
resource "aws_internet_gateway" "main_gw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = "main-gateway"
    Environment = "dev"
  }
}
```

This will create and attach the internet gateway to the Vpc created

**Route.tf**

```hashicorp
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_gw.id
  }

  tags = {
    Name        = "public-route-table"
    Environment = "dev"
  }
}

resource "aws_route_table_association" "public_subnet_1_assoc" {
  subnet_id      = aws_subnet.public-1.id
  route_table_id = aws_route_table.public_rt.id
}

resource "aws_route_table_association" "public_subnet_2_assoc" {
  subnet_id      = aws_subnet.public-2.id
  route_table_id = aws_route_table.public_rt.id
}
```

This will create the Route table. The route table has been associated with the two subnets (public-1,public-2). An internet gateway id has also been associated with the Route table and the Vpc id has been assigned to the Route table.

**Sg.tf**

```hashicorp
resource "aws_security_group" "eks_sg" {
  name        = "eks-cluster-sg"
  description = "Security group for EKS cluster"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "Allow SSH access"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow HTTPS traffic"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow HTTP traffic"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "eks-cluster-sg"
    Environment = "dev"
  }
}
```

This will create the security group attached to the created Vpc with both ingress and egress rules. Ingress rules allow incoming(inbound) connection with our Vpc while egress rule allows outgoing (outbound) connection.

**iam_role.tf**

```hashicorp
resource "aws_iam_role" "eks_cluster_role" {
  name = "eks-cluster-role"

  assume_role_policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
POLICY
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster_role.name
}

resource "aws_iam_role_policy_attachment" "eks_service_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSServicePolicy"
  role       = aws_iam_role.eks_cluster_role.name
}

resource "aws_iam_role_policy_attachment" "eks_vpc_resource_controller" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSVPCResourceController"
  role       = aws_iam_role.eks_cluster_role.name
}

resource "aws_iam_role" "eks_worker_role" {
  name = "eks-worker-role"

  assume_role_policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
POLICY
}

resource "aws_iam_policy" "eks_autoscaler_policy" {
  name   = "eks-autoscaler-policy"
  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeTags",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "eks_worker_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_worker_role.name
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.eks_worker_role.name
}

resource "aws_iam_role_policy_attachment" "ssm_managed_instance" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
  role       = aws_iam_role.eks_worker_role.name
}

resource "aws_iam_role_policy_attachment" "ecr_readonly" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.eks_worker_role.name
}

resource "aws_iam_role_policy_attachment" "xray_daemon_write" {
  policy_arn = "arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess"
  role       = aws_iam_role.eks_worker_role.name
}

resource "aws_iam_role_policy_attachment" "s3_readonly" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
  role       = aws_iam_role.eks_worker_role.name
}

resource "aws_iam_role_policy_attachment" "autoscaler_attachment" {
  policy_arn = aws_iam_policy.eks_autoscaler_policy.arn
  role       = aws_iam_role.eks_worker_role.name
}

resource "aws_iam_instance_profile" "eks_worker_profile" {
  depends_on = [aws_iam_role.eks_worker_role]
  name       = "eks-worker-profile"
  role       = aws_iam_role.eks_worker_role.name
}
```

This will create all the necessary IAM roles and Policies for the EKS cluster. It will be attached to the EKS cluster once the EKS Cluster is created. This will enable the EKS Cluster to have all the necessary permissions needed.

**eks_cluster.tf**

```hashicorp
resource "aws_eks_cluster" "eks" {
  name     = "pc-eks-cluster"
  role_arn = aws_iam_role.eks_cluster_role.arn

  vpc_config {
    subnet_ids = [aws_subnet.public-1.id, aws_subnet.public-2.id]
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy,
    aws_iam_role_policy_attachment.eks_service_policy,
    aws_iam_role_policy_attachment.eks_vpc_resource_controller
  ]
}
```

This will create the EKS Cluster. depends_on =[ means that the EKS Cluster being created depends on the completion of the creation of the IAM roles. The two subnet IDs have also been mentioned here as well.

**eks_node_group.tf**

```hashicorp
resource "aws_instance" "kubectl_server" {
  ami                         = "ami-06ca3ca175f37dd66"
  key_name                    = "EKS_KEY_PAIR"
  instance_type               = "t2.micro"
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.public-1.id
  vpc_security_group_ids      = [aws_security_group.allow_tls.id]

  tags = {
    Name = "kubectl-server"
  }
}

resource "aws_eks_node_group" "eks_node_group" {
  cluster_name    = aws_eks_cluster.eks.name
  node_group_name = "pc-node-group"
  node_role_arn   = aws_iam_role.eks_worker_role.arn
  subnet_ids      = [aws_subnet.public-1.id, aws_subnet.public-2.id]
  capacity_type   = "ON_DEMAND"
  disk_size       = 20
  instance_types  = ["t2.small"]

  remote_access {
    ec2_ssh_key               = "EKS_KEY_PAIR"
    source_security_group_ids = [aws_security_group.allow_tls.id]
  }

  labels = {
    env = "dev"
  }

  scaling_config {
    desired_size = 2
    max_size     = 3
    min_size     = 1
  }

  update_config {
    max_unavailable = 1
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node_policy,
    aws_iam_role_policy_attachment.eks_cni_policy,
    aws_iam_role_policy_attachment.ec2_container_registry_read_only
  ]
}
```

This will create 2 resources.

The first block will create an EC2 instance for the Kubectl server.

Note that you need a key pair in your AWS account in the region you are deploying this in other to be able to ssh into the kubectl server. If you do not have one then you will need to create one.

The second block will create the AWS EKS node group, capacity type is set to ON DEMAND, instance type is t2.small, and disk size is set to 20. Scaling Config, maximum size is 3, desired size is 2, and minimum size is set to 1.

Now that the Terraform codes are ready its time to run our Terraform commands

```
terraform init
```

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*UxeZbeBl51QevGrL9iPWvg.png)

```
terraform validate
terraform plan
terraform apply
```

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*_LTmj4Wzy4Vl7l9TB2Y92w.png)

Terraform is creating the resources after running the terraform apply command.

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*gsglbUoHSzkiB5fiPcQVsw.png)

All resources have been created successfully. Let's check our AWS Account

### EKS Cluster created

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*xdnnYnrCLBVXNlUVIw_NOw.png)

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*LBjDPi-3Wxc9Gzsir6xUXQ.png)

Let's check the Kubectl Server and the other instances created

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*alier38pVsZJvOOfCTOFCA.png)

Let's check the VPC and Security Group, Route Table, and Subnets

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*7n4JnMcd23EC-dAd89rXAg.png)

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*YDKnJEyfFUdE0UuaD8Gq4Q.png)

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*ji5fWaBvWiyfLMOBX24yrQ.png)

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*UPpOOPJa5WF194R7SOryRQ.png)

Now that all resources have been provisioned by Terraform next step is to try to ssh into the Kubectl server.

I am using a newer version of Windows so I do not need to ssh with putty. If you are using an older version of Windows you will need to ssh into your instance using putty. You can learn how to do that here

[Putty Installation for Connect to your Linux instance from Windows](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html?source=post_page-----e37f4c5c66ad--------------------------------)

Click on the Instance Kubectl

![aws-eks](https://miro.medium.com/v2/resize:fit:640/format:webp/1*VcGdXFmmyd0grNY0l5pqAA.png)

On the next page click on connect

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*pgwyVO5GT4k5H5Gd9sL-sA.png)

Copy the ssh — i under Example

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*WG5suJgZlBfMOD9Ts4a6Cw.png)

Now head to your cmd line on your machine and cd into the directory where your Keypair(.pem) is stored. In my case, it's stored in the Downloads directory. Yours could be different so take note. Once you are in the directory where your Keypair is stored paste in the ssh line you copied and press enter

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*iLCh9pQQkNYATzYO0rlkRA.png)

I had connected to mine earlier so there was no prompt but if you are connecting for the first time there will be a prompt. Just type yes and you will be connected to your ec2 instance.

Let's install AWS CLI on the Kubectl Server

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*GEglP0MPpnyRehMMElW6ag.png)

Next up is to set up Kubectl on the ec2 instance. For that, we will run these command

```
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.27.1/2023-04-19/bin/linux/amd64/kubectl
openssl sha1 -sha256 kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
kubectl version --short --client
```

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*6yw33fTuQHPY0utoQxgh7A.png)

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*gQgmbljBM5PuuAhuG5RzCQ.png)

Let's set up our EKS Cluster on the Kubectl Server. These two commands should do just that

```
aws eks --region us-east-1 describe-cluster --name pc-eks --query cluster.status
aws eks --region us-east-1 update-kubeconfig --name pc-eks
```

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*lUEeeOGPreofeKEayrxEWQ.png)

While the first command showed our Kubectl is active the second command updated the Kubectl server by adding my aws:arn to the Kubectl server config

Finally, we will run this last command

```
kubectl get nodes
```

![aws-eks](https://miro.medium.com/v2/resize:fit:750/format:webp/1*1YG8YBSMiRE8Ib_aNSlnBA.png)

And here are the two nodes.

_**Note: Run terraform destroy to remove all provisioned infrastructures from your AWS account so as not to incur unnecessary bills on your account. Also, remember to delete the S3 bucket and the DynamoDB table**_

---

## **CONCLUSION**  

This guide provides a **step-by-step approach** to provisioning an **Amazon Elastic Kubernetes Service (EKS) cluster** on AWS using **Terraform**. It covers:  

✅ Setting up prerequisites, including an **S3 backend for state storage** and **DynamoDB for state locking**.  
✅ Creating **Terraform configuration files** to provision **networking, IAM roles, security groups, and the EKS cluster**.  
✅ Running **Terraform commands** to deploy infrastructure and verifying resources in the AWS account.  
✅ **Accessing the Kubectl server via SSH**, installing **AWS CLI & Kubectl**, and setting up cluster authentication.  
✅ Deploying an **EKS node group** for running workloads and ensuring **high availability**.  
✅ Verifying the **EKS cluster status** and managing it using **kubectl commands**.  

By following this guide, you can **successfully deploy, configure, and manage an EKS cluster** on AWS using Terraform. 🚀  

---

## 🤝 **Contributing**  

Contributions are welcome! If you'd like to improve this project, feel free to submit a pull request.  

---

## **Hit the Star!** ⭐

**If you find this repository helpful and plan to use it for learning, please give it a star. Your support is appreciated!**

---

## 🛠️ **Author & Community**  

This project is crafted by **[Haswanth](https://github.com/Haswanthkondamadugula)** 💡
I’d love to hear your feedback! Feel free to share your thoughts.  

