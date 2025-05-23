# Copilot Githooks Tutorial

This tutorial shows you how to enforce a **“repo-first”** patch guarantee for both human- and Copilot-generated changes. It covers:

1. Embedding the policy in `CONTRIBUTING.md`  
2. Adding a shareable pre-commit hook  
3. Enabling the hook locally  
4. Adding a GitHub Actions CI check  
5. Extending the hook to other file types (e.g. `.json`)  
6. Understanding `runs-on: ubuntu-latest` vs. self-hosted runners

---

## 1. Document the policy in CONTRIBUTING.md

Create or update the file `CONTRIBUTING.md` at your repo root with a **Repo-first patch rule**:

```markdown
# Contributing

[...]

## Repo-first patch rule

All code patches—whether human-written or AI-generated—**must**:

1. Be generated against the **current** `HEAD` of the target file.  
2. Include at least three lines of unchanged context before and after each hunk.  
3. Apply cleanly with `git apply --check`.  
4. Fail CI or local checks if any hunk cannot be applied to `HEAD`.

Please do **not** edit from memory or stale copies.
```

---

## 2. Add a sharable pre-commit hook

1. Create a versioned hooks folder:

   ```bash
   mkdir -p .githooks
   ```

2. Add the script `.githooks/validate-copilot-patch`:

   ```bash name=.githooks/validate-copilot-patch
   #!/usr/bin/env bash
   # ensure patches apply cleanly to HEAD and include context
   set -e

   # only run on staged .py or .json files
   if git diff --cached --name-only | grep -E '\.(py|json)$' >/dev/null; then
     # generate a patch of staged changes
     git diff --cached > /tmp/copilot.patch

     # check that no patch hunks fail to apply against HEAD
     if ! git apply --check /tmp/copilot.patch; then
       echo "ERROR: Patch must apply cleanly against HEAD." >&2
       exit 1
     fi

     # verify each hunk has at least one original-context (“- ”) line
     if grep -E '^@@' /tmp/copilot.patch | grep -vE '^- ' >/dev/null; then
       echo "ERROR: Patch hunks missing context. Always include at least 3 lines of context from HEAD." >&2
       exit 1
     fi
   fi
   ```

3. Make it executable and commit:

   ```bash
   chmod +x .githooks/validate-copilot-patch
   git add .githooks/validate-copilot-patch
   git commit -m "chore: add repo-first patch validation hook"
   ```

---

## 3. Enable the hook locally

Configure your clone to use the versioned hooks folder:

```bash
git config core.hooksPath .githooks
```

From now on, every `git commit` (or `git commit --dry-run`) will trigger `validate-copilot-patch`.

---

## 4. Add a GitHub Actions CI check

Create the workflow file `.github/workflows/ci.yml`:

```yaml name=.github/workflows/ci.yml
name: Validate Repo-First Patches

on:
  pull_request:
    paths:
      - '**/*.py'
      - '**/*.json'
      - '.githooks/**'
      - 'CONTRIBUTING.md'

jobs:
  repo-first-patch-check:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Diff against main
        run: |
          git fetch origin main
          git diff origin/main...HEAD > copilot.patch

      - name: Ensure patch applies cleanly
        run: git apply --check copilot.patch

      - name: Verify hunk context
        run: |
          if grep -E '^@@' copilot.patch | grep -vE '^- ' >/dev/null; then
            echo "ERROR: Patch hunks missing context lines." >&2
            exit 1
          fi
```

Commit and push:

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add repo-first patch validation workflow"
```

---

## 5. Extending to `.json` (or other) files

To include JSON files (or any extension), update the file-matcher regex in the hook:

```bash
# only run on staged .py or .json files
if git diff --cached --name-only | grep -E '\.(py|json|<your-ext>)$' >/dev/null; then
  …
fi
```

And adjust the workflow’s `paths:` block accordingly:

```yaml
paths:
  - '**/*.py'
  - '**/*.json'
  - '**/*.<your-ext>'
  - '.githooks/**'
  - 'CONTRIBUTING.md'
```

---

## 6. CI runner vs. Local OS

- **Local hook** (`.githooks/validate-copilot-patch`) runs in *your* shell—Ubuntu, Fedora, Mint, macOS, etc.  
- **GitHub Actions** uses a hosted runner image when you specify:

  ```yaml
  runs-on: ubuntu-latest
  ```

  This is entirely separate from your local OS.  
  - To use your own machines, register them as **self-hosted** runners and then:

    ```yaml
    runs-on:
      - self-hosted
      - <your-label>
    ```

---

With these steps in place, **no patch can slip through** without being generated against the live `HEAD` and including proper context lines. Happy coding!  
