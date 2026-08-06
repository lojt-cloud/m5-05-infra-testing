Lab M5.05 - Infrastructure Testing & Validation
## Scenario

Recent audits found S3 buckets without encryption, resources missing required cost-allocation tags, and naming conventions violated across environments. This repository implements a multi-layer testing strategy so that no infrastructure change can merge without passing automated validation — catching issues at every stage from the local editor to the CI pipeline.

## Infrastructure

Terraform configuration provisions:

aws_s3_bucket.data_store — versioned, KMS-encrypted, with a public access block
aws_s3_bucket_versioning.data_store — versioning enabled
aws_s3_bucket_server_side_encryption_configuration.data_store — SSE-KMS with S3 bucket keys enabled
aws_s3_bucket_public_access_block.data_store — blocks all public ACLs/policies
aws_dynamodb_table.state_lock — pay-per-request table with point-in-time recovery enabled

## Testing Layers

## Layer 1: Static Analysis
terraform fmt -check — enforces consistent formatting
terraform validate — checks syntax and provider compatibility
tflint — catches provider-specific issues and best practice violations

## Layer 2: Convention Validation (scripts/validate-conventions.sh)
Resource naming must be lowercase with underscores
Required tags: Name, Environment, ManagedBy
All variables must have descriptions
No hardcoded AWS regions

## Layer 3: Plan Validation (scripts/validate-plan.sh)
Verifies all expected resource types appear in the plan
Warns on unexpected resource destruction
Reports the total resource count to be created

## Layer 4: Security Scanning (.github/workflows/security-scan.yml, Extra Mile)
Runs Checkov against the Terraform config on every PR
soft_fail: false — a real finding blocks the merge, same as any other check
Skipped checks (CKV_AWS_18, CKV_AWS_119, CKV_AWS_144, CKV2_AWS_62, CKV2_AWS_61) require infrastructure outside this lab's scope — a separate access-logging bucket, a customer-managed KMS key, a cross-region replication target, an event notification target, and a lifecycle policy, respectively
Two findings were fixed directly instead of skipped: DynamoDB point-in-time recovery (CKV_AWS_28) and the S3 public access block (CKV2_AWS_6), since both were a single resource/argument addition and directly on-theme with the lab's security scenario

## CI Pipeline (.github/workflows/ci.yml)

Three jobs run on every PR and push to main:

static-analysis — fmt, validate, tflint
convention-checks — runs validate-conventions.sh
plan-validation — runs validate-plan.sh against live AWS credentials; depends on static-analysis passing first via needs:, so a broken config never triggers an AWS API call

## PR Test Report (.github/workflows/test-report.yml, Extra Mile)

Posts a markdown summary table as a PR comment after every run, showing pass/fail for format, validate, and convention checks at a glance — no need to dig through job logs.

## Branch Protection

A ruleset on main requires a pull request and requires Static Analysis, Convention Checks, and Plan Validation to pass before merging is allowed. Verified by opening a PR with intentionally broken formatting and confirming GitHub blocked the merge (mergeStateStatus: UNSTABLE) until fixed.

## Pre-Commit Hooks
Run ./scripts/install-hooks.sh to install
Writes a .git/hooks/pre-commit script (no external pre-commit tool dependency) that runs terraform fmt -check, terraform validate, and the convention script before every commit
.pre-commit-config.yaml is also included for teams using the standard pre-commit framework

## Key Learnings

Multiple test layers catch different categories of issues: syntax, style, organizational convention, and actual infrastructure drift. Static analysis is fast and catches errors early, before any AWS API call is made. Custom scripts enforce organization-specific standards that generic tools miss — naming, tagging, hardcoded values. Plan parsing validates the intended infrastructure changes before they hit real resources. Security scanning with Checkov catches misconfigurations neither tflint nor a custom script would think to check. Pre-commit hooks shift testing left to the developer's machine, and CI enforcement — backed by branch protection — ensures none of this is optional.

A practical lesson from building this: a new GitHub Actions workflow file only gets evaluated on pull_request events once it already exists on the base branch. The test-report.yml workflow had to be merged to main once before it would trigger on any subsequent PR.