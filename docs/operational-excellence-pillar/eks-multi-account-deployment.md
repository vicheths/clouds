# Deploying a Production-Ready Amazon EKS Cluster with EC2 Worker Nodes and Loadbalance using IaC Terraform

This guide explains how to deploy a Kubernetes cluster on AWS using Amazon EKS (v1.31), Amazon EC2 worker nodes managed by an Auto Scaling Group, and a Network Load Balancer (NLB) configured for a gRPC or HTTP gateway. 
All infrastructure is provisioned using Terraform with a modular and reusable code structure for multi-account setup.

---

## 🧱 Architecture Overview

* **EKS Cluster**: I'll be using version 1.31 deployed in my custom VPC/subnets, you can use any version, but recommended with the latest version. see lifecycle event from aws here: https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html
* **Worker Nodes**: EC2 instances (Amazon Linux 2, ie:`m5a.large`) managed via Launch Template and ASG.
* **Auto Scaling Group**: Configured with min=1, max=2. (Update based on your application consumption needed).
* **IAM Roles**: For EKS control plane and EC2 worker nodes.
* **Security Group**: Allows communication between the control plane and worker nodes.
* **Network Load Balancer (NLB)**: Listens on port `443` and forwards to port `9080` on worker nodes (Note: The port 9080 is for API-apisix, and you should update to your application port).

---

## 📁 Terraform Project Structure 
* This terraform structure is best for multi-account deployment: For example, you have dev, sit,uat, and production environments, and you want to deploy the configuration to all env then you only need to update the environment variables under envs/.tfvars

```
terraform-eks/
├── main.tf
├── provider.tf
├── variables.tf
├── envs/
│   └── dev.tfvars
└── modules/
    ├── eks_cluster/
    │   ├── main.tf
    │   ├── iam.tf
    │   ├── variables.tf
    │   └── user_data.sh
    ├── network_lb/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── security_group/
        ├── main.tf
        └── variables.tf
```

---

## 🌐 `provider.tf`

```hcl
provider "aws" {
  region = var.region
}

terraform {
  required_version = ">= 1.3"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

---

## 📥 `variables.tf`

```hcl
variable "region" {
  type    = string
  default = "ap-southeast-1"
}

variable "vpc_id" {}

variable "subnet_ids" {
  type = list(string)
}
```

---

## 📁 `envs/dev.tfvars`

```hcl
region     = "ap-southeast-1"
vpc_id     = "vpc-xxxxxxxx"
subnet_ids = ["subnet-aaa", "subnet-bbb"]
```

---

## 🔐 `modules/security_group/main.tf`

```hcl
resource "aws_security_group" "eks_worker_sg" {
  name   = "eks-worker-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

output "eks_sg_id" {
  value = aws_security_group.eks_worker_sg.id
}
```

---

## ☸️ `modules/eks_cluster/main.tf`

```hcl
resource "aws_eks_cluster" "this" {
  name     = "my-eks-cluster"
  version  = var.eks_version
  role_arn = aws_iam_role.eks_cluster_role.arn

  vpc_config {
    subnet_ids         = var.subnet_ids
    security_group_ids = var.security_group_ids
  }
}

resource "aws_launch_template" "eks_worker_lt" {
  name_prefix   = "eks-worker-"
  image_id      = data.aws_ami.eks.id
  instance_type = var.instance_type

  user_data = base64encode(data.template_file.user_data.rendered)

  iam_instance_profile {
    name = aws_iam_instance_profile.worker_profile.name
  }
}

data "aws_ami" "eks" {
  most_recent = true
  owners      = ["602401143452"]
  filter {
    name   = "name"
    values = ["amazon-eks-node-${var.eks_version}-*"]
  }
}

data "template_file" "user_data" {
  template = file("${path.module}/user_data.sh")

  vars = {
    cluster_name = aws_eks_cluster.this.name
  }
}

resource "aws_autoscaling_group" "eks_workers" {
  desired_capacity     = 1
  max_size             = 2
  min_size             = 1
  vpc_zone_identifier  = var.subnet_ids
  launch_template {
    id      = aws_launch_template.eks_worker_lt.id
    version = "$Latest"
  }
  target_group_arns = var.target_group_arns

  tag {
    key                 = "kubernetes.io/cluster/${aws_eks_cluster.this.name}"
    value               = "owned"
    propagate_at_launch = true
  }
}
```

---

## ☸️ `modules/eks_cluster/iam.tf`

```hcl
resource "aws_iam_role" "eks_cluster_role" {
  name = "eks-cluster-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action = "sts:AssumeRole",
      Principal = {
        Service = "eks.amazonaws.com"
      },
      Effect = "Allow"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_attach" {
  role       = aws_iam_role.eks_cluster_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_iam_role" "eks_worker_role" {
  name = "eks-worker-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action = "sts:AssumeRole",
      Principal = {
        Service = "ec2.amazonaws.com"
      },
      Effect = "Allow"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_worker_attach_1" {
  role       = aws_iam_role.eks_worker_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "eks_worker_attach_2" {
  role       = aws_iam_role.eks_worker_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}

resource "aws_iam_role_policy_attachment" "eks_worker_attach_3" {
  role       = aws_iam_role.eks_worker_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_instance_profile" "worker_profile" {
  name = "eks-worker-profile"
  role = aws_iam_role.eks_worker_role.name
}
```

---

## 🧾 `modules/eks_cluster/user_data.sh`

```bash
#!/bin/bash
set -o xtrace
/etc/eks/bootstrap.sh ${cluster_name}
```

---

## 🌐 `modules/network_lb/main.tf`

```hcl
resource "aws_lb" "nlb" {
  name               = var.name
  internal           = false
  load_balancer_type = "network"
  subnets            = var.subnet_ids
}

resource "aws_lb_target_group" "tg" {
  name        = "${var.name}-tg"
  port        = var.target_port
  protocol    = "TCP"
  vpc_id      = var.vpc_id
  target_type = "instance"
}

resource "aws_lb_listener" "listener" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = var.listener_port
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg.arn
  }
}
```

---

## 📤 `modules/network_lb/outputs.tf`

```hcl
output "target_group_arn" {
  value = aws_lb_target_group.tg.arn
}
```

---

## 🧮 `modules/network_lb/variables.tf`

```hcl
variable "name" {}
variable "subnet_ids" {
  type = list(string)
}
variable "vpc_id" {}
variable "listener_port" {}
variable "target_port" {}
```

---

## ✅ Final Notes

* You can now apply this with the terraform command below:
 
`terraform init` - To initialize the terraform state and configuration.  
`terraform workspace list` - To list current workspace or env where this configuration will deploy to.  
`terrafrom workspace new dev` - Create a new workspace or env call: "dev".  
`terraform workspace select dev` - To select which workspace or env you wants to deploy to.  
`terraform plan -var-file="envs/dev.tfvars"` - The Command to check what the configuration will be deployed (ie: change, add, or destroy) before apply it to the env.  
`terraform apply -var-file="envs/dev.tfvars"` - The Command to apply the change to our env.  

* One it deployed The resourecs will be created in to your aws infrastructure, which you can verify on aws console.
* The worker nodes will automatically register to the NLB target group via ASG.
* Customize `user_data.sh` for installing node agents or runtime configs.

---

Feel free to fork or clone this example and deploy it into your own AWS environment. Contributions are welcome!

---

**Author:** Vicheth Sim
**GitHub:** https://github.com/vicheths/clouds

---

Let me know if you'd like to add GitHub Actions for CI/CD or integrate Helm charts for app deployments next!
