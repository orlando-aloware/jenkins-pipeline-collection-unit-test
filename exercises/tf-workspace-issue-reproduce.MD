xß# Exercise: Reproduce Terraform workspace / S3 bucket collision

Goal
- Create a Jenkins pipeline that creates a Terraform workspace named `sandbox-<shortsha>` and deploys an S3 bucket named `<shortsha>-nov7-2025` (use the current date in the bucket name). The pipeline should wait (sleep) 2 minutes after creating the workspace and bucket. Trigger the job twice in quick succession with the same `shortsha` to reproduce collisions:
  - Terraform error: workspace already exists
  - S3 error: bucket already exists

Target repo & path (for reference)
- Jenkinsfile to add (developer exercise):
  https://github.com/aloware/jenkins-pipeline-collection/blob/sandbox/sandbox-environment/Jenkinsfile
- Terraform config files should be in the same folder (sandbox-environment)
- We will only add this README in the unit-test exercises folder for the junior developer to follow.

Prerequisites
- Jenkins with Git plugin and a job that checks out the repo
- Terraform installed on the Jenkins agent
- AWS credentials/profile available on the agent (or set via Jenkins credential bindings)
- AWS account permission to create CloudFront/S3/Route53 if required (for this exercise we only create an S3 bucket)

What the developer must implement
1. A Jenkins pipeline (declarative) that:
   - Checks out the branch (whatever branch triggers the job)
   - Computes a short sha (7 chars) from the branch commit and stores it in `SHORT_SHA`
   - Sets a workspace name: `sandbox-${SHORT_SHA}`
   - Writes a small Terraform config in the workspace (or references an existing one) to create an S3 bucket named `${SHORT_SHA}-nov7-2025`
   - Runs `terraform init` and `terraform workspace new|select` (the exercise intention is to reproduce collision by running same shortsha twice)
   - Applies the Terraform plan to create the S3 bucket
   - Sleeps 120 seconds (2 minutes) AFTER creation to allow a race / duplicate trigger
   - Optionally, after the sleep, run another lightweight command or leave the job to finish

2. Terraform config (example main.tf) that creates a single S3 bucket with a name based on a variable `shortsha`.

Detailed Jenkinsfile example (developer should add this to `sandbox-environment/Jenkinsfile`):

```groovy
pipeline {
  agent any
  environment {
    // Jenkins will provide GIT_COMMIT; fallback to shell git if missing
    SHORT_SHA = "${env.GIT_COMMIT?.take(7) ?: sh(returnStdout: true, script: 'git rev-parse --short=7 HEAD').trim()}"
    WORKSPACE_NAME = "sandbox-${SHORT_SHA}"
    BUCKET_NAME = "${SHORT_SHA}-nov7-2025" // adjust date as needed
  }
  stages {
    stage('Prepare') {
      steps {
        echo "SHORT_SHA=${SHORT_SHA}"
        echo "WORKSPACE_NAME=${WORKSPACE_NAME}"
        echo "BUCKET_NAME=${BUCKET_NAME}"
        sh 'mkdir -p terraform && cp -r s3_bucket/* terraform/ || true'
      }
    }

    stage('Terraform Init') {
      steps {
        dir('terraform') {
          sh '''
            terraform init -input=false
            terraform validate || true
          '''
        }
      }
    }

    stage('Choose/Create Workspace') {
      steps {
        dir('terraform') {
          script {
            // Check if workspace exists and select or create accordingly
            def exists = sh(script: "terraform workspace list | grep -w ${WORKSPACE_NAME} || true", returnStdout: true).trim()
            if (exists) {
              echo "Workspace ${WORKSPACE_NAME} exists, selecting..."
              sh "terraform workspace select ${WORKSPACE_NAME}"
            } else {
              echo "Creating workspace ${WORKSPACE_NAME}"
              sh "terraform workspace new ${WORKSPACE_NAME}"
            }
          }
        }
      }
    }

    stage('Apply') {
      steps {
        dir('terraform') {
          sh "terraform apply -var 'shortsha=${SHORT_SHA}' -var 'bucket_name=${BUCKET_NAME}' -auto-approve"
        }
      }
    }

    stage('Hold for collision') {
      steps {
        echo 'Sleeping 120 seconds to allow collision reproduction'
        sh 'sleep 120'
      }
    }

    stage('Post') {
      steps {
        echo 'Done (exercise)'
      }
    }
  }
}
```

Notes about the Jenkinsfile
- The pipeline computes `SHORT_SHA` from `GIT_COMMIT` (provided by Jenkins); the fallback uses `git rev-parse` to support other runners.
- The `workspace` name is `sandbox-<shortsha>` to intentionally collide when the same short SHA is used twice.
- The `sleep 120` allows a second trigger to run while the first job still holds resources, increasing the chance to reproduce the "workspace already exists" or "bucket already exists" errors.

Terraform example files (developer should place these under `sandbox-environment/terraform` or the folder Jenkins copies from):

`terraform/main.tf` (simple, single S3 bucket):

```hcl
variable "shortsha" { type = string }
variable "bucket_name" { type = string }

provider "aws" {
  region = "us-west-2"
}

resource "aws_s3_bucket" "sandbox_bucket" {
  bucket = var.bucket_name
  acl    = "private"

  tags = {
    Name        = "sandbox-${var.shortsha}"
    Environment = "sandbox"
  }
}
```

`terraform/outputs.tf` (optional):

```hcl
output "bucket" {
  value = aws_s3_bucket.sandbox_bucket.id
}
```

Cleanup notes (important!)
- After reproducing the error, the developer should:
  - Run `terraform workspace select default` then `terraform workspace delete sandbox-<shortsha>` (if safe)
  - Run `terraform destroy -var 'shortsha=<shortsha>' -var 'bucket_name=<bucket>' --auto-approve` to remove the S3 bucket
- If the bucket creation failed with "bucket already exists" because it is owned by someone else, the exercise will show the error; cleanup may not be possible from your account (this is expected behaviour to demonstrate collision).

Expected errors to see during the exercise
- terraform workspace new <name>
  - "Workspace \"sandbox-<shortsha>\" already exists"
- terraform apply
  - "Error: Error creating S3 bucket: BucketAlreadyExists: The requested bucket name is not available"

Testing process
1. Push a branch that triggers the Jenkins job. Confirm the job runs and creates `sandbox-<shortsha>` and creates the bucket.
2. While the first job is sleeping (2 minutes), trigger the job again using the same commit (same shortsha). The second job will try to create the same workspace and S3 bucket and should surface the collision errors.
3. Observe Jenkins console logs for the workspace create/select messages and S3 error.

Variants & extensions (extra credit)
- Have the pipeline use `terraform workspace select` only and rely on the workspace being created by an external process; discuss pros/cons.
- Add a cleanup stage that checks PR status in GitHub (closed or merged) and then destroys the namespace/workspace.
- Extend the script to tag S3 buckets with PR numbers so it's easier to link buckets back to PRs.

What to submit
- A branch in `jenkins-pipeline-collection` named `sandbox` (already created) with the `sandbox-environment/Jenkinsfile` and `sandbox-environment/terraform/*` files implementing the snippets above.
- This README is only the exercise text and should be placed at:
  `/exercises/tf-workspace-issue-reproduce.MD` in the unit-test repo (this file)

If anything is unclear or you want me to produce the actual Jenkinsfile and Terraform files and push them to the `sandbox` branch, say the word and I will do that.

---

Generated: Nov 7, 2025
