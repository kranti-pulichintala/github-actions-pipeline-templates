# GitHub Actions Pipeline Templates

A set of reusable GitHub Actions workflows (`workflow_call`) that standardize the most common DevOps pipelines: building and scanning container images, planning and applying Terraform with PR feedback, and running Python CI with lint and coverage. Built as a reference architecture for teams that want consistent, secure, and reviewable delivery pipelines.

## Why it matters

Most teams copy-paste CI YAML between repositories. That spreads drift: one repo pushes images without scanning, another applies Terraform without a plan review, and a third pins an outdated Python version. Reusable workflows fix this by keeping the pipeline logic in one place. Callers pass a few inputs and inherit the security defaults: registry login, image vulnerability scanning with Trivy, Terraform plans posted as PR comments, and linted, covered Python tests across a version matrix.

## Prerequisites

- A GitHub repository or organization where you can add workflows.
- For the Docker template: write access to a container registry. The default is GitHub Container Registry (`ghcr.io`); the built-in `GITHUB_TOKEN` handles authentication. For Amazon ECR, see the commented alternative login step in the workflow.
- For the Terraform template: an AWS account with an IAM role that trusts GitHub's OIDC provider, plus repository or organization secrets for the role ARN. No long-lived AWS keys are needed.
- For the Python template: a Python project with a test suite that runs under `pytest`.

## Step-by-step usage

1. Copy the `.github/workflows/` directory from this repository into the repository that should consume the templates, or publish this repository as its own and reference the workflow paths with a versioned ref.
2. In the consuming repository, create a caller workflow under `.github/workflows/` for each template you want to use. Copy one of the examples below and adjust the inputs.
3. For the Terraform template, add the `AWS_ROLE_TO_ASSUME` secret (or pass `role_to_assume` as an input) and create a `production` environment in the repository settings if you want apply protection rules.
4. Open a pull request to see the plan comment, merge to `main` to trigger Docker builds or Terraform applies, and check the Actions tab for lint and coverage results.

## Templates

| Template | Purpose | Required inputs | Optional inputs | Secrets |
| - | - | - | - | - |
| `reusable-docker-build.yml` | Build, push, and vulnerability-scan a container image | `image_name` | `dockerfile` (default `Dockerfile`), `context` (default `.`), `registry` (default `ghcr.io`), `tag` (default `sha`) | `GITHUB_TOKEN` (provided automatically) |
| `terraform-plan-apply.yml` | Terraform plan on pull requests with a PR comment, apply on push to main | none | `working_directory` (default `.`), `terraform_version` (default `1.9.0`), `aws_region` (default `us-east-1`), `role_to_assume` | `AWS_ROLE_TO_ASSUME` (IAM role for OIDC) |
| `python-ci.yml` | Lint, format check, test, and upload coverage across Python versions | none | `python_versions` (default `["3.11", "3.12"]`), `working_directory` (default `.`), `requirements_file` (default `requirements.txt`) | none |

## Caller workflow examples

### Docker build caller

Copy this into `.github/workflows/docker.yml` in the consuming repository:

```yaml
name: Docker Build

on:
  push:
    branches: [main]

jobs:
  build:
    uses: your-org/github-actions-pipeline-templates/.github/workflows/reusable-docker-build.yml@v1
    with:
      image_name: myorg/api
      dockerfile: Dockerfile
      context: .
      registry: ghcr.io
      tag: sha
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The image is pushed only when the workflow runs on `main`. On other refs the workflow still builds (and caches) without pushing, which keeps feature-branch runs fast and safe.

### Terraform caller

Copy this into `.github/workflows/terraform.yml` in the consuming repository:

```yaml
name: Terraform

on:
  pull_request:
  push:
    branches: [main]

jobs:
  terraform:
    uses: your-org/github-actions-pipeline-templates/.github/workflows/terraform-plan-apply.yml@v1
    with:
      working_directory: infra
      terraform_version: 1.9.0
      aws_region: us-east-1
    secrets:
      AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE_TO_ASSUME }}
```

Pull requests run plan and post the output as a PR comment. Pushes to `main` run apply against the `production` environment.

### Python CI caller

Copy this into `.github/workflows/python-ci.yml` in the consuming repository:

```yaml
name: Python CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  python-ci:
    uses: your-org/github-actions-pipeline-templates/.github/workflows/python-ci.yml@v1
    with:
      python_versions: '["3.11", "3.12"]'
      working_directory: .
      requirements_file: requirements.txt
```

Tests run with `pytest`, coverage is collected with `pytest-cov`, and the coverage XML report is uploaded as an artifact named `coverage-report-<version>`.

## File layout

- `README.md` - Project overview, prerequisites, usage steps, template table, and caller examples.
- `.github/workflows/reusable-docker-build.yml` - Reusable workflow that builds a container image with Buildx, pushes it to a registry on `main`, scans it with Trivy, and uploads the SARIF report to code scanning. Includes a commented-out Amazon ECR login step as an alternative.
- `.github/workflows/terraform-plan-apply.yml` - Reusable workflow that runs Terraform init, validate, and plan on pull requests and posts the plan as a PR comment, then runs apply on pushes to `main` under a `production` environment. AWS authentication uses OIDC.
- `.github/workflows/python-ci.yml` - Reusable workflow that runs Ruff lint and format checks plus `pytest` with coverage across a matrix of Python versions, uploading the coverage report as an artifact.

## Notes

- These workflows pin their third-party actions to major versions for stability. Pin to commit SHAs in high-compliance environments.
- Replace `your-org` in the caller examples with the organization or user that hosts the templates repository.
