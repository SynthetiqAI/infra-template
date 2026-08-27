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
   - plan — subject `repo:<owner>/<repo>:pull_request`, policy from `synthetiq infra permissions --stage generate`
   - apply — subject `repo:<owner>/<repo>:ref:refs/heads/main`, policy from `synthetiq infra permissions --stage provision`

   > **Check which subject your organization sends.** Some GitHub organizations emit
   > *ID-qualified* subject claims, appending the numeric org and repo ids:
   > `repo:<owner>@<org-id>/<repo>@<repo-id>:pull_request`. IAM compares the subject
   > exactly, so a trust built from the plain form is denied with a bare
   > `Not authorized to perform sts:AssumeRoleWithWebIdentity` — the Actions log never
   > says a claim comparison is what failed. List **both** forms in the trust policy's
   > `StringEquals` condition (a list is an OR, and both are exact strings, so this
   > widens nothing) and the role works whichever form your org sends:
   >
   > ```json
   > "token.actions.githubusercontent.com:sub": [
   >   "repo:<owner>@<org-id>/<repo>@<repo-id>:pull_request",
   >   "repo:<owner>/<repo>:pull_request"
   > ]
   > ```
   >
   > To read the subject your org actually sends, look at the CloudTrail
   > `AssumeRoleWithWebIdentity` event for the failed attempt — the full presented
   > subject is in `userIdentity.principalId`.
4. **Create the Synthetiq service account + trust** for the apply identity:
   ```bash
   synthetiq role list   # id of "CI Provision Apply"
   synthetiq service-account create <name> --role-id <role-id>
   synthetiq trust create \
     --service-account-id <service-account-id> \
     --issuer https://token.actions.githubusercontent.com \
     --subject "repo:<owner>/<repo>:ref:refs/heads/main"
   ```

   The subject caveat from step 3 applies here too, with a different error. If your
   organization sends ID-qualified claims, the token exchange fails with
   `invalid_grant: No OIDC trust is configured for this issuer and subject in the
   specified organization`. Create a second trust on the same service account for the
   ID-qualified form — one trust per (issuer, subject) pair, both mapping to the same
   service account:

   ```bash
   synthetiq trust create \
     --service-account-id <service-account-id> \
     --issuer https://token.actions.githubusercontent.com \
     --subject "repo:<owner>@<org-id>/<repo>@<repo-id>:ref:refs/heads/main"
   ```
5. **Fill in** `plan-role-arn`, `apply-role-arn`, and `organization-id` in
   `.github/workflows/synthetiq-infra.yml`.
6. **Author the config**:
   ```bash
   npx synthetiq infra init --domain apps.yourcompany.com
   git add _infra/synthetiq.yaml && git commit -m "infra: initial config"
   ```

Open a PR to see the plan; merge to apply.

## Nested layout

By default the infra root sits at the repo root. To keep it under a subdirectory
of a larger repo — e.g. a shared infrastructure monorepo:

```
your-repo/
└── partner/
    └── synthetiq/          ← the infra root
        ├── package.json
        ├── package-lock.json
        ├── .npmrc
        └── _infra/
            └── synthetiq.yaml
```

Move `package.json`, `package-lock.json`, `.npmrc`, and `_infra/` together into
that directory (they resolve as a unit), then make two edits in
`.github/workflows/synthetiq-infra.yml`:

1. **Set `working-directory`** in the `with:` block to that directory:
   ```yaml
   with:
     ...
     working-directory: partner/synthetiq
   ```
2. **Prefix the trigger `paths:`** with the same directory. GitHub Actions path
   filters are static globs — they can't read the input, so they must be edited
   by hand or CI won't run on your config changes:
   ```yaml
   pull_request:
     paths: ["partner/synthetiq/_infra/**", "partner/synthetiq/package.json", "partner/synthetiq/package-lock.json"]
   push:
     branches: [main]
     paths: ["partner/synthetiq/_infra/**", "partner/synthetiq/package.json", "partner/synthetiq/package-lock.json"]
   ```

Run the CLI (`npm install`, `synthetiq infra init`, etc.) from inside that
directory — it discovers `_infra/` by walking up, so any cwd at or below the
infra root works.
