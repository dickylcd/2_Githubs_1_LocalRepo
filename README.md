## Goal and safety model
While you've local repo code (e.g. IaC code) and willing to host at two different Github repos (e.g. Personal and Office). You can follow below steps to achieve this goal.

Your personal GitHub repository is already synchronized with local `IaC_Surce` and is the source of truth. You will make a backup, scan the full local Git history for secrets, then replace **only the office repository’s target branch** with the personal branch using a narrowly scoped `--force-with-lease` push. A mirror clone preserves repository history and can be restored by pushing it back to a remote. [docs-internal.github](https://docs-internal.github.com/en/repositories/archiving-a-github-repository/backing-up-a-repository)

```text
Local IaC_Surce = Personal GitHub = source of truth
                      │
                      └── controlled branch replacement ──► Office GitHub
```

## 1. Back up the office repository

Run these commands **outside** the `IaC_Surce` working directory:

```bash
mkdir -p ~/github-backups
cd ~/github-backups

git clone --mirror \
  git@github-office.com:YOUR_OFFICE_ORG/IaC_Surce.git \
  office-IaC_Surce-backup.git

tar -czf office-IaC_Surce-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  office-IaC_Surce-backup.git
```

Keep both the mirror directory and the compressed archive until the office repository has been verified. GitHub explicitly recommends `git clone --mirror` for backing up a repository, including its revision history. [docs-internal.github](https://docs-internal.github.com/en/repositories/archiving-a-github-repository/backing-up-a-repository)

## 2. Confirm personal is current

In your local repository:

```bash
cd IaC_Surce

git status
git fetch origin --prune
git switch main
git pull --ff-only origin main
```

If your personal repository uses `master`, replace `main` with `master` in every command.

Verify that local `main` exactly matches personal `origin/main`:

```bash
git rev-parse HEAD
git rev-parse origin/main
```

Both commands must print the same commit hash. If they differ, resolve that discrepancy before touching the office repository.

## 3. Add and inspect the office remote

If the office remote is not already configured:

```bash
git remote add office git@github-office.com:YOUR_OFFICE_ORG/IaC_Surce.git
```

Fetch it and identify its actual branch name:

```bash
git fetch office --prune

git remote -v
git branch -r
git remote show office
git ls-remote --heads office
```

Do not assume the office branch is `main`. If `git ls-remote --heads office` shows `refs/heads/master`, set:

```bash
PERSONAL_BRANCH=main
OFFICE_BRANCH=master
```

Otherwise, for the common case:

```bash
PERSONAL_BRANCH=main
OFFICE_BRANCH=main
```

Verify that both refs resolve:

```bash
git rev-parse --verify "origin/${PERSONAL_BRANCH}"
git rev-parse --verify "office/${OFFICE_BRANCH}"
```

Each command must print a commit hash.

## 4. Scan full Git history for secrets

This is the corrected step. Your installed Gitleaks supports the `detect` interface, so run:

```bash
gitleaks detect --source . --verbose --redact
```

This scans the Git repository’s history—not merely the files currently visible in your working directory. Older Gitleaks documentation identifies `detect` as the command for scanning repositories and explains that it parses `git log -p` output. 

For an auditable report:

```bash
gitleaks detect \
  --source . \
  --verbose \
  --redact \
  --report-format json \
  --report-path gitleaks-report.json
```

Prevent accidental commit of the report:

```bash
grep -qx 'gitleaks-report.json' .gitignore || echo 'gitleaks-report.json' >> .gitignore
```

Also inspect likely sensitive filenames:

```bash
git ls-files | grep -Ei '(^|/)(\.env|\.tfvars|secrets?|credentials|id_rsa|\.pem|\.key)'
```

If any finding is real:

1. Stop; do not push to the personal repository.
2. Revoke and rotate every exposed credential immediately.
3. Remove it from current files and prevent recurrence.
4. Move secrets into a secrets manager, environment variables, or a Terraform/OpenTofu secrets backend.
5. Re-run `gitleaks detect --source . --verbose --redact`.
6. Decide whether prior Git history must be rewritten; deleting a secret in a later commit does not remove it from earlier commits.

## 5. Record the office branch before overwrite

Save the current office branch commit ID for recovery and audit purposes:

```bash
git rev-parse "office/${OFFICE_BRANCH}"
```

Example:

```text
a1b2c3d4e5f6...
```

Also retain the mirror backup created in Step 1. This is the only straightforward recovery path if the overwrite needs to be reversed.

## 6. Overwrite only the office branch

Use an explicit refspec. This replaces exactly one branch and does not affect other office branches, tags, releases, or repository settings:

```bash
git push --force-with-lease \
  "office" \
  "+${PERSONAL_BRANCH}:${OFFICE_BRANCH}"
```

For the common case where both branches are `main`:

```bash
git push --force-with-lease office +main:main
```

Explanation:

- `+` permits the intentional non-fast-forward replacement.
- `main:main` means “push local/personal `main` to office `main`.”
- `--force-with-lease` checks that the office branch still matches the remote-tracking reference Git observed when you fetched it, reducing the risk of overwriting an unexpected newer commit. [github](https://github.com/scolladon/tsgit/blob/main/docs/adr/015-push-force-with-lease.md)

Do **not** use any of these broader commands:

```bash
git push --force office
git push --force --all office
git push --force --tags office
```

## 7. Handle branch protection

If GitHub rejects the push because `main` is protected, GitHub blocks force pushes on protected branches by default. Branch protection rules can allow force pushes only for selected users or teams, and enabling force pushes does not remove other protection requirements. 

Recommended approach:

1. Ask the office repository administrator to grant temporary force-push permission to only your account or a narrowly scoped team.
2. Perform the single branch-specific push in Step 6.
3. Immediately restore the original branch-protection configuration.
4. Do not permanently disable required reviews, status checks, or other protections.

## 8. Verify the overwrite

Fetch the updated office state and compare commit hashes:

```bash
git fetch office --prune

git rev-parse "origin/${PERSONAL_BRANCH}"
git rev-parse "office/${OFFICE_BRANCH}"
```

The hashes must be identical.

Then verify that the file trees match:

```bash
git diff --stat "origin/${PERSONAL_BRANCH}...office/${OFFICE_BRANCH}"
```

No output means the file trees are identical.

Finally, check the office repository in GitHub:

- The default branch shows the personal repository’s latest commit message and hash.
- CI/CD or Terraform/OpenTofu plan workflows complete successfully, if configured.
- No unrelated branches, tags, releases, or repository settings were changed.

## 9. Future synchronization

After this one-time overwrite, ordinary pushes should work whenever the histories remain aligned:

```bash
git push origin main
git push office main
```

If you later rewrite personal history, repeat only the controlled branch-specific overwrite:

```bash
git push --force-with-lease office +main:main
```

Before every push, run IaC checks appropriate to your stack:

```bash
terraform fmt -check
terraform validate
# or, for OpenTofu:
tofu fmt -check
tofu validate
```

## Optimized prompt

```text
Act as an experienced GitHub, Git, and infrastructure-as-code security expert.

I have one local IaC repository named IaC_Surce. My personal GitHub repository is already fully synchronized with the local repository and is the authoritative source of truth. I need to overwrite the existing office GitHub repository with the personal repository’s current branch history.

Please provide the safest, fastest, complete step-by-step solution for macOS/Linux and Windows Git Bash.

Requirements:
- Start with a complete mirror backup of the existing office GitHub repository and explain how to restore it.
- Configure separate SSH keys and SSH config aliases for the personal and office GitHub accounts.
- Use `origin` for the personal repository and `office` for the office repository.
- Do not assume the default branch is `main`; discover actual branch names using `git remote show` and `git ls-remote --heads`.
- Verify that the local branch exactly matches the personal repository before overwriting office.
- Scan the complete Git history for secrets before publishing internal IaC code to the personal account.
- Use the Gitleaks command compatible with my installed version:
  gitleaks detect --source . --verbose --redact
- Explain that `gitleaks detect` scans Git history through `git log -p`, not only current working-directory files.
- Include an optional JSON report command and ensure the report is excluded from Git.
- If secrets are found, provide an incident-response workflow: stop the push, rotate credentials, remove secrets from future commits, use a secrets manager, and assess whether history rewriting is required.
- Overwrite only the office repository’s target branch using an explicit refspec and `git push --force-with-lease`; explicitly prohibit broad `git push --force`, `--all`, or `--tags`.
- Explain how to handle GitHub branch protection, including temporary narrowly scoped force-push permission and immediate restoration of protections.
- Verify success by comparing commit hashes and file trees between personal and office.
- Explain how to handle unrelated Git histories and how to recover from the mirror backup.
- Include Terraform/OpenTofu validation, `.gitignore`, and secret-hygiene checks before pushing.
- Include troubleshooting for permission denied, protected branches, wrong SSH identity, non-fast-forward rejection, and accidental overwrite recovery.
- Use placeholders: YOUR_PERSONAL_USERNAME, YOUR_OFFICE_ORG, IaC_Surce, PERSONAL_BRANCH, and OFFICE_BRANCH.
- Prioritize safety, auditability, minimal disruption, and explicit verification.
```

The corrected end-to-end command sequence is: **mirror-backup office → verify personal is current → fetch and identify office branch → `gitleaks detect --source . --verbose --redact` → record office commit → `git push --force-with-lease office +PERSONAL_BRANCH:OFFICE_BRANCH` → verify hashes and file trees**. [docs-internal.github](https://docs-internal.github.com/en/repositories/archiving-a-github-repository/backing-up-a-repository)
