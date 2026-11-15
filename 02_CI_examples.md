# a production-quality CI pipeline for a Node.js application from scratch.
# .github/workflows/ci.yml

``` yaml
name: CI Pipeline
# ── TRIGGERS ──────────────────────────────────────────────────────────────────
on:
  push:
    branches: [main]          # Run on every push to main
  pull_request:
    branches: [main]          # Run on every PR targeting main
  workflow_dispatch:          # Allow manual trigger from GitHub UI

# ── GLOBAL ENV VARS ───────────────────────────────────────────────────────────
env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}   # e.g. myorg/myapp

# ── JOBS ──────────────────────────────────────────────────────────────────────
jobs:

  # ── JOB 1: Test ─────────────────────────────────────────────────────────────
  test:
    name: Build & Test
    runs-on: ubuntu-latest

    steps:
      # Step 1: Check out the repo code onto the runner
      - name: Checkout code
        uses: actions/checkout@v4

      # Step 2: Set up Node.js with the version defined above
      # Also sets up npm caching automatically — huge speed win
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'          # Caches node_modules based on package-lock.json hash

      # Step 3: Install dependencies
      # Using ci instead of install: faster, respects lock file exactly
      - name: Install dependencies
        run: npm ci

      # Step 4: Run linter
      - name: Lint
        run: npm run lint

      # Step 5: Run tests with coverage
      - name: Run tests
        run: npm test -- --coverage

      # Step 6: Upload coverage report as an artifact
      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 7     # Auto-delete after 7 days

  # ── JOB 2: Security scan (parallel to test) ──────────────────────────────────
  security:
    name: Security Scan
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Audit npm packages for known vulnerabilities
      - name: Audit dependencies
        run: npm audit --audit-level=high
        # --audit-level=high means: fail only on HIGH or CRITICAL severity

  # ── JOB 3: Build Docker image (only after test + security pass) ─────────────
  build:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: [test, security]   # Only runs if BOTH jobs above succeed

    # Only push image on main branch — not on PRs (no credentials on PRs from forks)
    if: github.ref == 'refs/heads/main'

    # Permissions needed to push to GitHub Container Registry
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Log in to GitHub Container Registry using auto-generated token
      - name: Log in to container registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}   # Auto-available, no setup needed

      # Extract metadata — automatically generates tags and labels
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-          # Tag with commit SHA: sha-a3f7c21
            type=ref,event=branch         # Tag with branch name: main
            type=semver,pattern={{version}} # Tag with semver if you push a tag: v1.2.3

      # Build and push — uses BuildKit for layer caching
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha           # Use GitHub Actions cache for Docker layers
          cache-to: type=gha,mode=max    # Save layers to cache after build
```

## Now the matrix build — test across multiple versions
This is a powerful pattern for libraries or apps that must support multiple environments:
``` yaml
jobs:
  test:
    name: Test (Node ${{ matrix.node-version }}, OS ${{ matrix.os }})
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        node-version: [18, 20, 22]
        os: [ubuntu-latest, windows-latest]
      fail-fast: false    # Don't cancel other matrix jobs if one fails

    steps:
      - uses: actions/checkout@v4

      - name: Set up Node ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - run: npm ci
      - run: npm test
```
This single test job definition spawns 6 parallel jobs automatically (3 Node versions × 2 OSes). Each appears separately in the GitHub UI with its own logs.

## Secrets — the right way
Never hardcode credentials. GitHub Actions has a layered secrets system:
``` yaml
# Accessing secrets in your workflow:
steps:
  - name: Deploy to AWS
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    run: |
      aws s3 sync ./dist s3://my-bucket

  # Or pass directly to an action:
  - name: Deploy
    uses: some/deploy-action@v1
    with:
      api-key: ${{ secrets.DEPLOY_API_KEY }}
```
Scope	Set in	Available to
```
Repository secret	Repo → Settings → Secrets	That repo only
Environment secret	Repo → Settings → Environments	Jobs targeting that environment
Organization secret	Org → Settings → Secrets	Selected repos in the org
```
Environment secrets are the production-safety feature: you define an environment called production, add secrets to it, and optionally add a protection rule requiring reviewer approval before any job targeting that environment runs.
```
jobs:
  deploy-prod:
    environment: production    # Triggers environment protection rules
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          PROD_DB_URL: ${{ secrets.PROD_DB_URL }}   # Only available after approval
        run: ./deploy.sh
```
## Reusable workflows — DRY at the pipeline level
When you have 5 microservices that all need the same CI pipeline, you don’t copy-paste. You create a reusable workflow:

``` yaml
# .github/workflows/_reusable-ci.yml
# The underscore prefix is convention for "internal, not triggered directly"

name: Reusable CI

on:
  workflow_call:            # This makes it callable from other workflows
    inputs:
      node-version:
        required: false
        type: string
        default: '20'
      working-directory:
        required: false
        type: string
        default: '.'
    secrets:
      NPM_TOKEN:
        required: false     # Optional secret passed from caller

jobs:
  ci:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
          cache-dependency-path: '${{ inputs.working-directory }}/package-lock.json'

      - run: npm ci
      - run: npm test
```
``` yaml
# .github/workflows/service-a-ci.yml  — in any microservice repo
name: Service A CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: myorg/shared-workflows/.github/workflows/_reusable-ci.yml@main
    with:
      node-version: '20'
      working-directory: './service-a'
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```
## Composite actions — DRY at the step level
Reusable workflows share whole jobs. Composite actions share individual steps:
``` yaml
# .github/actions/setup-app/action.yml
name: Setup Application
description: Checkout, setup Node, install deps — used across many workflows

inputs:
  node-version:
    description: Node.js version
    default: '20'

runs:
  using: composite
  steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - run: npm ci
      shell: bash
```
```
# Any workflow can now use this in one line:
steps:
  - uses: ./.github/actions/setup-app
    with:
      node-version: '20'

  - run: npm test   # Node + deps already set up
```
Conditional execution — power patterns
``` yaml
steps:
  # Only run on push to main, not on PRs
  - name: Deploy
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: ./deploy.sh

  # Always run — even if previous steps failed (useful for cleanup)
  - name: Upload test results
    if: always()
    uses: actions/upload-artifact@v4
    with:
      name: test-results
      path: test-results/

  # Run only if a previous step failed
  - name: Notify on failure
    if: failure()
    run: |
      curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
        -H 'Content-type: application/json' \
        -d '{"text": "Pipeline failed on ${{ github.ref }}"}'

  # Skip based on commit message — useful for [skip ci] convention
  - name: Deploy
    if: "!contains(github.event.head_commit.message, '[skip ci]')"
    run: ./deploy.sh
```
## Outputs — passing data between jobs
Jobs run on separate VMs, so to pass data you use outputs:
``` yaml
jobs:
  get-version:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}   # Expose step output as job output

    steps:
      - uses: actions/checkout@v4

      - name: Get version from package.json
        id: version
        run: |
          VERSION=$(node -p "require('./package.json').version")
          echo "version=$VERSION" >> $GITHUB_OUTPUT   # Write to special file

  deploy:
    needs: get-version
    runs-on: ubuntu-latest
    steps:
      - name: Use version
        run: |
          echo "Deploying version ${{ needs.get-version.outputs.version }}"
          docker tag myapp:latest myapp:${{ needs.get-version.outputs.version }}
```


