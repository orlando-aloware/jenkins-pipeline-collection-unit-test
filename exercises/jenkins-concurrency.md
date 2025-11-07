# Exercise: Reproduce Terraform workspace / S3 name collision and learn Jenkins concurrency

Objective
- Reproduce a collision caused by near-simultaneous commits that run Terraform in parallel.
- Learn how Jenkins handles concurrency (disableConcurrentBuilds, abortPrevious), queueing, and how to protect Terraform with locks and idempotent checks.
- Produce a minimal pipeline + Terraform config that uses a workspace name `sandbox-<shortsha>` and an S3 bucket `shortsha-nov07-2025` (date example). Add a 2 minute sleep to force overlap.

Prerequisites
- Access to the Jenkins instance used by your org and a job you can edit.
- AWS credentials with permission to create S3 buckets (use a sandbox account).
- Git repo checked out under: `jenkins-pipeline-collection` branch `sandbox`.
- The exercise file should be added at this path in your local workspace:
  `/Users/orlando/_tmp/alwr/jenkins-pipeline-collection-unit-test/exercises/tf-workspace-issue-reproduce.MD`

Learning goals
- Understand `pipeline { options { disableConcurrentBuilds(...) } }` and what `abortPrevious: true` does.
- Learn how Jenkins queues jobs and how to scope serialization to only the critical Terraform section.
- See how a killed Terraform run can leave partial state/resources and why backend locking (DynamoDB) + Jenkins-level serialization are recommended.
- Learn simple idempotent checks for Terraform workspace and S3 bucket to reduce noisy errors.

Step 1 — Minimal Terraform config
- Create `main.tf` (same folder as Jenkinsfile in the job workspace) with a single S3 bucket resource. Use a variable for the bucket name.

Example (main.tf):
```hcl
variable "bucket_name" { type = string }

provider "aws" {
  region = "us-west-2"
}

resource "aws_s3_bucket" "site" {
  bucket = var.bucket_name
  acl    = "public-read"
  tags = {
    Name = "exercise-${var.bucket_name}"
  }
}
```

Step 2 — Jenkinsfile (exercise pipeline)
- The pipeline demonstrates two behaviors: (A) abort previous builds (race leading to interrupt), and (B) queue/lock terraform-critical steps so runs wait instead of aborting.

Example Jenkinsfile (simplified):
```groovy
pipeline {
  agent any

  // For the exercise toggle abortPrevious true/false to observe behavior
  options {
    // Change this between runs to see difference:
    // abortPrevious: true  -> Jenkins kills running build when new trigger happens
    // abortPrevious: false -> new triggers are queued
    disableConcurrentBuilds(abortPrevious: true)
  }

  stages {
    stage('Prepare') {
      steps {
        script {
          // compute short sha and date
          def shortsha = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          env.SHORTSHA = shortsha
          env.BUCKET = "${shortsha}-nov07-2025"
          env.WORKSPACE_NAME = "sandbox-${shortsha}"
          echo "SHORTSHA=${env.SHORTSHA}"
          echo "BUCKET=${env.BUCKET}"
          echo "WORKSPACE_NAME=${env.WORKSPACE_NAME}"
        }
      }
    }

    stage('Terraform') {
      steps {
        dir('terraform') {
          // Option A: no Jenkins lock — run may be aborted by Jenkins
          // Option B (preferred for real use): wrap critical section in lock(resource: "tf-${env.WORKSPACE_NAME}")
          script {
            // Intentionally sleep 2 minutes to force overlap and reproduce collision
            sh '''
              set -eu
              terraform init -backend-config="bucket=your-tfstate-bucket" \
                             -backend-config="key=sandbox/terraform.tfstate" \
                             -backend-config="region=us-west-2" \
                             -backend-config="dynamodb_table=terraform-state" || true

              # simulate long apply to cause overlap
              echo "Sleeping 120s to simulate long running apply..."
              sleep 120

              # create/select workspace idempotent pattern
              if terraform workspace list | grep -qw "${WORKSPACE_NAME}"; then
                terraform workspace select "${WORKSPACE_NAME}"
              else
                terraform workspace new "${WORKSPACE_NAME}" || terraform workspace select "${WORKSPACE_NAME}"
              fi

              # plan/apply
              terraform apply -var "bucket_name=${BUCKET}" --auto-approve
            '''
          }
        }
      }
    }
  }

  post {
    always {
      echo "Cleanup (local only)"
      sh 'terraform workspace select default || true'
    }
  }
}
```

Step 3 — Run the experiment
- Push two commits quickly (or trigger the same job twice within a short time window) so both runs use the same shortsha (or run twice with same shortsha).
- Observations to make:
  - With `abortPrevious: true`: Jenkins interrupts the first build when the second starts -> Terraform may fail with provider/plugin errors and leave partial resources.
  - With `abortPrevious: false`: Jenkins queues the second run; if you use a lock around the Terraform section the second build will wait at the lock and run only after the first completes.
  - If no backend DynamoDB locking is configured, two near-simultaneous applies can still race at the Terraform state level if Jenkins concurrency configuration is incorrect.

Expected errors to look for
- "Workspace 'pr-XXXX' already exists" on workspace new (benign but noisy).
- "Error: Plugin did not respond" or provider errors if Terraform was interrupted mid-apply.
- S3 create errors if the bucket already exists.

Step 4 — Variations and suggestions
- Replace `disableConcurrentBuilds(abortPrevious: true)` with `false` and re-run to see difference.
- Wrap the Terraform critical section with Lockable Resources:
```groovy
lock(resource: "tf-${env.WORKSPACE_NAME}") {
  // terraform init/select/apply here
}
```
- Enable S3 + DynamoDB backend locking in Terraform to prevent simultaneous state writes:
```hcl
terraform {
  backend "s3" {
    bucket         = "your-tfstate-bucket"
    key            = "sandbox/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-state"
    encrypt        = true
  }
}
```
- Implement retries + select-if-exists pattern for workspaces to make pipeline resilient.

Cleanup
- Manually delete any S3 buckets and Terraform workspaces created by the exercise:
```bash
# delete terraform workspace
terraform workspace select default
terraform workspace delete sandbox-<shortsha>

# delete s3 bucket (careful: will remove data)
aws s3 rb s3://<bucket-name> --force
```

Deliverables for the junior developer
- Add a Jenkinsfile (sandbox branch) and Terraform `main.tf` that follow the examples above.
- Run two near-simultaneous builds and capture Jenkins console logs + terraform logs.
- Produce a short 1-page note summarizing:
  - What happened when `abortPrevious: true` vs `false`
  - Whether a lock prevented the collision
  - Which combination of Jenkins locks + Terraform backend locking you recommend and why

Bonus tasks
- Implement `lock(resource: "...")` around the terraform section and re-run the experiment.
- Add a small helper script that implements: select-if-exists workspace, retry create if it fails (exponential backoff).
- Automate cleanup with a Jenkins job that runs after tests to destroy created resources.

Notes for reviewers / mentors
- This exercise is intentionally simple. Emphasize safe testing (use sandbox AWS account) and ensure you do not run this against production.
- Encourage the developer to explain why aborting in-progress Terraform runs is dangerous and how queueing + locks are a safer approach.
