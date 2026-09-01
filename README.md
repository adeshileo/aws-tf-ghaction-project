# Personal Website — AWS · Terraform · GitHub Actions

A static personal/portfolio website for **Adeshile Osunkoya**, deployed to AWS S3 entirely through
Infrastructure as Code and a CI/CD pipeline. The site itself is intentionally simple — the point of
the repo is to demonstrate a clean, repeatable cloud deployment workflow.

## What this project shows

- **Infrastructure as Code** — the S3 bucket that hosts the site is defined in Terraform, not clicked
  together in the console.
- **Remote Terraform state** — state is stored in a shared S3 backend with DynamoDB state locking, so
  the pipeline is safe to run repeatedly and from different machines.
- **Keyless CI/CD** — GitHub Actions authenticates to AWS via OpenID Connect (OIDC) and assumes an IAM
  role. No long-lived AWS access keys are stored as secrets.
- **Automated deploys** — every push to `main` provisions/updates the infrastructure and syncs the
  site files to S3.

## Tech stack

| Layer            | Tooling                                             |
|------------------|----------------------------------------------------|
| Frontend         | Hand-written HTML + CSS (no framework, no build step) |
| Infrastructure   | Terraform (AWS provider)                            |
| State backend    | S3 + DynamoDB lock table                            |
| CI/CD            | GitHub Actions                                      |
| Cloud            | AWS S3, IAM (OIDC federation)                       |

## Repository layout

```
.
├── index.html                 # Site markup
├── styles.css                 # Site styles
├── headshot.jpg               # Profile image
├── terraform/
│   ├── provider.tf            # AWS provider + S3 remote state backend
│   ├── main.tf                # S3 bucket resource
│   ├── variables.tf           # Input variables (bucket_name)
│   └── output.tf              # Terraform outputs
└── .github/
    └── workflows/
        └── test.yaml          # Build + deploy pipeline
```

## How the pipeline works

On every push to `main` (or a manual `workflow_dispatch`):

1. **Checkout** the repository.
2. **Configure AWS credentials** by requesting a short-lived token via OIDC and assuming the IAM role
   in `vars.IAM_ROLE`.
3. **`terraform init`** — connect to the remote S3 state backend.
4. **`terraform plan`** — preview infrastructure changes.
5. **`terraform apply -auto-approve`** — create/update the S3 bucket.
6. **`aws s3 cp`** — upload `index.html`, `styles.css`, and `headshot.jpg` to the bucket.

### Required GitHub configuration

The workflow expects these repository-level **Variables** (`Settings → Secrets and variables → Actions → Variables`):

| Variable      | Purpose                                            |
|---------------|---------------------------------------------------|
| `IAM_ROLE`    | ARN of the IAM role GitHub Actions assumes via OIDC |
| `AWS_REGION`  | AWS region to operate in                            |
| `BUCKET_NAME` | Globally-unique name for the site's S3 bucket       |

The S3 backend for Terraform state (`bucket`, `dynamodb_table`) is configured in
[`terraform/provider.tf`](terraform/provider.tf) and must exist before the first run.

## Running Terraform locally

```bash
cd terraform
terraform init
terraform plan  -var="bucket_name=<your-bucket-name>"
terraform apply -var="bucket_name=<your-bucket-name>"
```

You'll need AWS credentials with permission to manage the bucket and access to the state backend.

## Notes

- The bucket is currently **private**; commented-out resources in
  [`terraform/main.tf`](terraform/main.tf) show the public-website / bucket-policy configuration used
  when serving the site publicly (e.g. via S3 static website hosting or a CDN in front of it).
