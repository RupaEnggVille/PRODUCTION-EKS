# **Production-Grade AWS EKS Deployment with ALB, TLS & Route53 (Terraform Automated)**

## 📌 Overview

This project provisions a production-ready private Amazon EKS cluster using Infrastructure as Code (Terraform) and deploys microservices using Kubernetes with:

- AWS Load Balancer Controller (ALB)

- Kubernetes Ingress

- TLS termination using ACM

- Route53 DNS configuration

- Host-based routing

- Bastion host for secure access

The entire infrastructure layer is automated using Terraform.

## Architecture
Core AWS Services Used
- Amazon Web Services
- Amazon EKS
- AWS Application Load Balancer
- Amazon Route 53
- AWS Certificate Manager

<img src="EKS-Project/images/vpc.png">
<img src="./images/eks.png">
<img src="./images/k8s.png">

## Infrastructure Provisioned with Terraform
### Networking (Custom VPC)
- VPC
- Public Subnets
- Private Subnets
- Internet Gateway (IGW)
- NAT Gateway
- Route Tables
- Security Groups

#### Architecture model:
```
Public Subnet:
  - Bastion Host
  - NAT Gateway
  - ALB

Private Subnet:
  - EKS Worker Nodes
```

### Private EKS Cluster
- Private Endpoint Enabled
- IAM Roles & Policies
- OIDC Provider
- IRSA (IAM Roles for Service Accounts)
- Managed Node Groups

#### Security-first design:
- No public access to worker nodes
- Access only via Bastion host

### Bastion Host
Used for:
- Secure SSH access
- Kubectl access to private cluster
- Helm installation
- Controller installation

# **Prerequisites Setup for This Repository (AWS CLI + Terraform via Chocolatey)**

Before running this EKS Terraform project, install the required tools on your system using Chocolatey.

# **1. Install Chocolatey (if not already installed)**

Open PowerShell as Administrator and install Chocolatey:
```shell

Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; ` iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

```

Verify installation:

```shell
choco -v
```

# **2. Install AWS CLI using Chocolatey**

Install AWS CLI:
```shell
choco install awscli -y
```
Verify:
```shell
aws --version
```
# **3. Install Terraform using Chocolatey**

Install Terraform:
```shell
choco install terraform -y
```
Verify:
```shell
terraform -version
```
# **EKS Project Run Steps (Terraform)**

## **1.Clone the repository to local: create a empty directory in local .Then clone it**

or 
## **in vs code-->click on terminal-->new terminal-->select git bash-->change to local directory --> run git clone command-->after cloning finished--->cick on file-->open Folder-->select your cloned repository**
```shell
git clone https://github.com/RupaEnggVille/PRODUCTION-EKS.git

cd PRODUCTION-EKS
```
## **2. Go to Terraform working directory**

Based on your instructions:
```shell
ls
cd eks-project
ls
cd terraform
ls
cd eks
```
Make sure this folder contains: main.tf ,eks.tf,vpc.tf,provider.tf,dev.tfvars

## **3. Configure AWS CLI (mandatory):** for that go to browser create a IAM user (eks-user) in aws console for this project save access key and secret keys locally
```shell
aws configure

Set:

AWS Access Key : give your access key
Secret Key  : give your secret key
Region → us-east-1
json
```
## **4. Create S3 backend bucket (if not already created)**

Through aws console or through command
```shell
aws s3 mb s3://your-terraform-state-bucket --region us-east-1
```

in **backend.tf** change the bucket name 

bucket       = "backend-bucket-final-6526"   #create s3 bucket through aws console .and replace bucket name here

Update the region based on your AWS region. in backend block 


### In dev.tfvars 

Change Ami id of Ubuntu Server Based on region

ami_id        = "ami-091138d0f0d41ff90"  #replace with your ubuntu ami id based on region

Instance type based on your project size

instance_type = "t3.medium"  #change instance type

aws_region                = "us-east-1"   #change region

instance_types   = ["t3.medium"]  

if reguired change 

kubernetes_version        = "1.34" also


## **5. Create EC2 Key Pair (VERY IMPORTANT)**

Your project uses: check it in ec2.tf file which block you are using

data "aws_key_pair"

So key MUST already exist in AWS.**(create manually through aws console)**

If you use resource "aws_key_pair"
Generate through ssh-keygen

### Step 1: Create local key
```shell
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ec2_keypair
```
### Step 2: Import into AWS
```shell
aws ec2 import-key-pair \
  --key-name ec2_keypair \
  --public-key-material fileb://~/.ssh/ec2_keypair.pub \
  --region us-east-1
```  
### Step 3: Verify
```shell
aws ec2 describe-key-pairs --key-names ec2_keypair
```  
## **6. Initialize Terraform**
```shell
terraform init
```  
This will: download AWS provider ,initialize backend (S3) ,prepare modules

## **7. Validate configuration**
```shell
terraform validate
```
## **8. Plan infrastructure**
```shell
terraform plan -var-file="dev.tfvars"    #because all global configuration settings are defined in dev.tfvars. If you run only terraform plan, it will prompt you to enter values manually.”
```
We use dev.tfvars with terraform plan so Terraform already knows all values. If we don’t use it, Terraform will stop and ask us to enter them one by one.

## **9. Apply infrastructure**
```shell
terraform apply -var-file="dev.tfvars"

Type:yes
```
After resource creation completes, verify in the AWS Console that the bastion server, cluster, and VPCs ,IAM Roles are available. The process usually takes 10–15 minutes. Then copy the bastion server’s public IP address and launch a new Git Bash session.


## **10. Post Deployment (Bastion Access & Kubernetes Setup)**

Because cluster is private:SSH into Baston server 
```shell
ssh -i Downloads/ec2_keypair.pem ubuntu@bastion_public_ip
```

(example: ssh -i Downloads/ec2_keypair.pem ubuntu@(bastion_public_ip))

### **Configure Kubernetes access**
```shell
aws configure    #enter here access keys and secret keys ,region and json

AWS Access Key : give your saved access key

Secret Key  : give your saved secret key

Region → us-east-1

Output → json
```
```shell
aws eks update-kubeconfig --region us-east-1 --name dev-eks-demo
```
**Verify:**
```shell
kubectl get nodes
```
## **11. Install Helm (inside bastion)**
```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## **12. Install AWS Load Balancer Controller**
```shell
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```
Install:
```shell
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=(dev-eks-demo) \
  --set region=us-east-1 \
  --set vpcId=(vpc-id) \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=(IAM-role-arn)
```

replace with your clustername ,vpc id and iam role of loadbalancer(search in aws console iam-->roles-->search  AWSLoadBalancerControllerRole   copy the arn ) arn and remove () also

## **13. Clone your repository on the Bastion Server for microservices deployment**
```shell
git clone "https://github.com/harathi-mutyam/PRODUCTION-EKS.git"
cd PRODUCTION-EKS
ls
cd EKS-Project
cd k8s

kubectl get deployment -n kube-system

```

## **14. Deploy Microservices for path based routing  or Set up microservices deployment with path-based routing.**
```shell
kubectl apply -f ns.yaml
kubectl apply -f product.yaml
kubectl apply -f cart.yaml
kubectl apply -f payments.yaml
vim path-ingress.yaml
```

These configuration lines are used for HTTPS and SSL certificate setup, so comment 3 these lines in path-ingress.yaml file 
```shell
alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}, {"HTTP": 80}]'
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:071325923620:certificate/e6c8ab5f-3dcd-42b6-bbbb-2ff4e2b811fc
alb.ingress.kubernetes.io/ssl-redirect: '443'
```
**Ingress Deployment (HTTP ALB)**
```shell
kubectl apply -f path-ingress.yaml    #loadbalancer created now
kubectl get ingress -n e-commerce
```
**You will see:**

**ALB DNS name → k8s-default-xxxx.elb.amazonaws.com**

Before test it in browsers check it in aws console-->ec2-->load balncer-->provisioning or active.If state changed to active test in browser with alb-dns name

**Test in browser:**
```shell
http://`<ALB-DNS>`    -->payments
http://`<ALB-DNS>`/cart   --->cart
http://`<ALB-DNS>`/products   --->products
```

 ## 14. **HTTPS Setup** (ACM + GoDaddy)  Enable TLS (HTTPS Setup)
 
### **step 1: Request an acm certificate in aws console**

choose Request a public certificate  --> click on next --> Fully qualified domain name : your godaddy domainname (*.enggville.xyz)

click on request

Certificate Status --> pending validation

### **step 2: Add DNS in GoDaddy**

Open godaddy.com  --> Domain -->DNS  ---> Add New Record  -->

Type: CNAME 

Name : Copy CNAME Name upto before .enggville.xyz (.domainname)  from aws console --->Paste it here

value: Copy CNAME value from aws console (completly)  -->paste it here

After completing the above steps, the certificate status in the AWS Console changes from Pending to Issued.


## **15.Route53 Setup**

### **Create Route53 Hosted Zone**

Go to AWS Console → Route53 --> Click Hosted Zones  -->  Click on Create Hosted Zone --->

Domain: enggville.xyz   <Name must be your godaddy domain name>
Type: choose Public Hosted Zone
Click Create

You will see 2 important things:
NS (Name Servers) → 4 values
SOA record

Example: 4 values of ns servers
```shell
ns-123.awsdns-45.org
ns-456.awsdns-90.com
ns-789.awsdns-12.net
ns-222.awsdns-34.co.uk
```
copy ns records

## **16. Update GoDaddy Nameservers**

go to GoDaddy in browser:

Login
Go to:  My Products → enggville.xyz → DNS   --> Find Nameservers Tab   -->  Click Change 

Select:

Custom Nameservers : select “I’ll use my own nameservers”

Paste the 4 Route53 NS records without end . (remove last . from NSServers values)

**for example:**
```shell
ns-xxx.awsdns-xx.org
ns-xxx.awsdns-xx.com
ns-xxx.awsdns-xx.net
ns-xxx.awsdns-xx.co.uk
```
Save

Check ACM Region

Make sure certificate is in:us-east-1 (N. Virginia) ✅ REQUIRED for ALB

## **17. Enable HTTPS Ingress**

Now we will connect the AWS Load Balancer to Kubernetes. Open the path-ingress.yaml file, uncomment the following lines, and update the certificate ARN with your own value. After making the changes, save the file.

open path-ingress.yaml file
```shell
vim path-ingress.yaml
```
**uncomment the following lines** 
```shell
alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}, {"HTTP": 80}]'
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:071325923620:certificate/e6c8ab5f-3dcd-42b6-bbbb-2ff4e2b811fc    #Replace the certificate ARN with your own from AWS Certificate Manager
alb.ingress.kubernetes.io/ssl-redirect: '443'
```
**save the file  :wq!**
```shell
kubectl apply -f path-ingress.yaml
kubectl get ingress -n e-commerce
```
```shell
NAME                 CLASS   HOSTS   ADDRESS                                                                   PORTS   AGE
e-commerce-ingress   alb     *       k8s-ecommerc-ecommerc-949981ca5a-1099050560.us-east-1.elb.amazonaws.com   80      32s
```

**Test it in browser for Path-Based Routing
```shell
https://(ALB-DNS)    -->payments**
https://(ALB-DNS)/cart   --->cart  (https://k8s-ecommerc-ecommerc-949981ca5a-1471282601.us-east-1.elb.amazonaws.com/cart/)
https://(ALB-DNS)/products   --->products
```
Notes: ALB path-based routing  From your ALB rules: /product → product target group  ,/cart → cart target group ,/ → payments (default)

## **18. Host-Based Routing Setup:**
```shell
vim hostbased-ingress.yaml  # replace acm certicate arn with your certificate arn
kubectl get ingress  -n e-commerce
```

You will see:
```shell
NAME                     HOSTS   ADDRESS
e-commerce-ingress    cart.enggville.xyz,..     abc123.us-east-1.elb.amazonaws.com
```
```shell
kubectl describe ingress e-commerce-ingress -n e-commerce   #this command optional
```
#remove old alb(application load balancer of path based and create a new alb for host based)
```shell
kubectl delete ingress e-commerce-ingress -n e-commerce #syntax kubectl delete ingress IngressName -n NameSPaceName
```
*replace this (ingress name) with your ingress name*

```shell
kubectl apply -f hostbased-cart.yaml 
kubectl apply -f hostbased-product.yaml
kubectl apply -f payments.yaml    #payments.yaml is same for both path and host based because here we used /html/index.html but in cart and products we /prodcut/index.html , /cart/index.html for referece check it in another notes
kubectl apply -f hostbased-ingress.yaml
```

for debuggin if you got any isse use below commands check pods ,svc ,ingress and roll out 

#kubectl rollout restart deployment -n e-commerce

#kubectl get pods -n e-commerce

#kubectl get svc -n e-commerce

#kubectl get ingress -n e-commerce



## **19. Connect Domain → ALB using Route 53**

**Step 1: Open Hosted Zone**

Now go to Route53--> Hosted Zone → enggville.xyz

**Step 2: Create A Records (Alias → ALB)**

Create records: For EACH subdomain:
1. cart.enggville.xyz

REcord Name: cart 
Record Type: A record

Enable: Alias = YES

Route Traffic to : select Alias to Application and Classic Load Balancer

select region of your ALB : US East(N. Virginia)
select your ALB here  (for example: dualstack.k8s-ecommerc-ecommerc-949981ca5a-1099050560.us-east-1.elb.amazonaws.com)

2. product.enggville.xyz

REcord name: product 


same foloow here also

3. payments.enggville.xyz

REcord name: payments

same ALB


**Check it in browser :** with host names

https://cart.ehmutyam.xyz → output will be Welcome to cart Service    EnggVille Innovations

https:product.ehmutyam.xyz → Product Service

https:payments.ehmutyam.xyz → Payments Service


## **20. Process of Deletion**

**delete load balancer from baston server gitbash first**
```shell
kubectl delete -f hostbased-ingress.yaml
```
run this command in vs code gitbash

**Destroy Infrastructure (Cleanup)**
```shell
terraform destroy -var-file="dev.tfvars"
```



# GoDaddy is only: domain registrar

You either:

Option A:
**point NS records to Route53**

OR

Option B:
**directly add CNAME records**


# Other way to add CNAME Records in godaddy instead of ns records follow the below procedure

use default ns records of godaddy.com 

👉 DELETE ALL NS records if you have added in the first process

⚠️ No NS records needed at all.

## Step 1: Create SSL Certificate (ACM)

first create a ACM Certificate in aws console 

In Amazon Certificate Manager:--->Request certificate:  *.enggville.xyz

Choose: DNS validation

It will give CNAME records 

later add cname name  and cname value of a certificate in godaddy.com  as a DNS Records--> Add a New Record--> 

choose Type: CNAME record  Name: CNAME Name  Value: CNAME Value    -->click on Save

Go to Amazon Certificate Manager -->check Status is pending or ISSUED

Wait until status: ISSUED




## STEP 2 — Configure ALB Ingress (VERY IMPORTANT)

Update your Kubernetes hostbased-ingress.yaml:  <ACM-ARN>  #replace with your certificate ARN

### Apply:

```shell
 kubectl apply -f ns.yaml
 kubectl apply -f  hostbased-cart.yaml
 kubectl apply -f hostbased-product.yaml
 kubectl apply -f payments.yaml
 kubectl apply -f hostbased-ingress.yaml   
 kubectl get ingress -n e-commerce  #copy the alb DNS name
```

### create CNAME records in godaddy.com for https purpose 
```shell
type: CNAME       Name: cart          Data: alb DNS name 

type: CNAME       Name: product       Data: alb DNS name 

type: CNAME       Name: payments      Data: alb DNS name 
```
Example:

k8s-ecommerc-xxxxx.us-east-1.elb.amazonaws.com


## STEP 5 — Test flow

First check DNS:

nslookup cart.enggville.xyz

Then open:
```shell
https://cart.enggville.xyz
https://product.enggville.xyz
https://payments.enggville.xyz
```

# **Note**: **Difference between path based and host based routing**

## Why cart.yaml was modified (Path-based → Host-based Routing)
🔹 **Path-Based Routing (Before)**
URL used:

http://(ALB-DNS)/cart

Application needed to serve content from:

/cart/index.html

Health check:

/cart/index.html
## Host-Based Routing (Now)

URL used:

http://cart.enggville.xyz
No /cart path in URL ❌
Request goes directly to root / ✅
🔹 Changes Made

Moved file from:

/cart/index.html → /index.html

Updated health check:

/cart/index.html → /index.html
🔹 Reason

**In host-based routing, traffic comes to the root path (/), not /cart.
So the application must serve content from index.html instead of a subfolder.**

# ✅ Result

Application works correctly with:

http://cart.enggville.xyz
No 404 errors
ALB health checks pass successfully

Add 

