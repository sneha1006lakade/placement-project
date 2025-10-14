
# Deployment of 3-tier Application
---

## 🧾 Project Overview

This project demonstrates a complete deployment of 3-tier application.
The entire process automated build, test, docker image creation, image push to Dockerhub and deployment on eks.

---

## ⚙️ Tech stack

- Frontend - Angular
- Backend - Springboot (java)
- Database - Amazon RDS (MariaDB)

---

## 🧰 Tools Used & Why

| **Tool/Service**          | **Purpose**                                      | **Why?**                                                         |
| ------------------------- | ------------------------------------------------ | ---------------------------------------------------------------- |
| **Git & GitHub**          | Source control                                   | Store and version application + CI/CD pipeline code              |
| **Maven**                 | Code build                                       | Build the project code and create a artifact out of it           |
| **Sonarqube**             | Code test                                        | To test the code                                                 |
| **Docker**                | Containerization                                 | Package frontend and backend in portable images                  |
| **Jenkins**               | CI/CD automation                                 | Automate build, test, Docker image creation, and EKS deployment  |
| **AWS EKS**               | Container orchestration                          | Managed Kubernetes cluster for scalable deployment               |
| **Amazon RDS (MariaDB)**  | Database service                                 | Fully managed database, integrated securely with backend         |
| **AWS CLI & kubectl**     | Cluster access                                   | Used by Jenkins to connect and deploy on EKS                     |
| **Terraform**             | Infrastructure automation                        | Created VPC, Subnets, EKS Cluster, stored state in S3 + DynamoDB |
| **Datadog**               | Monitoring tool                                  | Used monitor the EKS cluster                                     |

---

## ⚙️ Deployment Steps

## Step 1: Infrastructure Setup
Launch an EC2 instance and provision AWS resources using terraform.
  
```bash
// Install AWS-CLI
apt update
apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

// Install Terraform
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

// Configure AWS
aws configure
  ```
---
Components created by terraform script:
 - VPC, Public & Private Subnets
 - Internet Gateway & NAT Gateway
 - EKS Cluster + Node Group
 - S3 & DynamoDB for remote backend
 - Route tables
---

- Terrafrom Structure 
```bash
Placement-project/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   |── network/
│   |   ├── vpc.tf
│   |   ├── nat.tf
│   |   ├── route_tables.tf
│   |   ├── variables.tf
│   |   └── outputs.tf
|   ├── backend/
|   │   ├── main.tf
|   │   ├── variables.tf
|   │   └── outputs.tf
|   ├── eks/
|   |   ├── main.tf
|   │   ├── variables.tf
|   │   └── outputs.tf
```
---
- network/vpc.tf
```bash
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "app-vpc" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags = { Name = "app-igw" }
}

# Public subnets
resource "aws_subnet" "public" {
  count                   = length(var.public_subnets)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnets[count.index]
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "public-${count.index + 1}"
  }
}

# Private subnets
resource "aws_subnet" "private" {
  count                   = length(var.private_subnets)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.private_subnets[count.index]
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = false

  tags = {
    Name = "private-${count.index + 1}"
  }
}
```
---
- network/nat.tf
```bash
resource "aws_eip" "nat" {
  domain = "vpc"
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id

  depends_on = [aws_internet_gateway.igw]
}
```
---
- network/route-tables.tf
```bash
# Public RT
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = { Name = "public-rt" }
}

resource "aws_route_table_association" "public_assoc" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private RT
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "private-rt" }
}

resource "aws_route" "private_nat" {
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat.id
}

resource "aws_route_table_association" "private_assoc" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```
---
- network/variables.tf
```bash
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "public_subnets" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnets" {
  type    = list(string)
  default = ["10.0.3.0/24", "10.0.4.0/24"]
}

variable "azs" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}
```
---
- network/output.tf
``` bash
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}
```
---
This version creates:

- 1 VPC
- 2 Public subnet 
- 2 Private Subnet
- 1 IGW
- 1 NAT Gateway
- Proper route tables
---
### 📄 Root-level main.tf
```bash
provider "aws" {
  region = var.aws_region
}

module "network" {
  source = "./network"
  aws_region = var.aws_region
}

output "vpc_id" {
  value = module.network.vpc_id
}

output "public_subnet_ids" {
  value = module.network.public_subnet_ids
}

output "private_subnet_ids" {
  value = module.network.private_subnet_ids
}
```
---
### 📄 Root-level variables.tf
```bash
variable "aws_region" {
  type    = string
  default = "us-east-1"
}
```
---
📄 Root-level outputs.tf
```bash
output "network_info" {
  value = {
    vpc_id           = module.network.vpc_id
    private_subnet_ids = module.network.private_subnet_ids
    public_subnet_ids  = module.network.public_subnet_ids
  }
	}
```
---
✅ Once this structure is ready:
1. Open terminal in:
    ```
    aws-deployment/terraform/
    
    ```
    
2. Run:
    
    ```bash
    terraform init
    terraform plan
    terraform apply
    
    ```
3. Terraform will create **VPC + subnets + IGW + NAT + routes**.

Let’s now set up **Terraform backend** — this ensures infrastructure state (`terraform.tfstate`) is safely stored in AWS (not just locally).

## What we’ll create:

- **S3 bucket** → stores Terraform state file
- **DynamoDB table** → used for Terraform state locking (prevents parallel edits)
---
### 📄 `backend/main.tf`
This will **create the S3 bucket + DynamoDB table** to store state.

```hcl
terraform {
  required_version = ">= 1.0.0"
}

provider "aws" {
  region = var.aws_region
}

resource "aws_s3_bucket" "tf_state" {
  bucket = var.bucket_name

  tags = {
    Name        = "terraform-state-bucket"
    Environment = "infra"
  }
}

resource "aws_s3_bucket_versioning" "versioning" {
  bucket = aws_s3_bucket.tf_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_dynamodb_table" "tf_lock" {
  name         = var.table_name
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name        = "terraform-lock-table"
    Environment = "infra"
  }
}

```
---

### 📄 `backend/variables.tf`

```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "bucket_name" {
  type    = string
  default = "my-terraform-state-bucket-demo"
}

variable "table_name" {
  type    = string
  default = "terraform-lock-table"
}

```
---

### 📄 `backend/outputs.tf`

```hcl
output "bucket_name" {
  value = aws_s3_bucket.tf_state.bucket
}

output "dynamodb_table" {
  value = aws_dynamodb_table.tf_lock.name
}
```
### Steps to apply backend

1. Go into the backend folder:
    
    ```bash
    cd aws-deployment/terraform/backend
    
    ```
    
2. Initialize Terraform:
    
    ```bash
    terraform init
    
    ```
    
3. Create the backend resources:
    
    ```bash
    terraform apply
    
    ```
### ⚙️ Then configure main Terraform backend

In root `aws-deployment/terraform/main.tf`,

add this **inside the `terraform {}` block**:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket-demo"
    key            = "global/terraform.tfstate"
    region         = "us-east-1"
    use_lockfile = true
    encrypt        = true
  }
}

```
---

### Next Step: **EKS Cluster**

Now we can move to provisioning **EKS cluster** on top of this network.

### Folder Structure

```
terraform/
└── eks/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf

```
---

### `variables.tf` (EKS module inputs)

```hcl
variable "cluster_name" {
  type    = string
  default = "app-eks-cluster"
}

variable "region" {
  type    = string
  default = "us-east-1"
}

variable "vpc_id" {
  type = string
}

variable "private_subnet_ids" {
  type = list(string)
}

variable "public_subnet_ids" {
  type = list(string)
}

variable "node_instance_type" {
  type    = string
  default = "t3.medium"  
}

variable "desired_capacity" {
  type    = number
  default = 2
}

variable "max_capacity" {
  type    = number
  default = 2
}

variable "min_capacity" {
  type    = number
  default = 2
}

```
---

### `main.tf` (EKS Cluster + Managed Node Group)

```hcl
provider "aws" {
  region = var.region
}

resource "aws_eks_cluster" "this" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster_role.arn

  vpc_config {
    subnet_ids = var.private_subnet_ids
  }

  depends_on = [aws_iam_role_policy_attachment.eks_cluster_policy]
}

resource "aws_iam_role" "eks_cluster_role" {
  name = "${var.cluster_name}-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster_role.name
}

# Managed Node Group
resource "aws_eks_node_group" "this" {
  cluster_name    = aws_eks_cluster.this.name
  node_group_name = "${var.cluster_name}-nodes"
  node_role_arn   = aws_iam_role.eks_node_role.arn
  subnet_ids      = var.private_subnet_ids

  scaling_config {
    desired_size = var.desired_capacity
    max_size     = var.max_capacity
    min_size     = var.min_capacity
  }

  instance_types = [var.node_instance_type]

  depends_on = [aws_eks_cluster.this]
}

resource "aws_iam_role" "eks_node_role" {
  name = "${var.cluster_name}-node-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "eks_worker_node_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_node_role.name
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.eks_node_role.name
}

resource "aws_iam_role_policy_attachment" "eks_registry_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.eks_node_role.name
}
```
---

### `outputs.tf` (EKS outputs)

```hcl
output "cluster_name" {
  value = aws_eks_cluster.this.name
}

output "cluster_endpoint" {
  value = aws_eks_cluster.this.endpoint
}

output "cluster_arn" {
  value = aws_eks_cluster.this.arn
}

output "node_group_role_arn" {
  value = aws_iam_role.eks_node_role.arn
}
```
Once this module is ready, we can integrate it in **root `main.tf`** like we did for network:
add in root main.tf
```hcl
module "eks" {
  source             = "./eks"
  cluster_name       = "app-eks-cluster"
  region             = var.aws_region
  vpc_id             = module.network.vpc_id
  private_subnet_ids = module.network.private_subnet_id
  public_subnet_ids  = module.network.public_subnet_id
}

```
- Our control plane is running on AWS cloud. We will add our master plane to this EC2 and perform operations

- Install kubectl

```jsx
# Install kubectl Download the latest release with the command

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

- Validate the binary

```jsx
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
```

- Validate the kubectl binary against the checksum file:

```jsx
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
```

- #Install kubectl:

```jsx
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

- #Note: If you do not have root access on the target system, you can still install kubectl to the ~/.local/bin directory:

```jsx
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
kubectl version --client
```

- Add cluster

```jsx
aws eks update-kubeconfig --region us-east-1 --name app-eks-cluster
```
- Now Access the cluster
---

## step 2: Create RDS Database 
- Go to AWS Console → RDS → Create Database

### In the **Connectivity section:**

- Choose **VPC** = EKS cluster VPC
- **Subnets** → choose **private subnets**
- **Public access** → ❌ *No (keep it private)*
- **VPC Security group** → create or select one that:
  - Allows inbound port **3306** from **EKS worker node security group**
---

### Wait for DB to be in “Available” state

Once it’s up:

- Copy its **endpoint** (something like `placement-project.cpq123abc.us-east-1.rds.amazonaws.com`)
- Keep that handy — we’ll put it in backend’s `application.properties` next.
---

### ✅ Next after that:

Once RDS is created and reachable, we’ll use the database:-
```bash
apt install mysql-client

mysql -h app-db.c4bmesy0ehmh.us-east-1.rds.amazonaws.com -u admin -p

CREATE DATABASE springbackend;
USE springbackend;
CREATE TABLE tbl_workers (
  id BIGINT(20) NOT NULL AUTO_INCREMENT,
  status VARCHAR(255) DEFAULT NULL,
  workerfname VARCHAR(255) DEFAULT NULL,
  workerlname VARCHAR(255) DEFAULT NULL,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO `tbl_workers` (`id`, `status`, `workerfname`, `workerlname`) VALUES
(1, 'Working', 'Ivan', 'Holicek'),
(37, 'Vacation', 'Marko', 'Markovic'),
(40, 'Working', 'Ivo', 'Ivica'),
(41, 'Working', 'Luka', 'Lukovic'),
(42, 'Working', 'Filip', 'Filipovic');

SELECT * FROM tbl_workers;

```
---
update frontend/.env
```
.env
```
in frontend build to use
```
VITE_API_URL="http://backend-service:8085/api"
```
---

## Step 3: Jenkins Setup on EC2

Install Jenkins, Docker, and dependencies:
```bash
sudo apt update -y
sudo apt install -y openjdk-17-jdk docker.io awscli unzip
sudo systemctl enable docker
sudo systemctl start docker
```

Install Jenkins:
```bash
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
/usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update -y
sudo apt install -y jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
```
---
Add Jenkins user to Docker group:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
- Jenkins will be accessible at :—
```bash
http://<EC2-1-Public-IP>:8080
```
- Jenkins plugins:
    - Git
    - Docker Pipeline
    - Pipeline: aws steps
    - pipeline: stage view
    - sonarqube scanner
    - AWS Credentials
      
- Credentials to add in Jenkins
  Go to **Manage Jenkins → Credentials → Global**
    - `dockerhub` → Docker Hub username/password
    - `aws-credentials` → AWS Access Key & Secret
    - 'sonarqube-token' → secret text
    - 'github-token' → secret text
---
- Now we need to run sonarqube server inside docker container 
```bash
docker run -d --name sonarqube-custom -p 9000:9000 sonarqube:10.6-community
```

- Sonarqube accessible at:
```
http://<EC2-1-Public-IP>:9000
```

- **Create Project in SonarQube UI**
 - Go to **Projects → Create Project → Manually**.
 - Enter project key.
 - Generate a **token** (used for authentication by Jenkins/Maven).
 - Create a webhook for sonarqube
 - Go to Administration —> configuration —> webhook —> enter name, <jenkins_url>/<sonarqube-webhook> —> create
---
 - Add environment of sonarqube server in jenkins system
   settings —> system —> sonarqube server —> check the box of Environment variables —> enter name, <Url of server > —> select generated token —> save
---

- Now write a  **`Jenkinsfile`** and save this file as in repo root.
- In Jenkins → Create a **Pipeline job** → Choose “Pipeline script from SCM”.
- Connect GitHub repo.
- Build the job — it should:
    - Build and test the code
    - Build & push image to dockerhub
    - Deploy to EKS backend and frontend pod
    - Sent post success and failure mail

---

- After deploying the application successfully Verify Deployment:
```bash
kubectl get pods
kubectl get svc
kubectl get ingress
kubectl describe pod <pod-name>
```
---

- To monitor the pods install datadog-agent by using helm and we can easily see the logs on datadog dashboard 
Datadog UI → Logs → Log Explorer

---
## 🧩 Troubleshooting & Common Errors
   
| **Issue**                                	| ** Root Cause **                                         |	**Fix**                                                     |
|-------------------------------------------|----------------------------------------------------------|--------------------------------------------------------------|
| Can't connect to MySQL server	            | RDS not in same VPC/subnet or SG mismatch	               | Allow inbound from EKS SG to RDS                             |
| Authorization failed while pulling image  |	Private Docker repo	                                     | Made DockerHub images public                                 | 
| Unable to locate credentials in Jenkins	  | Missing AWS credentials	                                 | Configured AWS CLI                                           |
| YAML mapping values not allowed	          | Indentation error	                                       | Fixed YAML syntax alignment                                  |
| ImagePullBackOff                          |	Image inaccessible                                       | Checked repo name/tag, fixed indentation, made public        |
| Jenkins email error                      	| No recipient set - authentication required	             | Added app password and enable 2-steps verification           |



