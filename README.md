# infra-workflow

Reusable GitHub Actions workflow for Synthetiq BYOI infrastructure: **plan on PR,
apply on merge**, with no long-lived credentials.

Your infra repo calls it (see [`SynthetiqAI/infra-template`](https://github.com/SynthetiqAI/infra-template) for a ready-to-use starting point):

```yaml
name: synthetiq-infra
on:
  pull_request:
    paths: ["_infra/**"]
  push:
    branches: [main]
    paths: ["_infra/**"]
permissions:
  id-token: write       # OIDC for AWS and Synthetiq
  contents: write       # plan commits the reviewed changeset to the PR branch
  pull-requests: write  # the plan comment
jobs:
  infra:
    uses: SynthetiqAI/infra-workflow/.github/workflows/plan-apply.yml@v1
    with:
      plan-role-arn:   arn:aws:iam::111122223333:role/synthetiq-infra-plan
      apply-role-arn:  arn:aws:iam::111122223333:role/synthetiq-infra-apply
      organization-id: <your-org-id>
    secrets: inherit
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `plan-role-arn` | yes | — | AWS role for the plan (change-set verbs only) |
| `apply-role-arn` | yes | — | AWS role for the apply (full provisioning policy) |
| `organization-id` | yes | — | Synthetiq org the apply provisions into |
| `aws-region` | no | `us-east-1` | Region for STS/role assumption |
| `oidc-audience` | no | `https://api.synthetiq.com` | Audience the Synthetiq trust verifies the GitHub token against |
| `synthetiq-login-url` | no | `https://login.synthetiq.com` | Synthetiq OAuth base URL |
| `node-version` | no | `22` | Node version |

## Secrets

| Secret | Required | Description |
|---|---|---|
| `SYNTHETIQ_NPM_KEY` | yes | Auth for the Synthetiq private npm registry, to install `@synthetiq/cli`. Pass with `secrets: inherit`. |

## Caller prerequisites

- `package.json` pinning `@synthetiq/cli` and an `.npmrc` for the private registry (the template provides both).
- Two AWS roles trusting GitHub OIDC for the caller repo — `plan` on `…:pull_request`, `apply` on `…:ref:refs/heads/main` — with policies from `synthetiq infra permissions --stage generate|provision`.
- A Synthetiq service account + OIDC trust on `repo:<owner>/<repo>:ref:refs/heads/main`.

The OIDC subject is always the **caller** repo, so trusts key off the consumer repo — this shared workflow never widens anyone's access.

## Versioning

Consumers pin a tag in `uses: …@v1`. This repo keeps two kinds of tags:

- **`vX.Y.Z`** — immutable releases (e.g. `v1.0.0`). Pin here for zero surprises.
- **`v1`** — a floating major alias that always points at the latest `v1.x.y`. The
  docs/template pin this so consumers pick up fixes without editing their wrapper.

### Releasing

```bash
# from a clean main with the change committed:
git tag -a v1.0.1 -m "v1.0.1: <summary>"   # immutable release
git push origin v1.0.1
git tag -f v1 v1.0.1^{}                     # move the floating major alias
git push -f origin v1
```

Bump the major (`v2`, and a new `v2.0.0`) only for breaking changes to the
inputs/secrets contract; leave `v1` in place so existing callers keep working.
