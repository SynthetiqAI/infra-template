# Synthetiq infrastructure (template)

A starting point for a Synthetiq BYOI infrastructure repo: config in git, plan on
PR, apply on merge — no stored credentials. Click **Use this template** to create
your own repo, then follow the steps below. Full background:
[CI Integration](https://www.synthetiq.com/docs/platform-docs/deployments/byoi/ci-integration).

## What's here

- `.github/workflows/synthetiq-infra.yml` — calls the reusable
  [`SynthetiqAI/infra-workflow`](https://github.com/SynthetiqAI/infra-workflow) workflow.
- `package.json` + `.npmrc` — pin `@synthetiq/cli` from the Synthetiq private registry.
- `_infra/` — your config lives here (created by `synthetiq infra init`).

## Setup

1. **Install the CLI lockfile** (so CI's `npm ci` is reproducible):
   ```bash
   SYNTHETIQ_NPM_KEY=<your-key> npm install
   git add package-lock.json && git commit -m "chore: lockfile"
   ```
2. **Add the repo secret** `SYNTHETIQ_NPM_KEY` (Settings → Secrets and variables → Actions).
3. **Create the two AWS roles** trusting GitHub OIDC for this repo:
   - plan — `repo:<owner>/<repo>:pull_request`, policy from `synthetiq infra permissions --stage generate`
   - apply — `repo:<owner>/<repo>:ref:refs/heads/main`, policy from `synthetiq infra permissions --stage provision`
4. **Create the Synthetiq service account + trust** for the apply identity:
   ```bash
   synthetiq role list   # id of "CI Provision Apply"
   synthetiq service-account create <name> --role-id <role-id>
   synthetiq trust create \
     --service-account-id <service-account-id> \
     --issuer https://token.actions.githubusercontent.com \
     --subject "repo:<owner>/<repo>:ref:refs/heads/main"
   ```
5. **Fill in** `plan-role-arn`, `apply-role-arn`, and `organization-id` in
   `.github/workflows/synthetiq-infra.yml`.
6. **Author the config**:
   ```bash
   npx synthetiq infra init --domain apps.yourcompany.com
   git add _infra/synthetiq.yaml && git commit -m "infra: initial config"
   ```

Open a PR to see the plan; merge to apply.
