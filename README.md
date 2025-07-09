WordPress on Kubernetes with AWS EBS/EFS and ALB
![diagram-export-7-9-2025-2_31_54-PM](https://github.com/user-attachments/assets/9824fd74-2802-4571-902c-93c1dd7db46b)


🚀 Project Overview

This project demonstrates how to deploy a highly available WordPress website on a self‑managed Kubernetes cluster running on AWS. It utilizes:

MySQL database with EBS (gp3) storage via CSI

WordPress application pods with shared EFS storage via CSI

Kubernetes Service of type LoadBalancer to provision an AWS Application Load Balancer (ALB)

AWS Target Group mapping node ports to EC2 worker nodes

Multi‑AZ setup for high availability

Kubernetes Secrets for secure database credentials

📋 Prerequisites

AWS account with permissions to create EKS/EC2, EBS, EFS, ALB, Target Groups

Kubernetes cluster (self‑hosted or managed) with AWS cloud‑controller enabled

kubectl configured to communicate with your cluster

Docker (optional, for local image testing)

![Screenshot 2025-07-08 164238](https://github.com/user-attachments/assets/4c3991fe-58fc-4916-a40d-2385ad7a6bdf)

🔄 Scaling and Availability

Scale WordPress replicas:

kubectl scale deploy wordpress --replicas=2

Kubernetes will schedule pods across nodes in multiple AZs for redundancy.

🧹 Cleanup

⚠️ Remember to delete all AWS resources manually to avoid charges:

EFS filesystem

EBS volumes

Application Load Balancer (ALB)

Target Groups

EC2 instances (if provisioned separately)

And delete Kubernetes resources:

kubectl delete -f wordpress-svc.yaml
kubectl delete -f wordpress-deployment.yaml
kubectl delete -f mysql-service.yaml
kubectl delete -f mysql-deployment.yaml

📚 Further Reading

Kubernetes Service Types

AWS ALB for Kubernetes

Amazon EFS CSI Driver

Amazon EBS CSI Driver

👤 Author

Reda Taii - DevOps & Cloud Enthusiast

Feel free to star ⭐️ and fork! For feedback or questions, reach out on LinkedIn.

https://www.linkedin.com/in/reda-taii-b67484337
