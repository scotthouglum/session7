# CI Pipeline

The Todo Service CI is split into a reusable workflow and a small caller workflow:

- `.github/workflows/golden-path-ci.yml` contains the platform team's shared CI/CD logic.
- `.github/workflows/todo-service-ci.yml` selects the triggers, inputs, and secrets for this service.

## Golden Path Workflow

`golden-path-ci.yml` is triggered with `workflow_call`, so other service repositories can reuse the same checks and deployment flow. It accepts Node.js and Terraform versions as inputs, along with flags controlling Terraform and image-publishing stages.

### Jobs

- **`lint`** checks both the backend and frontend with their workspace ESLint scripts. It catches style and static-analysis issues before code is merged.
- **`test`** installs dependencies and runs the backend Jest suite with coverage. Jest's repository configuration enforces the 80% global lines and branches threshold. The job also writes line, branch, function, and statement coverage to the GitHub Actions step summary.
- **`security-scan`** runs Checkov against `infra/` and fails on HIGH severity findings. It runs when `run_terraform_plan` is enabled so infrastructure security issues block the same workflow as the Terraform checks.
- **`terraform-plan`** installs the requested Terraform version, validates `infra/stacks/dev`, and creates a plan using the repository's AWS OIDC role. It writes a truncated human-readable plan to the step summary and uploads `tfplan` for a later apply. It runs when `run_terraform_plan` is enabled.
- **`docker-build`** builds both backend and frontend Dockerfiles on pull requests after `lint` and `test` pass. This validates that merge candidates can be containerized without pushing images or requiring AWS credentials.
- **`terraform-apply`** runs only when `run_terraform_apply` is enabled. It uses the uploaded plan and OIDC credentials to apply the development stack, then records the service URL in the step summary.
- **`build-and-push`** runs only when `build_and_push` is enabled and depends on `terraform-apply`. It resolves the ECR repositories, builds and pushes immutable SHA and `latest` tags for both images, and triggers an ECS service deployment.

All action references use version tags, and permissions are scoped to read repository contents, write pull-request results where needed, and request an OIDC token only for AWS jobs.

## Adoption

A service team creates a caller workflow that triggers on its supported events and invokes the reusable workflow. This is the minimum caller that enables the required plan and security checks:

```yaml
name: Todo Service CI

on:
  push:
    branches:
      - main
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
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

For a separate repository, replace the local `uses` path with the reusable workflow's repository reference and version. Set `run_terraform_apply: true` and `build_and_push: true` only for the protected deployment path, normally when the event is a push to `main`.

## Required Checks

The caller sets `run_terraform_plan: true`, which enables the four required golden-path checks:

| Check            | What it validates                                                                     | Why it is required                                                                               |
| ---------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `lint`           | ESLint for `packages/backend` and `packages/frontend`                                 | Prevents avoidable code-quality and static-analysis failures from reaching review or deployment. |
| `test`           | Backend Jest behavior and the configured 80% global coverage threshold                | Protects API behavior and requires meaningful regression coverage.                               |
| `security-scan`  | Checkov findings under `infra/`, failing on HIGH severity                             | Prevents known high-risk infrastructure misconfigurations from being approved.                   |
| `terraform-plan` | Terraform formatting-independent syntax/validation and the proposed dev-stack changes | Makes infrastructure changes reviewable and catches invalid plans before apply.                  |

`docker-build` is also required for pull requests by the reusable workflow's dependency chain, ensuring both deployable images build successfully. The deployment jobs are intentionally conditional and are not required for pull-request validation.

## OIDC Secret Configuration

The AWS role ARN is configured as a repository or organization secret named `AWS_ROLE_ARN`:

1. Create an AWS IAM role for GitHub Actions with a trust policy for GitHub's OIDC provider (`token.actions.githubusercontent.com`). Restrict the subject to the intended repository and branch or environment.
2. Grant the role only the AWS permissions needed for Terraform plan/apply and the deployment steps.
3. In GitHub, open **Settings > Secrets and variables > Actions** and add `AWS_ROLE_ARN` as an Actions secret containing the role ARN.
4. Keep `id-token: write` in the caller's top-level `permissions` block. Without it, GitHub does not issue the OIDC token to the reusable workflow.

The caller passes the repository secret as `secrets.aws_role_arn`. Inside `golden-path-ci.yml`, the `terraform-plan` job consumes that workflow-call secret as `secrets.aws_role_arn` when configuring `aws-actions/configure-aws-credentials`. The ARN is never hardcoded in the workflow.
