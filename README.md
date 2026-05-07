# **Prerequisites Setup for This Repository (AWS CLI + Terraform via Chocolatey)**

Before running this EKS Terraform project, install the required tools on your system using Chocolatey.

## Install Chocolatey (if not already installed)

Open PowerShell as Administrator and install Chocolatey:

Set-ExecutionPolicy Bypass -Scope Process -Force; `
[System.Net.ServicePointManager]::SecurityProtocol = `
[System.Net.ServicePointManager]::SecurityProtocol -bor 3072; `
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

### Verify installation:

choco -v

## **Install AWS CLI using Chocolatey**

### **Install AWS CLI:**

choco install awscli -y

### **Verify:**

aws --version

## **Install Terraform using Chocolatey**

### **Install Terraform:**

choco install terraform -y

### **Verify:**

terraform -version


# **For EKS-Project**

Steps to Clone and Run the Project

## **1. Create a Local Folder**

Create a folder named "production-eks" in any drive in your local.

## **2. Clone the Repository**

Open VS Code (or Git Bash) and clone the repository to production-eks folder as destination.

[git clone https://github.com/RupaEnggVille/PRODUCTION-EKS.git]

## **3. In AWS Console**

Create an IAM user with AdministratorAccess in your AWS account and generate access & secret keys for the user.

### **Configure AWS CLI**

Then configure the AWS CLI using the following command:

aws configure

Set: AWS Access Key, Secret Key, Region → us-east-1, Output Format

## **4. Create an S3 backend bucket**

### If your backend.tf uses S3:
Create an S3 bucket through aws console or through command (after aws configure using AWS CLI)

aws s3 mb s3://your-terraform-state-bucket --region us-east-1

Also ensure DynamoDB table exists if state locking is used.

## **5. After Cloning the Repository**

### Make the following changes:

Update the region in dev.tfvars based on the location where you want to create your infrastructure (such as EKS, VPC, etc.).
Example:
region = "us-east-1"

Ensure the region in backend.tf matches the region where your Terraform state storage (for example, an S3 bucket) is hosted.

Also Update other Variable Values in dev.tfvars like key-pair, AMI-Id, Instance type (ec2 & eks), capacity, etc.

## **6. Create Key-pair for EC2 & EKS cluster (VERY IMPORTANT)**

### **If your project uses data "aws_key_pair"**

So key MUST already exist in AWS.(create manually through aws console)

"Make sure that the key-pair name should match with the variable value in dev.tfvars."

### **If you use resource "aws_key_pair"**
Generate through ssh-keygen 

### **Step 1: Create local key**

ssh-keygen -t rsa -b 4096 -f ~/.ssh/ec2_keypair

### **Step 2: Import into AWS**

aws ec2 import-key-pair \
  --key-name ec2_keypair \
  --public-key-material fileb://~/.ssh/ec2_keypair.pub \
  --region us-east-1
  
### **Step 3: Verify**

aws ec2 describe-key-pairs --key-names ec2_keypair

## **7. Navigate to the Terraform Directory**

Always run Terraform commands from the folder where main.tf exists:

cd EKS-Project/terraform/EKS

## **8. Initialize Terraform and validate it**

terraform init

This will download AWS provider plugins, initialize backend (S3), prepare modules, initializes other plugins.

terraform validate

This will check whether the configuration is valid or not.

## **7. Plan the Infrastructure**

terraform plan -var-file="dev.tfvars"

**Check carefully:**
VPC creation, EKS cluster ,Node groups, EC2 bastion, IAM roles

## **8. Apply the Changes**

terraform apply -var-file="dev.tfvars"
Type: yes

or use 

terraform apply -var-file="dev.tfvars" --auto-approve

## **9. Post Provisioning Setup (Inside Bastion)**

After terraform completes

### Because cluster is private: 
SSH into Bastion to access EKS cluster nodes

ssh -i ~/.ssh/ec2_keypair ubuntu@bastion-public-ip

**To Configure Kubernetes access, configure AWS credentials in Bastion.**

aws configure

**Update .kube/config file** 

aws eks update-kubeconfig --region us-east-1 --name eks-demo

### Verify Cluster:

kubectl get nodes

## **10. Install Helm (inside bastion)**

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh

## **11. Add helm Repository to Install AWS Load Balancer Controller**

### Add Helm repository for EKS:

helm repo add eks https://aws.github.io/eks-charts

### Update helm repository:

helm repo update

### Install AWS Load Balancer controller:

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=dev-eks-demo \
  --set region=us-east-1 \
  --set vpcId=vpc-id \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=IAM-role

**Replace the values cluster name, region, VPC Id, AWS Load Balancer Controller IAM role ARN**

### Verify:

kubectl get deployment -n kube-system

## **12. Clone the Source code Repository for manifest files Or Create Manifest files manually in Bastion**

git clone https://github.com/RupaEnggVille/PRODUCTION-EKS.git

**change the working directory to k8s:**

cd PRODUCTION-EKS/EKS-Project/k8s/

## **13. Deploy Microservices**

kubectl apply -f ns.yaml
kubectl apply -f product.yaml
kubectl apply -f cart.yaml
kubectl apply -f payments.yaml

# Ingress Deployment

## **13. Path-Based Routing:**

### Ingress Deployment (HTTP ALB)

Before applying ingress.yaml comment listen-ports, certificate-arn & ssl-redirect lines under annotations.

vim ingress.yaml

#alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}, {"HTTP": 80}]'
#alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:071325923620:certificate/e6c8ab5f-3dcd-42b6-bbbb-2ff4e2b811fc
#alb.ingress.kubernetes.io/ssl-redirect: '443'


kubectl apply -f ingress.yaml

kubectl get ingress

You will see:

ALB DNS name → k8s-default-xxxx.elb.amazonaws.com

Test in browser:

http://<ALB-DNS>

 14. Enable TLS (HTTPS Setup)
Step 1: Create Route53 Hosted Zone
example.com

Go to AWS Console:
Create Hosted Zone
Domain: example.com

This gives:
NS records
Hosted zone ID

Map Domain → ALB

Create CNAME or A record (Alias):

Example:
Record	Value
cart.example.com	ALB DNS
product.example.com	ALB DNS
payments.example.com	ALB DNS

Step 2: Create ACM Certificate

Request certificate for:

cart.example.com
product.example.com
payments.example.com
Step 3: Update Ingress

Update ingress.yaml:

alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}, {"HTTP": 80}]'
alb.ingress.kubernetes.io/certificate-arn: <cert-arn>
alb.ingress.kubernetes.io/ssl-redirect: '443'

Apply:

kubectl apply -f ingress.yaml

15. Host-Based Routing (Production)
Domain	Service
cart.example.com	Cart
product.example.com	Product
payments.example.com	Payments

16. Destroy Infrastructure (Cleanup)
terraform destroy -var-file="dev.tfvars"






