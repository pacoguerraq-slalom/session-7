# Golden Path CI

The todo-service adopts the reusable workflow in `.github/workflows/golden-path-ci.yml` through the small caller workflow at `.github/workflows/todo-service-ci.yml`. The caller runs for pull requests and pushes to `main`, while the reusable workflow owns the checks every adopting service receives.

## Required Checks

### Lint

The `lint` job installs the npm workspace dependencies and runs ESLint against both the Express backend and React frontend.

### Test

The `test` job runs the backend Jest suite with coverage. The backend package defines global 80% line and branch coverage thresholds, so Jest fails the job when coverage falls below the project standard. The resulting percentages are also written to the GitHub Actions job summary.

### Security Scan

When `run_terraform_plan` is enabled, `security-scan` runs Checkov against `infra/`. The scan is soft-fail during Steps 1 and 2 because the exercise consumes a shared golden-path module with findings owned by the platform team. Step 3 can tighten this gate after the infrastructure is prepared for deployment.

### Terraform Plan

When `run_terraform_plan` is enabled, `terraform-plan` installs the requested Terraform version, initializes the dev stack with `-backend=false`, validates the configuration, and records a plan summary. Backend-free initialization keeps pull requests independent of AWS credentials and the remote S3 state bucket. The plan itself is allowed to report the expected mock-provider AWS lookup failure until Step 3 enables OIDC.

### Docker Build

For pull requests, `docker-build` waits for lint and test, then builds the backend and frontend Dockerfiles. It does not push images or require AWS credentials.

## Adoption

A service team can call the workflow with this minimal file:

```yaml
name: Todo Service CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
```

The reusable workflow also accepts `terraform_version` and can disable the Terraform-dependent jobs by setting `run_terraform_plan: false`.

## Permissions and Secrets

Both workflow files declare explicit permissions. The caller requests `id-token: write`, and passes the `AWS_ROLE_ARN` repository secret to the reusable workflow as `aws_role_arn`. Terraform apply and image publishing use that secret to receive short-lived AWS credentials through OIDC only on pushes to `main`.