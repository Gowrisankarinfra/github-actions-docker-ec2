# CI/CD: GitHub Actions → Docker → Amazon ECR → EC2 (SSM)

## Goal
On a push to `main`, GitHub Actions tests and builds a Docker image, authenticates to
Amazon ECR using GitHub OIDC, pushes the image, and invokes deployment on an EC2 host
through AWS Systems Manager Run Command.

The EC2 host must already exist, have Docker installed, be a managed SSM instance,
and have network access to ECR and SSM endpoints. This repository does not provision
that host.

## Flow

```mermaid
flowchart LR
  Dev[Git push to main] --> GH[GitHub Actions]
  GH --> OIDC[GitHub OIDC role]
  OIDC --> ECR[Amazon ECR]
  GH --> SSM[AWS Systems Manager]
  SSM --> EC2[EC2 deployment host]
  EC2 --> Docker[Pull and run container]
```

## GitHub repository setup
Create these GitHub Actions repository variables:
- `AWS_REGION` – e.g. `ap-south-1`
- `AWS_ACCOUNT_ID` – your 12-digit AWS account ID
- `ECR_REPOSITORY` – ECR repository name
- `EC2_INSTANCE_ID` – target managed EC2 instance ID

Create this repository secret:
- `AWS_ROLE_ARN` – IAM role ARN trusted for GitHub OIDC

Do not use long-lived AWS access keys in GitHub secrets. Configure an IAM OIDC
provider and restrict the role trust policy to your repository and branch. The role
needs ECR push permissions and permission to send SSM commands to the chosen instance.

## EC2 host preparation
1. Use a supported Linux AMI and attach an instance role with
   `AmazonSSMManagedInstanceCore` plus read access to the specific ECR repository.
2. Install and start Docker.
3. Confirm the instance is online in Systems Manager.
4. Ensure outbound connectivity to ECR, SSM, and required AWS endpoints.
5. Create the ECR repository.

## Deploy
1. Update workflow variables/secrets in GitHub.
2. Review `.github/workflows/deploy.yml`.
3. Push a change to `main`.
4. Inspect the Actions run and verify the new image tag in ECR.
5. Verify the app on the EC2 host and inspect `docker ps` / container logs.

## Local test

```bash
docker build -t portfolio-web:local .
docker run --rm -p 8080:80 portfolio-web:local
```

Open `http://localhost:8080`.

## Rollback
The workflow uses the Git commit SHA as an immutable image tag. To roll back, run
the previous successful workflow or manually deploy a previously published image tag.
Do not delete images needed for rollback.

## Security notes
Use GitHub OIDC rather than static AWS keys. Scope IAM to one repository, branch,
ECR repository, and EC2 instance. Protect `main` with branch rules. The sample web
container is intentionally minimal and should not be used as a production app.
