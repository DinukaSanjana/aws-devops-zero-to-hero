# Kubernetes End-to-End Project on EKS  
**Deploy 2048 Game with AWS ALB Ingress Controller using Fargate (Serverless)**  
**Instructor:** Abhishek Veeramalla (#abhishekveeramalla)  
**Video:** https://youtu.be/RRCrY12VY_s  
**GitHub Repo:** https://github.com/iam-veeramalla/aws-devops-zero-to-hero/tree/main/day-22  

---

### Why Do We Need a VM / Local Machine?
We need a laptop or virtual machine (Ubuntu/Windows/Mac) to install CLI tools and run commands. All the tools below talk to AWS APIs and the EKS cluster from outside the cluster.

---

### Tools Installation & Why We Need Them

| Tool        | Installation (Linux/macOS)                                                                                   | Why We Need It |
|-------------|--------------------------------------------------------------------------------------------------------------|----------------|
| **AWS CLI** | `curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && unzip awscliv2.zip && sudo ./aws/install` | To interact with all AWS services from terminal |
| **kubectl** | `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && chmod +x kubectl && sudo mv kubectl /usr/local/bin/` | To send commands to Kubernetes API server |
| **eksctl**  | `curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \| tar xz -C /tmp && sudo mv /tmp/eksctl /usr/local/bin` | Official CLI for creating & managing EKS clusters |
| **Helm**    | `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \| bash`                        | Package manager for Kubernetes (used to install ALB Controller easily) |

---

### Configure AWS CLI with Credentials
1. Go to AWS Console → IAM → Users → Your User → Security credentials  
2. Create Access Key → Download `.csv`  
3. Run:
```bash
aws configure
```
Enter:
- Access Key ID
- Secret Access Key
- Default region: `us-east-1`
- Output format: `json`

This allows all tools (eksctl, kubectl, helm) to authenticate with your AWS account.

---

### Why Fargate? (vs EC2 Worker Nodes)
| EC2 Worker Nodes                 | Fargate (Serverless)                              |
|----------------------------------|----------------------------------------------------|
| You manage OS, patching, scaling | AWS fully manages compute, no servers to manage   |
| Cheaper for long-running loads   | Pay only for pod runtime, perfect for learning    |
| More control                     | Zero server management, highly available by default |

**In this project → we use Fargate** so we don’t worry about worker nodes at all.

---

### Step 1: Create EKS Cluster with Fargate Profile
```bash
eksctl create cluster \
  --name demo-cluster \
  --region us-east-1 \
  --fargate
```
- Creates fully managed control plane + Fargate profile  
- Takes ~18–20 minutes

---

### Step 2: Update kubeconfig (Connect kubectl to cluster)
```bash
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```

---

### Step 3: Create Dedicated Fargate Profile for Game Namespace
```bash
eksctl create fargateprofile \
  --cluster demo-cluster \
  --region us-east-1 \
  --name alb-sample-app \
  --namespace game-2048
```
**Why?** So only pods in `game-2048` namespace run on Fargate (isolated & predictable billing)

---

### Step 4: Deploy 2048 Game (Deployment + Service + Ingress)
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
Creates:
- Namespace: `game-2048`
- Deployment (5 replicas)
- Service (NodePort)
- Ingress with class `alb`

---

### Verify Application
```bash
kubectl get pods -n game-2048
kubectl get svc -n game-2048
kubectl get ingress -n game-2048
```

---

### What is Ingress & Why Do We Need It?
- Kubernetes Service (ClusterIP/NodePort) cannot be accessed directly from internet  
- LoadBalancer service creates expensive Classic ELB (~$20/month)  
- **Ingress + ALB Controller** → creates AWS Application Load Balancer automatically (cheaper + advanced routing)

---

### Step 5: Install AWS Load Balancer Controller (ALB Ingress Controller)

#### 5.1 Enable IAM OIDC Provider
```bash
eksctl utils associate-iam-oidc-provider \
  --cluster demo-cluster \
  --region us-east-1 \
  --approve
```
**Why?** Allows pods to assume IAM roles securely (IRSA)

#### 5.2 Download IAM Policy
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

#### 5.3 Create IAM Policy
```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

#### 5.4 Create IAM Role + ServiceAccount (IRSA)
```bash
eksctl create iamserviceaccount \
  --cluster demo-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --override-existing-serviceaccounts
```

#### 5.5 Add Helm Repository
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

#### 5.6 Install AWS Load Balancer Controller using Helm
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=YOUR_VPC_ID
```

#### 5.7 Verify ALB Controller is Running
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

### Step 6: Check Final Ingress (ALB is now created)
```bash
kubectl get ingress -n game-2048
```
Example output:
```
NAME           CLASS HOSTS ADDRESS                                                                 PORTS AGE
ingress-2048   alb   *     k8s-game2048-ingress2048-abc123-123456789.us-east-1.elb.amazonaws.com   80    12h
```

Copy the ADDRESS → Paste in browser → **2048 Game Loads Successfully!**

---

### Cleanup (Delete Everything)
```bash
eksctl delete cluster --name demo-cluster --region us-east-1
```

---

**Project Completed!**  
You now have production-grade experience in:
- EKS + Fargate (Serverless K8s)
- AWS ALB Ingress Controller
- IAM Roles for Service Accounts (IRSA)
- Helm package management
- Secure AWS authentication

**#devops #aws #eks #kubernetes**
