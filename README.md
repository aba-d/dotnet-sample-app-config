# dotnet-sample-app-config

Release configuration for **dotnet-sample-app** across `dev`, `uat`, `prod`.

This repo carries **values only** — which artifact version runs in each
environment and with which knobs. It contains no deployment logic and no AWS
resource definitions.

| Concern | Where it lives |
|---------|----------------|
| Build + publish the artifact | `dotnet-sample-app` (CI) |
| AWS resources + OIDC deploy role | `dotnet-sample-app-infra` (IaC) → coordinates published to SSM `/dotnet-sample-app/<env>/…` |
| **Release values (this repo)** | `deploy/<env>.yml` |
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

AWS account, region, deploy-role ARN and resource coordinates are **not** set
here — they come from `deploy-defaults.yml` and SSM.

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
- **`dotnet-sample-app-infra`** must have created, per environment:
  the OIDC deploy role `gha-deploy-dotnet-sample-app` (trust scoped to
  `repo:aba-d/dotnet-sample-app-config:environment:<env>`) and the SSM
  parameters under `/dotnet-sample-app/<env>/…`.
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
