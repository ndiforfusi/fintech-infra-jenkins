# 🚀 **Fintech Infrastructure – Jenkins CI/CD for Terraform**

This repository provides a **Jenkins-based CI/CD pipeline** to deploy Terraform infrastructure, replacing the legacy **GitHub Actions** workflow.  
It supports **multi-environment deployments (dev, qa, uat, prod)**, integrates **AWS AssumeRole** or **static credentials**, and ensures **secure, auditable, and automated** provisioning.

---

## 🧭 **1. Overview**

### 🧩 **What It Does**
- ✅ Validates Terraform (`fmt`, `validate`)
- ✅ Runs plan and stores outputs as artifacts
- ✅ Supports manual approval gates before apply/destroy
- ✅ Handles per-environment state backends
- ✅ Supports both AssumeRole and static key authentication

### 🔁 **High-Level Flow**

```text
SCM Change ─▶ Checkout ─▶ Fmt/Validate ─▶ Plan + Archive ─▶ Manual Gate ─▶ Apply/Destroy
                 (Branch)     (TF 1.5.x)       (plan.tfplan)      (Approval)      (Safe Apply)


📁 2. Repository Layout
.
├── Jenkinsfile                    # CI/CD pipeline definition
├── dev/
│   ├── main.tf
│   └── backend.tf
├── qa/
├── uat/
└── prod/
Each environment folder corresponds to its own Terraform root and backend key.


⚙️ 3. Jenkins Requirements
🧩 Required Plugins
Pipeline (workflow-aggregator)

Git / GitHub Branch Source

Credentials Binding

AWS Steps (for AssumeRole)

AnsiColor

Timestamper 

Lock Resources

💻 Agent Requirements
Linux agent (Ubuntu preferred)

Terraform >= 1.5.x

Git, bash/sh

(Optional) Docker if containerized builds are preferred


🔐 4. AWS Authentication Options
🅰️ Option A — AssumeRole (Recommended)
Leverages AWS STS to assume a temporary role in the target account.


withAWS(region: params.REGION, role: params.ASSUME_ROLE_ARN, duration: 3600) {
  sh '''
    terraform init -upgrade -backend-config="key=${params.ENVIRONMENT}/terraform.tfstate"
    terraform plan -out plan.tfplan
    terraform apply -auto-approve plan.tfplan
  '''
}
🧾 Required IAM Configuration
Base IAM Role/User (Jenkins)
Grants permission to assume the target Terraform role:


{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::<TARGET_ACCOUNT>:role/TerraformDeployRole"
  }]
}
Target Role (TerraformDeployRole) — Trust Policy


{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::<JENKINS_ACCOUNT>:role/JenkinsBaseRole"
    },
    "Action": "sts:AssumeRole"
  }]
}
TerraformDeployRole Permissions Example


{
  "Effect": "Allow",
  "Action": [
    "s3:*",
    "dynamodb:*",
    "ec2:*",
    "iam:PassRole"
  ],
  "Resource": "*"
}
🅱️ Option B — Static Access Keys (Fallback)
For environments where AssumeRole is not available.


🔑 Step 1 — Create Jenkins Credentials
Go to: Manage Jenkins → Credentials → Global → Add Credentials

Type: AWS Credentials

ID: aws-static-creds

Fill in:

Access Key ID: <YOUR_ACCESS_KEY>

Secret Access Key: <YOUR_SECRET_KEY>


🧠 Step 2 — Reference in Jenkinsfile

withCredentials([[
  $class: 'AmazonWebServicesCredentialsBinding',
  credentialsId: 'aws-static-creds'
]]) {
  withEnv(["AWS_REGION=${params.REGION}"]) {
    sh '''
      terraform init -upgrade -backend-config="key=${params.ENVIRONMENT}/terraform.tfstate"
      terraform plan -out plan.tfplan
      terraform apply -auto-approve plan.tfplan
    '''
  }
}


🌿 5. Terraform Backend Configuration
Example S3 + DynamoDB backend (backend.tf):

terraform {
  backend "s3" {
    bucket         = "your-tf-state-bucket"
    region         = "us-east-2"
    dynamodb_table = "your-tf-locks"
    encrypt        = true
  }
}
The pipeline automatically uses:

terraform init -backend-config="key=${TF_ENV}/terraform.tfstate"


🧩 6. Jenkins Job Setup
🔹 Option A — Multibranch Pipeline (Recommended)
Jenkins → New Item → Multibranch Pipeline

Add GitHub Source

Point to your repo URL

Choose “By Jenkinsfile” for build configuration

Save → Jenkins auto-discovers branches and PRs


🔹 Option B — Single Pipeline Job
Jenkins → New Item → Pipeline

Set “Pipeline script from SCM”

Define script path as Jenkinsfile

Add parameters:

ENVIRONMENT → dev, qa, uat, prod

REGION → e.g., us-east-2

ACTION → apply | destroy

ASSUME_ROLE_ARN → optional


🧱 7. Pipeline Parameters
Parameter	Description	Default
ENVIRONMENT	Target environment directory	dev
REGION	AWS region	us-east-2
ACTION	Terraform action (apply or destroy)	apply
ASSUME_ROLE_ARN	ARN of IAM Role to assume	(optional)


⚡ 8. Pipeline Execution Flow
Scenario	Behavior
Pull Request	Runs fmt, validate, and plan only. Skips apply.
Release Branch	Runs full plan → approval gate → apply.
Destroy Run	Requires manual confirmation.


🔑 9. Approvals & Safeguards
✅ Manual approval step before apply or destroy

⏰ Timeout after 30 minutes (configurable)

🚫 disableConcurrentBuilds() prevents overlapping executions

🔒 Approval logic restricted to release branches


📦 10. Logs & Artifacts
Each build archives:

plan.tfplan (binary plan)

plan.txt (readable plan summary)

Logs include:

Timestamped output

ANSI-colored stages for clarity


🧠 11. Security Best Practices
🔄 Prefer STS AssumeRole over static keys

🧱 Restrict IAM permissions to minimal Terraform operations

🔐 Encrypt S3 and DynamoDB backends

🧩 Mask credentials in Jenkins logs

♻️ Rotate static keys regularly

🧰 Enable Jenkins RBAC

🕵️ Add pre-plan checks (tfsec, checkov, infracost)


🧩 12. Troubleshooting
Issue	Resolution
terraform not found	Install or configure via Manage Jenkins → Global Tool Configuration
backend error	Confirm S3 bucket, region, DynamoDB table
AssumeRole failed	Verify trust policy & sts:AssumeRole permissions
approval step stuck	Ensure authorized user clicks Proceed
PR triggered apply	Review when conditions in pipeline


🧱 13. Optional Enhancements
🧪 Quality Gates: Add tfsec or checkov

💸 Cost Insight: Integrate infracost

💬 ChatOps: Notify Slack/Teams on plan/apply

🔁 Workspaces: Parameterize workspace-based deployments

⚰️ Ephemeral Environments: Auto-destroy PR preview stacks



✅ 14. Verification Checklist
 Jenkins discovers all branches (Multibranch)

 Terraform 1.5.x installed on agents

 S3 backend and DynamoDB lock table configured

 IAM AssumeRole trust established

 Plan artifacts archived (plan.tfplan, plan.txt)

 Approval gate visible for apply/destroy

 Apply/Destroy succeeds per environment


🧾 15. References
Terraform Docs

Jenkins Pipeline Syntax

AWS AssumeRole Guide

AWS Steps Plugin

tfsec Security Scanner

🧡 Maintained by Fintech DevOps Team
"Automating infrastructure, empowering innovation."


---

### ✅ Improvements Made:
- Added emojis and markdown section dividers for GitHub readability.
- Clean tables and syntax highlighting.
- Clear AssumeRole + static credential implementation.
- Structured flow for CI/CD onboarding.
- Includes best practices, troubleshooting, and references.

---

Would you like me to now **add the actual Jenkinsfile** example (with both AssumeRole and static credential logic auto-detected) to go along with this README? It’ll make your repo completely plug-and-play.






