# GitHub Actions — Complete Deep Dive

GitHub Actions is the CI/CD platform / It’s the most widely used, lives inside your repo, and the concepts transfer directly to every other CI system (GitLab CI, CircleCI, Jenkins).

- GitHub Actions is an automation platform integrated with GitHub repositories.
It can react to events in your repository and execute predefined workflows.

- The execution model — how GitHub Actions actually works
<img width="1266" height="1051" alt="image" src="https://github.com/user-attachments/assets/e5a5d954-7fc3-409f-84c6-d51ead740d5b" />


# Important concepts
```
Workflow
   ↓
Trigger
   ↓
Job
   ↓
Runner
   ↓
Steps
```
- `workflow` - A workflow describes an automated process. ex: CI workflow, deployment workflow. .github/workflows/ is where github look for workflows.
- `Trigger` - used to run the workflow. ex: push, pull request, workflow_dispatch, schedule.
- `Job` - a workflow contains one or more jobs. ex:
```
Workflow
│
├── test job
│
├── build job
│
└── deploy job
```
- `runner` - runner is machine/environmnet that executes our workflows. GitHub provides hosted runners, and organizations can also use self-hosted runners.
```
Workflow
   ↓
Job
   ↓
Runner
   ↓
Commands
```
- `step` - a job contains a step.
ex:
Job: test
```
Step 1 → checkout repository
Step 2 → install Python
Step 3 → install dependencies
Step 4 → run pytest
```
so the hierarchy is: 
```
Workflow
    │
    └── Job
          │
          ├── Step
          ├── Step
          ├── Step
          └── Step
```

- A CI EXAMPLE
``` yml
name: CI

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run tests
        run: pytest
```
- For a Python application, we'd typically do something like:
``` yml
steps:
  - name: Checkout
    uses: actions/checkout@v4

  - name: Setup Python
    uses: actions/setup-python@v5
    with:
      python-version: "3.12"

  - name: Install dependencies
    run: pip install -r requirements.txt

  - name: Run tests
    run: pytest
```

if one job depends on other, we used need ex: we have build, test job. if we want to test first and then build then in build job we will use something like.
```
jobs:
  test:
    ...

  build:
    needs: test
    ...
```

- Parrallelism - without the use of needs, jobs can potentially run simultaneously. which can help in design of efficient pipeline.

## CI PIPELINE FOR FASTAPI PROJECT.
- project
```
React
FastAPI
PostgreSQL
Docker
AWS
```
- A CI Workflow could look like
```
Push / Pull Request
        ↓
     GitHub
        ↓
   ┌──────────────┐
   │ Test Backend │
   └──────┬───────┘
          │
   ┌──────┴───────┐
   ↓              ↓
Lint           Frontend Test
   │              │
   └──────┬───────┘
          ↓
      Docker Build
          ↓
        PASS
```
- THEN CD.
```
Docker image
     ↓
Container registry
     ↓
AWS
     ↓
Production
```

## Variables vs Secrets.
A variable might be like. ENVIRONMENT=PRODUCTION
SECRET IS SENSITIVE INFORMATION.
we can store secrests in github secretes and use them like. ${{ secrets.AWS_ACCESS_KEY_ID }}
Sensitive configuration should be injected at runtime rather than committed into source control.

## ENVIRONMENTS
```
CI
 ↓
Build
 ↓
Deploy staging
 ↓
Tests
 ↓
Approval
 ↓
Deploy production
``` 
