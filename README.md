**Prerequisites Setup for This Repository (AWS CLI + Terraform via Chocolatey)**

Before running this EKS Terraform project, install the required tools on your system using Chocolatey.

⚙️ 1. Install Chocolatey (if not already installed)

Open PowerShell as Administrator and install Chocolatey:

Set-ExecutionPolicy Bypass -Scope Process -Force; `
[System.Net.ServicePointManager]::SecurityProtocol = `
[System.Net.ServicePointManager]::SecurityProtocol -bor 3072; `
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

**Verify installation:**

choco -v

**2. Install AWS CLI using Chocolatey**

Install AWS CLI:

choco install awscli -y

**Verify:**

aws --version

**3. Install Terraform using Chocolatey**

**Install Terraform:**

choco install terraform -y

**Verify:**

terraform -version


**Changes were made to the EKS project on 27/04/2026.**

**For EKS-Project**

Steps to Clone and Run the Project

**1. Create a Local Folder**

Create a folder in your local directory named production-eks.


**2. Clone the Repository**

Open VS Code (or Git Bash) and clone the repository:

git clone https://github.com/harathi-mutyam/PRODUCTION-EKS.git

**3. After Cloning the Repository**

**Configure AWS CLI**

Create an IAM user with AdministratorAccess in your AWS account and generate access keys for it.

Then configure the AWS CLI using the following command:

aws configure

Set:

AWS Access Key

Secret Key

Region → us-east-1



Make the following changes:

Update the region in dev.tfvars based on the location where you want to create your infrastructure (such as EKS, VPC, etc.).

Example:

region = "us-east-1"

Ensure the region in backend.tf matches the region where your Terraform state storage (for example, an S3 bucket) is hosted.

create a s3 bucket in aws account

**4. Navigate to the Terraform Directory**

Always run Terraform commands from the folder where main.tf exists:

cd eks-project/terraform/eks

**5. Verify Terraform Installation**

terraform version

**6. Initialize Terraform and validate it**

terraform init

terraform validate

**7. Plan the Infrastructure**

terraform plan -var-file="dev.tfvars"


**changes were made on 28/4/2026**



Removed duplicate OIDC-related resources from eks.tf since they already exist in iam.tf as eks_oidc.

The following OIDC provider block was commented out to prevent duplication:

/* resource "aws_iam_openid_connect_provider" "eks-oidc" {

  client_id_list  = ["sts.amazonaws.com"]
  
  thumbprint_list = [data.tls_certificate.eks-certificate.certificates[0].sha1_fingerprint]
  
  url             = data.tls_certificate.eks-certificate.url
  
} */

Also removed the duplicate TLS certificate data source in iam.tf to keep a single source of truth:

/* data "tls_certificate" "eks-certificate" {

  url = aws_eks_cluster.eks[0].identity[0].oidc[0].issuer
  
} */

This ensures the OIDC configuration is defined only once in iam.tf, avoiding conflicts and duplication.


**YOU must create a keypair before terraform apply** 

If you use a data block for aws_key_pair in ec2.tf , you must first create the key pair in your AWS account before running Terraform.


data "aws_key_pair" "key_pair" {
  key_name = var.key_name
}



If you use a resource block for aws_key_pair in ec2.tf , you should first generate the SSH key locally, and then Terraform will create the AWS key pair for you automatically.



/*resource "aws_key_pair" "key_pair" {
  key_name   = var.key_name
  public_key = file("~/.ssh/id_rsa.pub")
}*/



Step 1: Create SSH key locally

ssh-keygen -t rsa -b 4096 -f ~/.ssh/ec2_keypair

This creates:

~/.ssh/ec2_keypair

~/.ssh/ec2_keypair.pub

**Step 2: Import into AWS (VERY IMPORTANT)**

aws ec2 import-key-pair \
  --key-name ec2_keypair \
  --public-key-material fileb://~/.ssh/ec2_keypair.pub \
  --region us-east-1

Now AWS knows your key.

**Step 3: Verify it exists**

aws ec2 describe-key-pairs --key-names ec2_keypair


**8. Apply the Changes**

terraform apply -var-file="dev.tfvars"

further steps follow stepstorun-eks-project-through-terraform notes



