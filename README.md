# dotnet-sample-app-config

Release configuration for **dotnet-sample-app** across `dev`, `uat`, `prod`.

This repo carries **deploy values** — which artifact version runs in each
environment, with which knobs, and against which AWS resources. It contains no
deployment logic.

| Concern | Where it lives |
|---------|----------------|
| Build + publish the artifact | `dotnet-sample-app` (CI) |
| **Release values + AWS resource coordinates (this repo)** | `deploy/<env>.yml` (`infra:` block) |
| OIDC deploy role + the AWS resources themselves | created out of band per env (console / Terraform / CloudFormation) |
| Deploy logic | `enterprise-ci-templates/.github/workflows/cd-template.yml` |
| Central CD defaults (accounts, regions, strategy) | `enterprise-ci-templates/policy/deploy-defaults.yml` |

## Layout

```
deploy/
  dev.yml  uat.yml  prod.yml     # per-env release values (contract: cd/v1)
.github/workflows/
  deploy.yml     # caller → cd-template.yml
  validate.yml   # PR: schema-validate deploy/*.yml
  promote.yml    # dev→uat→prod, carries the same version forward
CODEOWNERS       # prod.yml → Platform / SRE
```

## `deploy/<env>.yml`

Validated against
`enterprise-ci-templates/policy/schema/deploy-config.schema.json`. Key fields:

| Field | Notes |
|-------|-------|
| `service.target` | `ecs` \| `ec2` \| `lambda` \| `s3` — picks the branch of `cd-template.yml` |
| `release.version` | image digest / zip sha / bundle id. **Empty allowed only for `dev`** (deploys latest) |
| `release.strategy` | `rolling` \| `blue-green` \| `canary` — falls back to the env default if omitted |
| `runtime.env` | non-secret env vars |
| `runtime.secrets` | secret **names** only — resolved from Secrets Manager at deploy time |
| `infra.<target>` | AWS resource names for the deploy (ECS cluster/service, ECR repo, CodeDeploy app/group, Lambda fn/alias, S3 bucket/distribution). Only the block matching `service.target` is required. |

AWS account, region and the deploy-role ARN come from `deploy-defaults.yml`
(patterns) — everything else is in the `infra:` block above.

> `release.version` for each env is updated automatically by `cd-template.yml`
> on every successful deploy (committed back with `[skip ci]`).

## How it deploys

1. **dev** — `dotnet-sample-app` CI finishes a green build and sends a
   `repository_dispatch` (`artifact-published`) with the new version →
   `deploy.yml` auto-deploys dev.
2. **uat / prod** — run **Actions → deploy** (or **promote**) with the version to
   promote. The GitHub Environment protection rules (required reviewers, wait
   timer, `main`-only) gate the run. Same artifact flows forward — never rebuilt.

## One-time setup

- **GitHub Environments** `dev`, `uat`, `prod`. Add required reviewers + a wait
  timer to `uat` and `prod`; restrict `prod` to the `main` branch.
- Per environment, create: the AWS resources (ECS service/cluster, ECR repo,
  CodeDeploy app/group for blue-green, …) and the OIDC deploy role
  `gha-deploy-dotnet-sample-app` (trust scoped to
  `repo:aba-d/dotnet-sample-app-config:environment:<env>`). Then put their names
  into `deploy/<env>.yml` under `infra:`.
- **`dotnet-sample-app` CI** must send the `artifact-published` dispatch to this
  repo on a green build.

## Validate locally

```bash
cd ../enterprise-ci-templates/actions && npm ci
node -e '
  const {readYaml,validate}=require("./shared/lib");
  for (const e of ["dev","uat","prod"])
    validate(readYaml(`../../dotnet-sample-app-config/deploy/${e}.yml`),
             "../policy/schema/deploy-config.schema.json", `${e}.yml`);
  console.log("all valid");
'
```
