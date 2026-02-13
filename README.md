🚀 AWS EKS + ALB Ingress Project (Game 2048 Deployment)

This project demonstrates deploying a Kubernetes application on Amazon Elastic Kubernetes Service and exposing it using AWS Load Balancer Controller.

Reference Implementation: Abhishek Veeramalla – AWS DevOps Zero to Hero (Day 22)

🏗 Architecture Overview
```
User → ALB → Target Group → EKS Worker Nodes → Kubernetes Service → Pods
```

🛠 Prerequisites
```
AWS Account
IAM User (Not Root) with AdministratorAccess
AWS CLI
kubectl
eksctl
Helm
```

<img width="1920" height="1080" alt="Screenshot (9)" src="https://github.com/user-attachments/assets/d28c31af-24a7-47fa-bf84-dc83cba5702e" />
<img width="1920" height="1080" alt="Screenshot (8)" src="https://github.com/user-attachments/assets/098dd5bb-1dd7-4849-9d3d-4222becd4e1d" />

🔹 Step 1: Create EKS Cluster
```
eksctl create cluster \
  --name demo-cluster \
  --region ap-south-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2
```

✅ Why?

We need a managed Kubernetes control plane.
EKS handles:

API Server

etcd

High Availability

We manage:

Worker Nodes

Applications

🧠 Interview Answer

EKS is used to offload Kubernetes control plane management to AWS while maintaining scalability and availability.

<img width="1920" height="1080" alt="Screenshot (3)" src="https://github.com/user-attachments/assets/453f6892-b65b-4e53-ad8b-8fded6726120" />

<img width="1920" height="1080" alt="Screenshot (4)" src="https://github.com/user-attachments/assets/a9ba4090-9aed-4318-8daf-bc2dc89ce513" />


🔹 Step 2: Associate IAM OIDC Provider
```
eksctl utils associate-iam-oidc-provider \
  --cluster demo-cluster \
  --approve
```

✅ Why OIDC is Required?

EKS uses IAM Roles for Service Accounts (IRSA).

By default:

Pods cannot directly assume IAM roles.

OIDC enables:

Secure identity mapping between Kubernetes Service Account and IAM Role

Temporary credentials via AWS STS

Fine-grained permission control

Without OIDC:

AWS Load Balancer Controller cannot create ALB resources.

🧠 Interview Answer

OIDC allows Kubernetes service accounts to assume IAM roles securely without storing AWS credentials inside pods.

🔹 Step 3: Create IAM Policy for Load Balancer Controller
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

```
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

✅ Why?

AWS Load Balancer Controller needs permission to:

Create ALB

Create Target Groups

Modify Listeners

Register Targets

IAM policy defines what AWS resources the controller can manage.

🧠 Interview Answer

We follow least privilege principle by attaching only required ELB and EC2 permissions to the controller role.

<img width="1920" height="1080" alt="Screenshot (5)" src="https://github.com/user-attachments/assets/6e5aa2f3-bb40-4387-9304-e4b53085d288" />

🔹 Step 4: Create IAM Role & Kubernetes Service Account

```
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT-ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

✅ Why?

This step:

Creates IAM Role

Attaches policy

Creates Kubernetes Service Account

Links them via IRSA

Now controller pod can assume IAM role automatically.

🧠 Interview Answer

This enables secure AWS API access without embedding static access keys in containers.

🔹 Step 5: Install AWS Load Balancer Controller (Helm)
```
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=<YOUR-VPC-ID>
```

✅ Why?

The controller:

Watches Kubernetes Ingress resources

Automatically provisions ALB

Creates Target Groups

Attaches security groups

It bridges Kubernetes and AWS.

Without this controller:

Ingress resource will not create ALB.

🔹 Step 6: Deploy Application (2048 Game)
```
kubectl create namespace game-2048
```
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/examples/2048/2048_full.yaml
```

✅ What This Does

Creates Deployment

Creates Service

Creates Ingress

Ingress triggers ALB creation via controller.

<img width="1920" height="1080" alt="Screenshot (2)" src="https://github.com/user-attachments/assets/a0ee8ab4-558b-4723-a1f3-f40ab54084da" />

🔹 Step 7: Verify Ingress
```
kubectl get ingress -n game-2048
```

Copy ALB DNS.

🔹 Step 8: Access Application
```
http://<ALB-DNS>
```

⚠️ Must use HTTP (Port 80)

🔎 Troubleshooting Section
If ALB not created:

Check controller logs:
```
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

If Targets Unhealthy:

Check:
```
EC2 → Target Groups → Health
```

Ensure:

Security Group allows HTTP 80

Pods running

Correct health check path (/)

🧠 Key Learning Concepts

Kubernetes Ingress

ALB provisioning via controller

IAM Roles for Service Accounts (IRSA)

OIDC authentication flow

Target Group health checks

Security group communication

🏁 Final Output

Successfully exposed Kubernetes application using AWS ALB.

Game accessible publicly via internet-facing load balancer.

🧹 Cleanup
```
eksctl delete cluster --name demo-cluster --region ap-south-1
```

🔥 Why This Project Is Important

This demonstrates:
Real-world production-grade ingress setup
Secure IAM integration
Infrastructure automation
Cloud-native networking

🚀 Technologies Used

Amazon Elastic Kubernetes Service
AWS Load Balancer Controller
Kubernetes
Helm
IAM
ALB
EC2

Output :
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/6097944c-4b33-4490-8ced-ab814234142c" />
