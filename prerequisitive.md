
#  CI/CD - Continuous integration & Continuous deployment.

- ` Problem before Ci/Cd` 
imagine building a web application. 
Developer -> COde -> Github .
But our application needs more than source code to actually work.
we might need to: 
- install dependencies 
- Run Test 
- Build the frontend
- Build docker images
- run secuirty checks
- deploy to a server 
- restart services
- run database migrations 
- verify that deployment worked.

` So process looks like this in general : `
````
                Code
                 ↓
                Install dependencies
                 ↓
                Test
                 ↓
                Build
                 ↓
                Package
                 ↓
                Deploy
                 ↓
                Verify
````
If Humans have to do all of this manually every time someone changes the code, several problems appear.

- `Problem 1` - Human made mistakes. someone build wrong configuration. deployes wrong branch or forgets a migration.
-  `Problem 2` - It is slow - every dev has to repeatedly perfomr the same tasks.
- `Problem 3 `- inconsistency - different dev deploy in differenet ways. production behaves differently from their laptop.
- `Problem 4 `- Feedback comes too late 


# Fundamental Idea behind CI/CD. 
- whenever code changes , automatically perform the steps required to verify and deliver that code instead of manually.

- `what does CI means` 
suppose 5 developers working on same application, independentely witting code. eventually their changes have to be integrated frequently. 
- Historically , team might integrate their chnages infrequently. 
- ex: week 1: every each of them write code and in week 2 they tried to marge everything. this becomes nightmare. we face problems like,
merge conflicts + broken tests + incompatible changes + Unknown bugs.

- `CI - Continous Integration says`: integrate changes into the shared codebase frequently and automatically verify them.
```
Developer
   │
 git push
   ↓
GitHub
   ↓
CI system
   │
   ├── Install dependencies
   ├── Run tests
   ├── Lint
   ├── Security checks
   └── Build
   │
PASS / FAIL
```
# Why CI is powerful ? 

- `Without CI`
Monday      → write code
Tuesday     → write more code
Wednesday   → write more code
Thursday    → merge everything
Friday      → discover application is broken {we now have thousands of lines of possible causes.}

- `With CI`
```
commit 1 → test → ✅ 
commit 2 → test → ✅
commit 3 → test → ❌ {we now know problem is probably in commit 3}
```
- this dramatically reduces the feedback loop.

# What is CD ???
- CD refers to :  `Continous Delivery or Continous Deployment` 

# Continuous Delivery
- Every change that passes the pipeline is automatically prepared and made ready for production. 
- But a human may still decide when to actually release it.
-
````
                        Developer
                            ↓
                        Push
                            ↓
                        CI
                            ↓
                        Tests
                            ↓
                        Build
                            ↓
                        Package
                            ↓
                        Production-ready artifact
                            ↓
                        👤 Human approval
                            ↓
                        Production
````
# Continuous Deployment
- Continuous Deployment = automatically release every validated change to production.
```
              CI
   ┌─────────────────────┐
   │ Install             │
   │ Test                │
   │ Lint                │
   │ Security            │
   │ Build               │
   └──────────┬──────────┘
              │
           SUCCESS
              │
              ↓
             CD
   ┌─────────────────────┐
   │ Package             │
   │ Release             │
   │ Deploy              │
   │ Verify              │
   └─────────────────────┘
````

# PipeLine : 
- a CI/CD pipeline is simply: 
- An automated sequence of steps that takes a change in source code and moves it towards a deployable application.
- example pipeline:
```
Git Push
   ↓
Checkout code
   ↓
Install dependencies
   ↓
Run tests
   ↓
Build application
   ↓
Build Docker image
   ↓
Push image to registry
   ↓
Deploy
   ↓
Health check   
```
# Example Project.
- React -> FastAPI -> PostgreSQL . Dockerized and deployed to AWS. 
imagine we change: 
``` python
@app.get("/tasks") 
def get_tasks(): 
    ..... 
```
we push: 
``` bash
- git add . 
- git commit -m "fix task endpoint"
- git push 
```
Now CI/CD can take over.. 

- `STAGE 1:` Checkout. CI server gets our repository . 
- `STAGE 2:` Install dependencies. ex. pip install -r requirments.txt or npm install 
- `STAGE 3:` Test- ex. pytest or npm test
- `STAGE 4:` Build - maybe our app is packaged into a docker image. so docker build -t my-api .  now we have docker image (artifact). 

- `Artifact` - the output produced by a build process that can be stored, transported, and deployed eg. docker image, Dist/ (frontend)

# Container Registry.. 
- A storage/distribution system for container images.
- if we are using docker, the image goes to a container registry.
- example:
```
GitHub
   ↓
CI
   ↓
docker build
   ↓
Docker image
   ↓
Container Registry
```
- examples: Github container registery, Amazon ECR, Docker hub. Google Artifact Registry, Azure Container Registry.

# Deployment..
- now as our image exits. our deployment system can retrieve it and run it.
ex: ECR -> AWS server -> docker pull my-api:1.42 -> docker run ...  
or using k8s. Container registry -> K8s -> Pod -> Application 

- so the complete Flow becomes:
```
Developer
    ↓
git push
    ↓
GitHub
    ↓
CI
    │
    ├── Test
    ├── Lint
    ├── Build
    └── Security scan
    ↓
Docker image
    ↓
Container Registry
    ↓
CD
    ↓
AWS / Kubernetes
    ↓
Production
```
# Platforms for implementing CI/CD Automation.
GitHub Actions
GitLab CI/CD
Jenkins
CircleCI
Azure DevOps Pipelines
Argo CD
Tekton

# What is a Runner ?
suppose github recieves `git push` something needs to actually execute : pytest, docker build, npm install etc. that something is a runner.
- `A runner` is basically a machine that executes your CI/CD jobs.
- it could be: Github-hosted machines or our own self-hosted machine.

# Workflow → Jobs → Steps
```
Workflow
   │
   ├── Job
   │    ├── Step
   │    ├── Step
   │    └── Step
   │
   └── Job
        ├── Step
        └── Step
```
- WorkFlow = The entire automation definition. ex- CI Pipeline
- Job = A group of tasks executed together on a runner. ex- test
- step = An individual operation. ex- install dependencies, Run pytest.

- Trigger - trigger define when doe pipeline runs.
- ex triggers: push, pull_request, manual, schedule.

- Secrets = eg: db keys, aws , api keys etc. we store that in github secrets and then using runner we use them in pipeline.

- Docker = How do I package my application consistently?
- CI/CD = How do I package my application consistently?

# Deeper reason: CI/CD Exists....
software engineering has a feedback loop: 
- `Change => Build => Test => Deploy => Observe => FeedBack => Change again`
- Good engineering tries to make this loop fast, automated, reliable and repeatable. 

# CI/CD doesn't mean "deploy everything automatically"
- we can have CI only.
- CI + Continuous Delivery 
- CI + Continuous Deployment


# A realistic production pipeline
```
                    Developer
                       │
                       │ git push
                       ↓
                    GitHub
                       │
                       ↓
              ┌─────────────────┐
              │   CI Pipeline   │
              ├─────────────────┤
              │ Checkout        │
              │ Dependencies    │
              │ Lint            │
              │ Unit tests      │
              │ Security scan   │
              │ Build           │
              └────────┬────────┘
                       │
                     PASS
                       ↓
                Docker Build
                       │
                       ↓
                 Docker Image
                       │
                       ↓
              Container Registry
                       │
                       ↓
                CD Pipeline
                       │
                       ↓
                  Staging
                       │
                       ↓
               Integration Test
                       │
                       ↓
                  Production
                       │
                       ↓
                Health Checks
                       │
                       ↓
                  Monitoring

```
and then Prometheus / Grafana / logs => Observe => Feedback

# Summary with interview style .
- `What is CI ?`
Continuous Integration is the practice of frequently integrating code changes into a shared repository and automatically validating those changes through activities such as building, testing, linting, and security checks.

- `What is CD?`
Continuous Delivery automates the process of getting validated software into a production-ready state, while Continuous Deployment goes further and automatically releases validated changes to production.

- `What is CI/CD pipeline?`
A CI/CD pipeline is an automated sequence of stages that takes source-code changes through validation, building, packaging, and ultimately delivery or deployment.

- `What is a runner?`
A runner is the machine or execution environment that actually executes the CI/CD jobs and commands.

- `What is an artifact?`
An artifact is the output of a build process that can be stored and subsequently deployed, such as a Docker image or compiled application.


<img width="1290" height="591" alt="image" src="https://github.com/user-attachments/assets/d52edd75-bc89-4d4a-b9bc-e737681c056d" />

<img width="1266" height="890" alt="image" src="https://github.com/user-attachments/assets/93a8a1f4-e0cb-42cd-8b62-79f73484ff29" />

