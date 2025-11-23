# Kubernetes End-to-End Project on EKS  
**Deploy a 2048 Game Application with AWS ALB Ingress Controller using Fargate**

**Project Goal**: Create a fully managed EKS cluster on AWS Fargate, deploy the 2048 game application, and expose it publicly using AWS Application Load Balancer (ALB) via Ingress.

**Reference**: Abhishek Veeramalla – AWS DevOps Zero to Hero  
**GitHub Repo**: https://github.com/iam-veeramalla/aws-devops-zero-to-hero/tree/main/day-22  
**Video Tutorial**: https://youtu.be/RRCrY12VY_s

---

### Prerequisites (Installed on Local Machine)

- AWS CLI
- kubectl
- eksctl
- Helm (for ALB Controller installation)

---

### Step 1: Create EKS Cluster with Fargate

```bash
eksctl create cluster \
  --name demo-cluster \
  --region us-east-1 \
  --fargate
```

This command creates:
- Fully managed EKS control plane (highly available)
- Fargate profiles (serverless worker nodes – no EC2 management)
- Default add-ons: vpc-cni, kube-proxy, coredns, metrics-server

Cluster ready in ~18–20 minutes.

---

### Step 2: Update kubeconfig

```bash
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```

---

### Step 3: Create Fargate Profile for Application Namespace

```bash
eksctl create fargateprofile \
  --cluster demo-cluster \
  --region us-east-1 \
  --name alb-sample-app \
  --namespace game-2048
```

This ensures pods in `game-2048` namespace run on Fargate.

---

### Step 4: Deploy 2048 Game (Deployment + Service + Ingress)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

Creates:
- Namespace: `game-2048`
- Deployment: `deployment-2048` (5 replicas)
- Service: `service-2048` (NodePort)
- Ingress: `ingress-2048` (class: alb)

Verify:

```bash
kubectl get pods -n game-2048
kubectl get svc -n game-2048
kubectl get ingress -n game-2048
```

---

### Step 5: Install AWS Load Balancer Controller (ALB Ingress Controller)

#### 5.1 Enable IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster demo-cluster \
  --region us-east-1 \
  --approve
```

#### 5.2 Download IAM Policy for ALB Controller

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

#### 5.3 Create IAM Policy

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

#### 5.4 Create IAM Service Account (IRSA)

```bash
eksctl create iamserviceaccount \
  --cluster demo-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --override-existing-serviceaccounts
```

#### 5.5 Add Helm Repo & Install ALB Controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<YOUR_VPC_ID>
```

Verify deployment:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

### Step 6: Access the Application

After ~2–5 minutes, the ALB will be provisioned:

```bash
kubectl get ingress -n game-2048
```

Output example:

```
NAME          CLASS  HOSTS   ADDRESS                                                              PORTS   AGE
ingress-2048  alb    *       k8s-game2048-ingress2-bcac0b5b37-664855256.us-east-1.elb.amazonaws.com   80      11h
```

Open the ADDRESS in browser → 2048 game loads successfully!

---

### Key Concepts Covered

| Concept                     | Description |
|----------------------------|-----------|
| EKS + Fargate              | Fully serverless Kubernetes (no EC2 worker nodes) |
| ALB Ingress Controller    | Replaces Classic Load Balancer / NGINX Ingress with native AWS ALB |
| IRSA (IAM Roles for Service Accounts) | Secure way for pods to access AWS services |
| Ingress with `alb` class   | Automatic ALB provisioning based on Ingress rules |
| Fargate Profiles           | Control which namespaces/pods run on Fargate |

---

### Cleanup (Delete Cluster)

```bash
eksctl delete cluster --name demo-cluster --region us-east-1
```

---

**Project Completed Successfully!**  
You now have hands-on experience with:
- EKS cluster creation using eksctl
- Fargate (serverless Kubernetes workers)
- Deploying apps with Ingress
- AWS ALB Ingress Controller setup
- IRSA and secure IAM permissions
