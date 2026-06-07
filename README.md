# terraform-aws-litellm

> **This is a read-only mirror.** Source of truth is [`BerriAI/litellm//terraform/litellm/aws/`](https://github.com/BerriAI/litellm/tree/main/terraform/litellm/aws). Open issues and PRs there — anything pushed directly to this repo will be overwritten on the next release.

Terraform module that deploys [LiteLLM](https://github.com/BerriAI/litellm) on AWS — gateway, backend, and UI on ECS Fargate, Aurora Postgres with IAM auth, ElastiCache Redis, S3, and an Application Load Balancer with path-based routing. Each release tag here is mirrored from a `BerriAI/litellm` release.

## Usage

```hcl
module "litellm" {
  source  = "BerriAI/litellm/aws"
  version = "~> 1.89"

  region = "us-west-2"
  azs    = ["us-west-2a", "us-west-2b"]
  tenant = "acme"
  env    = "prod"
}
```

Then:

```bash
terraform init
terraform apply
```

### Required inputs

| Name | Description |
|---|---|
| `region` | AWS region. Single-region module — fan out across regions with one root config per region. |
| `azs` | List of AZs to spread subnets across (at least 2). |
| `tenant` | Identifier prefix for every resource (`${tenant}-litellm-${env}`). |
| `env` | Environment slug (`prod`, `stage`, `dev`, ...). |

All other inputs have safe defaults — full list in [`variables.tf`](./variables.tf) or the auto-generated [registry page](https://registry.terraform.io/modules/BerriAI/litellm/aws/latest?tab=inputs).

### Common knobs

- **TLS**: set `acm_certificate_arn` for HTTPS; the ALB falls back to HTTP/80 only when `allow_plaintext_alb = true` (dev/trial use only).
- **Proxy config**: pass YAML as a typed map via `proxy_config` (mirrors the [Helm chart](https://github.com/BerriAI/litellm/tree/main/helm/litellm)'s `gateway.config.proxy_config`).
- **Provider API keys**: store in AWS Secrets Manager, reference ARNs via `gateway_extra_secrets` / `backend_extra_secrets`.
- **Image pins**: defaults to `ghcr.io/berriai/litellm-<component>:main-stable`. Pin to a specific tag for production via `gateway_image` / `backend_image` / `ui_image` / `migrations_image`.

## Architecture

```
Public ALB (HTTP/80)
  ├── LLM data-plane prefixes (/v1/chat/*, /v1/embeddings, ...) → ECS Fargate (gateway)
  ├── UI assets (/, /_next/*, /litellm-asset-prefix/*)         → ECS Fargate (ui)
  └── everything else (management: /key/*, /user/*, ...)        → ECS Fargate (backend)

Private subnets:
  Aurora Postgres (writer + reader, IAM auth)
  ElastiCache Redis (encrypted, multi-AZ)
  S3 bucket (versioned)
  Secrets Manager (LITELLM_MASTER_KEY, DB password, your API keys)
  One-off ECS task: prisma migrate deploy
```

## Migration job

The proxy runs `prisma migrate deploy` at startup, but on first apply the
gateway/backend can race the empty DB. The module exposes a one-off ECS
task (`litellm-migrations`) that runs the migration; `terraform apply`
auto-runs it via `local-exec` (requires the `aws` CLI on the apply
machine). To run manually, see the `migration_run_command` output.

## Issues & contributions

File issues at [BerriAI/litellm](https://github.com/BerriAI/litellm/issues). PRs against the Terraform module go to [`BerriAI/litellm`](https://github.com/BerriAI/litellm) under `terraform/litellm/aws/`.

## License

Same as [BerriAI/litellm](https://github.com/BerriAI/litellm/blob/main/LICENSE).
