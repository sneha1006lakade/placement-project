
# Deployment of 3-tier Application
---

##🧾 Project Overview

This project demonstrates a complete deployment of 3-tier application.
The entire process automated build, test, docker image creation, image push to Dockerhub and deployment on eks.

---

##⚙️ Tech stack

- Frontend - Angular
- Backend - Springboot (java)
- Database - Amazon RDS (MariaDB)

---

##🧰 3. Tools Used & Why

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

##⚙️ Deployment Steps

Step 1: Infrastructure Setup
Launch an EC2 instance and provision AWS resources using terraform.
  
'''bash
// Install AWS CLI
apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
//Install Terraform
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
//configure AWS
aws configure
'''


  - Components created:
     VPC, Public & Private Subnets
  - Internet Gateway & NAT Gateway

EKS Cluster + Node Group

IAM Roles & Policies

S3 & DynamoDB for remote backend

terraform init
terraform plan
terraform apply -auto-approve
