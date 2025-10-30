# fintech-infra-jenkins
This guide explains how to implement, run, and maintain the Jenkins-based CI/CD pipeline that replaces your GitHub Actions workflow for Terraform infrastructure deployments.

1) Overview

What it does

Validates Terraform (fmt/validate)

Plans with per-environment backend state key

Stores the plan as build artifacts

Requires a manual approval (for protected branches/actions)

Applies or Destroys based on a parameter

High-level flow

SCM Change  ──▶  Checkout  ──▶  Fmt/Validate  ──▶  Plan + Archive  ──▶  Manual Gate  ──▶  Apply/Destroy
                (Multibranch)       (TF 1.5.x)      (plan.tfplan)        (release)          (idempotent)

2) Repo Layout
.
├─ Jenkinsfile                         # The pipeline you were given
├─ dev/                                # TF root for DEV
│  ├─ main.tf
│  └─ backend.tf                       # (optional if you pass -backend-config only)
├─ qa/
├─ uat/
└─ prod/


The pipeline expects TF_DIR to match the environment name (e.g., dev/, qa/, uat/, prod/).

3) Jenkins Requirements
Plugins

Pipeline (workflow-aggregator)

Git (SCM or GitHub Branch Source if multibranch on GitHub)

Credentials Binding

AWS Steps (for withAWS assume-role support) — recommended

AnsiColor

Timestamper

Agents

Linux agent with:

Terraform 1.5.x available (install or configure via Manage Jenkins → Global Tool Configuration as terraform-1.5.0)

Git

bash/sh

Optional: Docker if you prefer containerized execution. Update agent { label 'linux && docker' } accordingly.

Credentials

Choose one of the two approaches:

A) Recommended: Assume Role (STS)

Jenkins instance/agent has AWS auth capable of assuming a role (e.g., via instance profile or a base key stored once).

Create or use an IAM Role in target accounts for CI/CD (trusts Jenkins’ principal).

Provide its ARN at run time via parameter ASSUME_ROLE_ARN.

B) Static Keys (fallback)

In Jenkins: Manage Credentials → Add AWS Credentials with ID: aws-static-creds.

Least-privilege policy (only Terraform needs: S3 state, DynamoDB locks, and the services you manage).

4) Branch Strategy & Parity with GitHub Actions
Feature	GitHub Actions (Old)	Jenkins (New)
Trigger branches	release, PRs to main/release	Multibranch auto-discovers branches & PRs
Manual approval gate	Actions UI environment gate	input step for release or apply runs
Runtime inputs	environment, region, action	Jenkins parameters (ENVIRONMENT, REGION, ACTION)
Concurrency	concurrency.group	disableConcurrentBuilds()
Plan artifact	Console only	plan.tfplan and plan.txt archived/stashed
AWS auth	GHA secrets / OIDC	AssumeRole via withAWS or static creds
5) Configure the Job
Option A: Multibranch Pipeline (recommended)

Jenkins → New Item → Multibranch Pipeline

Set Branch Source to GitHub (or Git). Provide repo URL and credentials if needed.

In Build Configuration, choose “by Jenkinsfile” (default name Jenkinsfile).

Save → Scan Multibranch. Jenkins will create jobs for each branch/PR automatically.

Option B: Single Pipeline Job

Jenkins → New Item → Pipeline

Choose Pipeline script from SCM, point to the repo, set script path Jenkinsfile.

Enable This project is parameterized with:

ENVIRONMENT: dev|qa|uat|prod

REGION: default us-east-2

ACTION: apply|destroy

ASSUME_ROLE_ARN: (optional)

6) Terraform State Backend

The pipeline runs:

terraform init \
  -upgrade \
  -backend-config="key=${TF_ENV}/terraform.tfstate"


Configure the rest of your backend in backend.tf, for example (S3 + DynamoDB locks):

terraform {
  backend "s3" {
    bucket         = "your-tf-state-bucket"
    region         = "us-east-2"
    dynamodb_table = "your-tf-locks"
    encrypt        = true
  }
}


State file keys become dev/terraform.tfstate, qa/terraform.tfstate, etc.

7) Parameters & Defaults

ENVIRONMENT: dev, qa, uat, prod (maps to folder name and state key)

REGION: default us-east-2

ACTION: apply (default) or destroy

ASSUME_ROLE_ARN: optional (recommended over static keys)

You can set default parameter values in the Jenkins UI if desired.

8) Running the Pipeline
For PRs

When a PR is opened, the fmt/validate/plan stages run.

Approval/apply typically skipped for PRs (guarded by when logic). Use this to review plan.txt in the build artifacts.

For release (or direct branch builds)

Run a build with parameters (or on merge).

Review plan.txt under Artifacts.

Approve at the Approval Gate step.

Pipeline proceeds to Apply.

To Destroy

Run with ACTION=destroy and the target ENVIRONMENT. An approval may be required depending on your branch/run context.

9) AWS Authentication
Option A — Assume Role (Preferred)

Ensure Jenkins (controller/agent) has a base identity that can assume the target role.

At build time, provide ASSUME_ROLE_ARN (e.g., arn:aws:iam::<acct-id>:role/TerraformDeployRole).

The pipeline uses:

withAWS(region: params.REGION, role: params.ASSUME_ROLE_ARN, duration: 3600) {
  // terraform commands
}

Option B — Static Keys

Add AWS creds in Jenkins with ID aws-static-creds.

Ensure IAM policy is least-privilege.

The helper function binds keys securely and exports AWS_REGION.

10) Approvals & Safeguards

Approval Gate triggers for:

release branch, and/or

Non-PR runs where ACTION == 'apply'.

Timeout (default 30 minutes) to avoid indefinite blocking.

disableConcurrentBuilds() prevents overlapping executions.

Adjust gate conditions in stage('Approval Gate') → when { ... }.

11) Logs & Artifacts

Terraform plan saved as:

plan.tfplan (binary)

plan.txt (human readable)

Both are archived and fingerprinted.

Console logs include timestamps and ANSI colors for readability.

12) Security Best Practices

Favor STS AssumeRole over long-lived keys.

Lock down S3 state bucket and DynamoDB table with least privilege.

Mask secrets in logs via Credentials Binding.

Enable Jenkins RBAC and folder/containerized agents.

Rotate any static credentials regularly.

Consider Open Policy Agent / tfsec / checkov as an additional pre-plan gate.

13) Troubleshooting

Terraform not found

Install Terraform on agents or configure a Jenkins tool named terraform-1.5.0.

Optionally add to PATH in the pipeline’s environment block.

State/Backend errors

Confirm S3 bucket, region, and DynamoDB are correct.

Ensure IAM permissions for s3:* (scoped to bucket) and dynamodb:* (scoped to lock table) as needed.

AssumeRole fails

Verify trust policy trusts Jenkins’ base principal.

Check sts:AssumeRole permissions and session duration.

Approval step stuck

A user with Jenkins permission must click Proceed.

The gate times out after 30 minutes by default—adjust timeout.

PRs attempting to apply

Confirm the when conditions: PRs should only plan. Tighten rules if needed.

14) Optional Enhancements

Quality gates: add tfsec/checkov stage before planning.

Cost visibility: integrate infracost to annotate PRs/builds.

ChatOps: notify Slack/Teams on plan ready and on apply.

Workspaces: add Terraform Workspaces if your design prefers that over folder-per-env.

Ephemeral envs: parameterize a PR-<id> key and include auto-destroy jobs.

15) Migration Tips (from GitHub Actions)

Copy Jenkinsfile to repo root.

Create Multibranch Pipeline pointing to the repo.

Configure AWS auth (AssumeRole or static creds).

Confirm per-env folders exist (dev/, qa/, uat/, prod/).

Trigger a plan-only run (PR or manual).

Review artifacts → approve → apply.

16) Quick Verification Checklist

 Jenkins can discover branches/PRs (Multibranch).

 Agents have Terraform 1.5.x.

 S3 backend and DynamoDB lock table exist and are reachable.

 AWS credentials/role configured.

 plan.txt is archived after the Plan stage.

 Approval gate appears for release/apply runs.

 Apply/Destroy works per selected ENVIRONMENT.
