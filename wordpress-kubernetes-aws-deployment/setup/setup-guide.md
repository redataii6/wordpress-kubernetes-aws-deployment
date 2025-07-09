# WordPress on Kubernetes with AWS Storage

A complete guide to deploying WordPress on Kubernetes using AWS EBS for MySQL storage and EFS for WordPress files, with external access via Application Load Balancer.

## Overview

This project demonstrates how to deploy a scalable WordPress application on Kubernetes with:
- **MySQL database** with persistent storage using AWS EBS
- **WordPress frontend** with shared storage using AWS EFS
- **High availability** with pod scaling across multiple nodes
- **External access** via AWS Application Load Balancer

## Architecture

```
Internet → AWS ALB → Kubernetes Cluster
                   ├── WordPress Pods (EFS storage)
                   └── MySQL Pod (EBS storage)
```

## Prerequisites

- Kubernetes cluster running on AWS EC2 instances
- AWS CLI configured with appropriate permissions
- kubectl configured to access your cluster
- AWS IAM user/role with EBS and EFS permissions

## AWS Setup

### 1. Create IAM User with Permissions

Create an IAM user with the following AWS managed policies:
- `AmazonEBSCSIDriverPolicy`
- `AmazonElasticFileSystemFullAccess`

### 2. Store AWS Credentials as Kubernetes Secret

```bash
kubectl create secret generic aws-credentials \
  --namespace kube-system \
  --from-literal=key_id=YOUR_ACCESS_KEY_ID \
  --from-literal=access_key=YOUR_SECRET_ACCESS_KEY
```

### 3. create IAM role and Add permissions EBS > "AmazonEBSCSIDriverPolicy" and EFS ...

Custom policy for EFS access:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:DescribeAccessPoints",
                "elasticfilesystem:DescribeFileSystems",
                "elasticfilesystem:DescribeMountTargets",
                "ec2:DescribeAvailabilityZones"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:CreateAccessPoint"
            ],
            "Resource": "*",
            "Condition": {
                "StringLike": {
                    "aws:RequestTag/efs.csi.aws.com/cluster": "true"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticfilesystem:TagResource"
            ],
            "Resource": "*",
            "Condition": {
                "StringLike": {
                    "aws:ResourceTag/efs.csi.aws.com/cluster": "true"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": "elasticfilesystem:DeleteAccessPoint",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/efs.csi.aws.com/cluster": "true"
                }
            }
        }
    ]
}
```

## Install CSI Drivers

### Install EBS CSI Driver

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.14"
```

### Install EFS CSI Driver

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.5"
```

### Verify Installation

```bash
kubectl get csidriver
```

Expected output:
```
NAME              ATTACHREQUIRED   PODINFOONMOUNT   STORAGECAPACITY   TOKENREQUESTS   REQUIRESREPUBLISH   MODES        AGE
ebs.csi.aws.com   true             false            false             <unset>         false               Persistent   50m
efs.csi.aws.com   false            false            false             <unset>         false               Persistent   48m
```

## MySQL Database Setup
# ---Create_a_secret_named_mysql-pass_that_stores_your_MySQL-password--- #
> ubuntu@master:~/WordPressPoject$ echo -n 'mypasswordsql' | openssl base64
### 1. Create MySQL Password Secret

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
type: Opaque
data:
  password: bXlwYXNzd29yZHNxbA==  # Base64 encoded password
```

### 2. Create EBS Storage Class

```yaml
# mysql-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mysql-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer  # Waits for pod to be scheduled
```

### 3. Create Persistent Volume Claim

```yaml
# mysql-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce  # EBS volumes support single pod access
  storageClassName: mysql-sc
  resources:
    requests:
      storage: 5Gi
```

### 4. Create MySQL Deployment

```yaml
# mysql-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-app
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.6
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass  # Reference to the secret created above
                  key: password
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql  # MySQL data directory
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
```

### 5. Create MySQL Service

```yaml
# mysql-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
spec:
  selector:
    app: wordpress
    tier: mysql
  ports:
    - port: 3306  # MySQL standard port
  type: ClusterIP  # Internal access only
```

## WordPress Application Setup

### 1. Create EFS Storage Class

```yaml
# wordpress-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
```

### 2. Create EFS File System and Access Point

**Manual Step**: Go to AWS Console → EFS → Create File System
- Note the File System ID (e.g., `fs-01454ca8b588f2d41`)
- Create an Access Point with root directory `/wordpress` and permissions `0777`
- Note the Access Point ID (e.g., `fsap-0fd51eb91e39a8083`)
- Ensure security groups allow NFS traffic (port 2049) from worker nodes

### 3. Create Persistent Volume

```yaml
# wordpress-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany  # EFS supports multiple pod access
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-01454ca8b588f2d41::fsap-0fd51eb91e39a8083  # Format: FileSystemID::AccessPointID
```

### 4. Create Persistent Volume Claim

```yaml
# wordpress-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

### 5. Create WordPress Deployment

```yaml
# wordpress-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
        - name: wordpress
          image: wordpress:php7.1-apache
          env:
            - name: WORDPRESS_DB_HOST
              value: mysql-svc  # Reference to MySQL service
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-pass
                  key: password
          ports:
            - containerPort: 80
          volumeMounts:
            - mountPath: /var/www/html  # WordPress files directory
              name: wordpress-storage
      volumes:
        - name: wordpress-storage
          persistentVolumeClaim:
            claimName: wordpress-efs-pvc
```

### 6. Create WordPress Service

```yaml
# wordpress-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-svc
spec:
  selector:
    app: wordpress
    tier: frontend
  ports:
    - port: 80
  type: LoadBalancer  # Creates AWS Load Balancer for external access
```

## External Access Setup

### AWS Application Load Balancer (Manual Creation)

1. **Go to EC2 Console** → Load Balancers → Create Load Balancer → Application Load Balancer

2. **Basic Configuration**:
   - Name your ALB
   - Scheme: Internet-facing

3. **Network Mapping**:
   - Select at least 2 Availability Zones
   - Choose public subnets
   - Ensure worker nodes are in these AZs

4. **Security Groups**:
   - Allow inbound HTTP (port 80)
   - Allow HTTPS (port 443) if needed

5. **Listeners and Routing**:
   - Listener: HTTP on port 80
   - Forward to Target Group with:
     - Protocol: HTTP
     - Port: NodePort from WordPress service
     - Target type: instance
     - Registered targets: EC2 worker nodes

## Scaling WordPress

Scale WordPress deployment to multiple replicas:

```bash
kubectl scale deploy wordpress --replicas=3
```

This demonstrates:
- Load balancing across multiple pods
- Shared storage access with EFS
- High availability setup

## Cleanup

### Delete Kubernetes Resources

```bash
kubectl delete -f /path/to/your/manifests/
```

### Delete AWS Resources (Manual)

⚠️ **Important**: Manually delete these AWS resources to avoid charges:
- EFS File System and Access Points
- Application Load Balancer
- EBS volumes (if not automatically deleted)

## Troubleshooting

### Common Issues

1. **PVC stuck in Pending**: Check if StorageClass and CSI drivers are installed
2. **EFS mount issues**: Verify security groups allow NFS traffic (port 2049)
3. **LoadBalancer pending**: Check if AWS Load Balancer Controller is installed
4. **Database connection errors**: Verify MySQL service name and secret

### Useful Commands

```bash
# Check pod status
kubectl get pods -o wide

# Check storage
kubectl get pv,pvc

# Check services
kubectl get svc

# View logs
kubectl logs deployment/wordpress
kubectl logs deployment/mysql-app
```

## Security Considerations

- Store sensitive data in Kubernetes Secrets
- Use IAM roles with minimal required permissions
- Configure security groups to restrict access
- Consider using TLS/SSL for production deployments
- Regularly update container images

## Next Steps

- Implement HTTPS with TLS certificates
- Add monitoring and logging
- Set up automated backups
- Configure horizontal pod autoscaling
- Implement CI/CD pipeline