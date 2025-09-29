# Git Actions CI/CD Course Project – YAML
---

# Lesson 3 — Workflow Syntax & Structure (step-by-step)

## 1) Quick conceptual overview

1. A **workflow** is defined by a YAML file in `.github/workflows/`. It contains `name`, `on` (triggers), and `jobs`. Each job contains `runs-on` and `steps`. ([GitHub Docs][1])

## 2) YAML basics you must get right

1. Use **spaces** (no tabs). Indentation matters.
2. YAML scalars: plain, single-quoted, double-quoted — use double quotes if you need interpolation or special characters.
3. Lists are `- item` and mappings are `key: value`.
4. Keep comments with `#` for documentation inside workflows.

## 3) Minimal working workflow (example)

Create `.github/workflows/ci.yml`:

```yaml
name: CI - Build & Test

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: "18.x"
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test
```

## 4) Line-by-line explanation (what each top-level field does)

* `name:` — human-friendly workflow name.
* `on:` — events that trigger the workflow (push, pull_request, workflow_dispatch, schedule, etc.). ([GitHub Docs][1])
* `jobs:` — top-level map of jobs to run (each job can run in parallel by default).
* `runs-on:` — runner OS (e.g., `ubuntu-latest`, `windows-latest`).
* `steps:` — sequence inside each job: `uses:` (action) or `run:` (shell).

---

# Module 3 — Implementing Continuous Integration

## Lesson 1 — Building and Testing Code (step-by-step)

### Goal

Set up build steps and run tests automatically on push/PR.

### 1) Repository setup (one-time)

1. Create repo on GitHub or use existing.
2. From project root, ensure `package.json` has `scripts` for `test` and (optionally) `build`:

   ```json
   "scripts": {
     "build": "npm run compile || echo 'build step'",
     "test": "npm test"
   }
   ```
3. Add `.github/workflows/ci.yml`.

### 2) Full example: build + test with caching + artifact upload

```yaml
name: CI - Build & Test

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

env:
  CI: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node (with caching)
        uses: actions/setup-node@v4
        with:
          node-version: "18.x"
          cache: 'npm'                # built-in cache helper for node package managers

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Run tests
        run: npm test -- --reporter=mocha-junit-reporter || true

      - name: Upload test report
        uses: actions/upload-artifact@v4
        with:
          name: junit-results
          path: test-results/**/*.xml
```

**Notes:**

* `actions/setup-node` supports an optional `cache` input to cache npm/yarn/pnpm dependencies — this reduces repeated installs. Use it when possible. ([GitHub][2])
* `npm ci` is the recommended installer in CI (reproducible installs).
* Use `actions/upload-artifact` to save build/test artifacts for later jobs or for debugging.

### 3) Explanation of key CI steps

1. **Checkout**: pulls repo (actions/checkout).
2. **Setup Node**: sets node on PATH & optionally caches package manager cache. ([GitHub][2])
3. **Install**: `npm ci` uses lockfile for deterministic installs.
4. **Build**: run build script so tests run against built output, if relevant.
5. **Test**: run unit tests; exit code non-zero marks job failed.
6. **Artifacts**: test reports or coverage exported for analysis.

---

# Additional YAML Concepts in GitHub Actions

## 1) Using environment variables

* Define `env:` at workflow, job, or step level.

```yaml
env:
  NODE_ENV: test
jobs:
  build:
    env:
      API_BASE: ${{ secrets.API_BASE }}
    steps:
      - run: echo "Running in $NODE_ENV, API=$API_BASE"
```

* Use `${{ env.VAR_NAME }}` or `${{ secrets.SECRET_NAME }}` in expressions. (See contexts reference.) ([GitHub Docs][3])

## 2) Working with secrets (safe handling)

1. Store secrets in repo Settings → Secrets & variables → Actions (or organization / environment scopes).
2. Access via `secrets.NAME`:

   ```yaml
   - name: Deploy
     env:
       AWS_TOKEN: ${{ secrets.AWS_TOKEN }}
     run: aws deploy ...
   ```
3. **Caution:** secrets are **masked** in logs and **not** passed to workflows triggered from forks by default (security reason). For PRs from forks, repository secrets are unavailable unless maintainer approval flows are used. ([GitHub Docs][4])

## 3) Conditional execution (`if`)

* Use `if:` with expressions to control steps and jobs.

```yaml
- name: Run only on default branch
  if: github.ref == 'refs/heads/main'
  run: echo "This runs only on main"
```

* Expressions and context functions (`success()`, `failure()`, comparisons) are supported. ([GitHub Docs][5])

## 4) Using outputs & passing data between steps and jobs

### Step outputs (within a job)

```yaml
steps:
  - id: get-version
    run: |
      echo "version=1.2.3" >> "$GITHUB_OUTPUT"
  - run: echo "version is ${{ steps.get-version.outputs.version }}"
```

### Job outputs (pass from one job to another)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get-version.outputs.version }}
    steps:
      - id: get-version
        run: echo "version=1.2.3" >> "$GITHUB_OUTPUT"

  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

* Modern recommended way to set step outputs is to write to the `$GITHUB_OUTPUT` file (older `::set-output` is deprecated). Job outputs are then referenced via `needs.<job_id>.outputs.<name>`. ([GitHub Docs][6])

---

# Lesson 2 — Configuring Build Matrices (step-by-step)

## 1) Why use a matrix?

Matrix allows running the same job across multiple combinations (Node versions, OSes, browsers) automatically — fewer duplicated YAML blocks. ([GitHub Docs][7])

## 2) Basic matrix example (Node x OS)

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: ["16.x", "18.x"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      - run: npm ci
      - run: npm test
```

This will run 3 (OS) × 2 (node) = 6 jobs.

## 3) Controlling matrix behavior

1. `fail-fast: false` — continue other matrix jobs when one fails (default `true`). Useful for getting full matrix results. ([Codefresh][8])
2. `max-parallel:` — limit concurrent matrix jobs to conserve runner capacity. Example: `strategy.max-parallel: 2`. ([GitHub Docs][7])
3. `include:` and `exclude:` — add or remove specific combinations.

```yaml
strategy:
  matrix:
    node: ["14.x","16.x","18.x"]
  exclude:
    - node: "14.x"   # skip 14.x
```

## 4) Example: matrix with excludes and include metadata

```yaml
strategy:
  fail-fast: false
  max-parallel: 3
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: ["16.x", "18.x"]
    include:
      - os: ubuntu-latest
        node: "20.x"
        extra: "special-case"
    exclude:
      - os: windows-latest
        node: "16.x"
```

---

# Additional tips, best practices & troubleshooting

## Best practices

1. Keep workflows small & single-purpose: `ci.yml` for test/build, `release.yml` for deploy.
2. Use `actions/setup-node` `cache` input to simplify caching. ([GitHub][2])
3. Add `env: CI: true` if tools detect CI environment.
4. Use `npm ci` in CI for consistent installs.
5. Avoid printing secrets in logs. Use secrets only where necessary. ([GitHub Docs][4])

## Common pitfalls & fixes

* **Indentation/YAML errors:** workflow fails to parse — check spaces, no tabs.
* **Cache not hitting:** ensure cache `key` includes lockfile hash (`hashFiles('**/package-lock.json')`) or use setup-node `cache`. ([GitHub Docs][9])
* **Secrets missing in fork PRs:** this is expected; require maintainer approval or use alternate approaches (e.g., test with dummy tokens, use GitHub Environments with protected secrets). ([Stack Overflow][10])
* **Passing outputs failing:** ensure you use `echo "name=value" >> "$GITHUB_OUTPUT"` and map job outputs in `jobs.<id>.outputs`. ([GitHub Docs][6])

---

# Quick reference: common YAML snippets

**Checkout**

```yaml
- uses: actions/checkout@v4
```

**Setup Node + cache**

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: "18.x"
    cache: 'npm'
```

**Cache (custom)**

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ matrix.node }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-${{ matrix.node }}-
```

**Upload artifact**

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: coverage
    path: coverage/*
```

**Set step output**

```yaml
- id: get-sha
  run: echo "sha=${GITHUB_SHA}" >> "$GITHUB_OUTPUT"
```

**Reference job output**

```yaml
jobs:
  next:
    needs: build
    steps:
      - run: echo "build sha: ${{ needs.build.outputs.sha }}"
```

---

# Suggested classroom exercises (quick)

1. Create a repo and add the minimal `ci.yml` above; push and observe workflow run.
2. Add `cache: 'npm'` to setup-node and measure time improvement.
3. Convert workflow to a matrix (two Node versions) and inspect parallel runs.
4. Add a job that sets a version output and a follow-up job that reads it and prints it.

---

# Sources (key docs I used)

* Workflow syntax & top-level keys (official docs). ([GitHub Docs][1])
* Contexts & expressions (env, secrets, matrix context). ([GitHub Docs][3])
* `actions/setup-node` (cache support for npm/yarn/pnpm). ([GitHub][2])
* Dependency caching patterns & cache key guidance. ([GitHub Docs][9])
* Matrix strategy (how matrix runs combinations, `max-parallel`, `fail-fast`). ([GitHub Docs][7])
