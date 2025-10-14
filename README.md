
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

|                      Tool                      | 	                Purpose	                           |                     Why                                            |
|                   Git & GitHub	               |              Source control                         | Store and version application + CI/CD pipeline code                |
|                      Docker	                   |             Containerization	                       |    Package frontend and backend in portable images                 |
|                     Jenkins	                   |             CI/CD automation                        |	Automate build, test, Docker image creation, and EKS deployment   |                   AWS EKS	                   |           Container orchestration	                 Managed Kubernetes cluster for scalable deployment
Amazon RDS (MariaDB)	Database service	Fully managed database, integrated securely with backend
AWS CLI & kubectl	Cluster access	Used by Jenkins to connect and deploy on EKS
Terraform	Infrastructure automation	Created VPC, Subnets, EKS Cluster, IAM Roles (stored state in S3 + DynamoDB)

