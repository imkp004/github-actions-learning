# GitHub Actions Architecture: Repository, Workflows, Jobs, and Steps

GitHub Actions is organized into a hierarchy.

The easiest way to understand it is:

```text
GitHub Repository
        │
        ▼
    Workflow
        │
        ▼
      Jobs
        │
        ▼
     Steps
        │
        ├── Run shell commands
        │
        └── Use Actions
```

Each level has a different responsibility.

A more detailed mental model is:

```text
Repository
    │
    ├── Workflow 1
    │       │
    │       ├── Job 1
    │       │      ├── Step 1
    │       │      ├── Step 2
    │       │      └── Step 3
    │       │
    │       └── Job 2
    │              ├── Step 1
    │              └── Step 2
    │
    └── Workflow 2
            │
            └── Job 1
                   ├── Step 1
                   └── Step 2
```

---

# 1. Repository

Everything starts with a **GitHub repository**.

Your repository contains your application code, configuration files, documentation, and potentially your GitHub Actions workflows.

For example:

```text
my-application/
│
├── app.py
├── requirements.txt
├── README.md
├── tests/
│   └── test_app.py
│
└── .github/
    └── workflows/
        ├── ci.yml
        └── deploy.yml
```

GitHub Actions workflows are usually stored inside:

```text
.github/workflows/
```

For example:

```text
.github/workflows/ci.yml
```

and:

```text
.github/workflows/deploy.yml
```

A single repository can have **multiple workflows**.

For example:

```text
Repository
│
├── CI Workflow
│   ├── Run tests
│   └── Build application
│
├── Deployment Workflow
│   └── Deploy application
│
├── Security Workflow
│   └── Scan dependencies
│
└── Terraform Workflow
    ├── terraform fmt
    ├── terraform validate
    └── terraform plan
```

The repository is the container where your workflows live.

---

# 2. Workflow

A **workflow** is an automated process defined in a YAML file.

A workflow is attached to a specific GitHub repository and contains one or more jobs.

A workflow usually answers these questions:

1. **When should this automation run?**
2. **What jobs should run?**
3. **What should those jobs do?**

The basic structure looks like this:

```yaml
name: CI Pipeline

on:
  push:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest
```

Let's break this down:

```text
Workflow: CI Pipeline
        │
        ├── Trigger:
        │   Push to main
        │
        └── Jobs:
            └── Test
```

The workflow is the **overall automation process**.

---

## Workflow Triggered by an Event

A workflow does not normally run randomly.

It waits for an **event**.

An event is something that happens inside or around the GitHub repository.

For example:

```text
Developer pushes code
        ↓
GitHub detects a push event
        ↓
Workflow is triggered
        ↓
Jobs begin running
```

Example:

```yaml
on:
  push:
    branches:
      - main
```

This means:

> Run this workflow when code is pushed to the `main` branch.

For example:

```bash
git add .
git commit -m "Add login feature"
git push origin main
```

The `git push` creates a GitHub event.

GitHub then checks the repository's workflows.

```text
git push
    ↓
Push Event
    ↓
GitHub checks workflow configuration
    ↓
Does the event match the trigger?
    ↓
Yes
    ↓
Start workflow
```

---

## Multiple Workflow Triggers

A workflow can also respond to multiple events.

Example:

```yaml
on:
  push:
    branches:
      - main

  pull_request:
    branches:
      - main
```

This workflow runs when:

1. Code is pushed to `main`
2. A pull request targets `main`

Conceptually:

```text
                 ┌── Push to main ──┐
                 │                  │
Developer Action │                  ▼
                 │             CI Workflow
                 │                  ▲
                 └── Pull Request ──┘
```

You can also manually trigger a workflow:

```yaml
on:
  workflow_dispatch:
```

This creates the ability to manually start the workflow from GitHub.

```text
User
  ↓
Click "Run workflow"
  ↓
GitHub Actions
  ↓
Workflow starts
```

This is useful for tasks such as:

* Manual deployments
* Running Terraform
* Database maintenance
* Creating backups

---

# 3. Jobs

A **job** is a group of related steps that perform a specific task.

A workflow contains one or more jobs.

For example:

```yaml
name: CI Pipeline

on:
  push:

jobs:
  test:
    runs-on: ubuntu-latest

  build:
    runs-on: ubuntu-latest
```

This workflow contains two jobs:

```text
CI Pipeline
│
├── Job: test
│
└── Job: build
```

Each job defines its own **execution environment**, called a **runner**.

---

# What Is a Runner?

A runner is the machine or execution environment where a job runs.

For example:

```yaml
runs-on: ubuntu-latest
```

This means the job will run on an Ubuntu environment.

Conceptually:

```text
GitHub Actions
      │
      ▼
┌─────────────────────┐
│   Ubuntu Runner     │
│                     │
│  Executes the Job   │
│                     │
└─────────────────────┘
```

A job can use different operating systems.

For example:

```yaml
runs-on: ubuntu-latest
```

```yaml
runs-on: windows-latest
```

```yaml
runs-on: macos-latest
```

You can also use a **self-hosted runner**, where the job runs on infrastructure you manage.

---

# Important: Each Job Has Its Own Execution Environment

Consider this workflow:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

  build:
    runs-on: ubuntu-latest
```

Conceptually, GitHub can provide separate runners:

```text
                 Workflow
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
       Test Job              Build Job
          │                     │
          ▼                     ▼
    Ubuntu Runner 1      Ubuntu Runner 2
```

This is an important concept.

A workflow is the overall process, while a job is a unit of work that runs in its own execution environment.

Because jobs can use separate runners, you should not assume that one job's local files automatically exist in another job.

For example:

```text
Job 1 Runner
├── Download code
├── Build application
└── Create file: app.zip

                X

Job 2 Runner
└── app.zip is NOT automatically available
```

If one job needs to pass files or build output to another job, you typically use mechanisms such as **artifacts**, a shared external service, or another explicit data-sharing method.

---

# Jobs Can Run in Parallel

By default, independent jobs can run in parallel.

Example:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running tests"

  security:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running security scan"
```

Conceptually:

```text
Workflow starts
      │
      ├───────────────┐
      ▼               ▼
   Test Job      Security Job
      │               │
      │               │
      ▼               ▼
    Running         Running
```

Both jobs can execute at approximately the same time.

This can make your pipeline faster.

For example, instead of this:

```text
Run tests
    ↓
Wait
    ↓
Run security scan
    ↓
Wait
    ↓
Run linting
```

You can have:

```text
              ┌── Run Tests ──────┐
              │                   │
Workflow ─────┼── Security Scan ──┼── Results
              │                   │
              └── Run Linter ─────┘
```

---

# Jobs Can Also Run Sequentially

Sometimes one job depends on another.

For example, you do not want to deploy an application before the tests pass.

You can define dependencies using `needs`.

Example:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - run: echo "Running tests"

  build:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - run: echo "Building application"

  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - run: echo "Deploying application"
```

Now the jobs execute in order:

```text
Test
 │
 │ Must succeed
 ▼
Build
 │
 │ Must succeed
 ▼
Deploy
```

Conceptually:

```text
Workflow
    │
    ▼
┌──────────┐
│   TEST   │
└──────────┘
      │
      │ Success
      ▼
┌──────────┐
│  BUILD   │
└──────────┘
      │
      │ Success
      ▼
┌──────────┐
│  DEPLOY  │
└──────────┘
```

If the test job fails:

```text
Test ❌
  │
  ▼
Build ⏭️ Skipped
  │
  ▼
Deploy ⏭️ Skipped
```

This is a common CI/CD pattern.

---

# Jobs Can Be Conditional

A job can also run only when a specific condition is true.

Example:

```yaml
deploy:
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest

  steps:
    - run: echo "Deploying to production"
```

This means:

> Only run the `deploy` job if the workflow is running from the `main` branch.

Conceptually:

```text
Push Event
    │
    ▼
Which branch?
    │
    ├── feature/login
    │       ↓
    │   Do not deploy
    │
    └── main
            ↓
       Deploy Job Runs
```

This is useful when you want different behavior for different branches.

For example:

```text
Feature Branch
      ↓
Run Tests
      ↓
Run Linter

Main Branch
      ↓
Run Tests
      ↓
Build Application
      ↓
Deploy to Production
```

---

# 4. Steps

A **step** is an individual task inside a job.

Steps are the actual instructions that the runner executes.

For example:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
      - name: Install Python
      - name: Install dependencies
      - name: Run tests
```

The job contains multiple steps:

```text
Test Job
│
├── Step 1: Checkout code
│
├── Step 2: Install Python
│
├── Step 3: Install dependencies
│
└── Step 4: Run tests
```

By default, steps inside the same job run **sequentially**.

```text
Step 1
   ↓
Step 2
   ↓
Step 3
   ↓
Step 4
```

The next step normally waits for the previous step to finish.

---

# Two Main Types of Steps

A step generally performs work in one of two ways:

1. **Run shell commands or scripts**
2. **Use an Action**

---

## Type 1: Run Shell Commands or Scripts

You can use the `run` keyword to execute commands directly on the runner.

Example:

```yaml
- name: Show current directory
  run: pwd
```

GitHub Actions executes:

```bash
pwd
```

on the runner.

Another example:

```yaml
- name: List files
  run: ls -la
```

The runner executes:

```bash
ls -la
```

You can also run multiple commands:

```yaml
- name: Install and test
  run: |
    pip install -r requirements.txt
    pytest
```

Conceptually:

```text
Step starts
    ↓
Open shell on runner
    ↓
Run: pip install -r requirements.txt
    ↓
Run: pytest
    ↓
Step completes
```

---

## Running Your Own Script

You can also execute scripts stored in your repository.

For example:

```text
my-project/
│
├── scripts/
│   └── deploy.sh
│
└── .github/
    └── workflows/
        └── deploy.yml
```

Your workflow can run:

```yaml
- name: Run deployment script
  run: ./scripts/deploy.sh
```

Conceptually:

```text
GitHub Actions Step
        │
        ▼
Execute deploy.sh
        │
        ▼
Your custom script performs deployment tasks
```

This means you can write your automation logic yourself.

---

# Type 2: Use an Action

An **Action** is reusable automation that someone has already packaged.

Instead of writing the logic yourself, you can reuse an existing Action.

For example:

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```

This step uses the `actions/checkout` Action.

Conceptually:

```text
Your Workflow Step
        │
        ▼
uses: actions/checkout@v4
        │
        ▼
Reusable Action
        │
        ▼
Repository is checked out
```

Another example:

```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: "3.12"
```

This uses an Action to set up Python.

The `with` section provides input to the Action.

```text
Step
 │
 ├── Action: actions/setup-python
 │
 └── Input:
       python-version = 3.12
```

---

# Your Own Actions vs Third-Party Actions

You can use:

### 1. Official Actions

For example:

```yaml
uses: actions/checkout@v4
```

```yaml
uses: actions/setup-python@v5
```

These are commonly used reusable Actions.

---

### 2. Third-Party Actions

Other developers and organizations can also create reusable Actions.

For example:

```text
GitHub Marketplace
       │
       ├── Docker Actions
       ├── AWS Actions
       ├── Terraform Actions
       ├── Slack Actions
       └── Security Actions
```

You can use an Action created by another organization if it meets your requirements.

Conceptually:

```text
Your Workflow
      │
      ▼
Your Step
      │
      ▼
Third-Party Action
      │
      ▼
Reusable Automation
```

However, third-party Actions should be reviewed carefully because they run as part of your CI/CD pipeline and may have access to repository data or credentials.

---

### 3. Your Own Custom Actions

You can also create your own reusable Action.

For example:

```text
my-repository/
│
├── .github/
│   └── actions/
│       └── setup-environment/
│           └── action.yml
│
└── .github/
    └── workflows/
        └── ci.yml
```

Then your workflow can use:

```yaml
- name: Set up application environment
  uses: ./.github/actions/setup-environment
```

This is useful when you have automation logic that you want to reuse across multiple workflows.

---

# Complete Example: Repository → Workflow → Jobs → Steps

Consider the following repository:

```text
my-python-app/
│
├── app.py
├── requirements.txt
├── tests/
│   └── test_app.py
│
└── .github/
    └── workflows/
        └── ci-cd.yml
```

Inside `ci-cd.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:

  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest

  deploy:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - name: Deploy application
        run: echo "Deploying application"
```

Now let's map this to the architecture.

```text
Repository
│
└── .github/workflows/ci-cd.yml
        │
        ▼
    WORKFLOW
    CI/CD Pipeline
        │
        │ Trigger:
        │ Push to main
        │
        ├──────────────────────────┐
        │                          │
        ▼                          │
    JOB: test                      │
    Runner: Ubuntu                 │
        │                          │
        ├── STEP                   │
        │   Checkout repository    │
        │                          │
        ├── STEP                   │
        │   Set up Python          │
        │                          │
        ├── STEP                   │
        │   Install dependencies   │
        │                          │
        └── STEP                   │
            Run tests              │
                                    │
             Test succeeds          │
                    │               │
                    ▼               │
              JOB: deploy ◄─────────┘
              Runner: Ubuntu
                    │
                    └── STEP
                        Deploy application
```

---

# How the Workflow Actually Executes

Let's walk through the complete process.

## Step 1: Code Is Pushed

A developer runs:

```bash
git push origin main
```

This creates a `push` event.

```text
Developer
    │
    │ Pushes code
    ▼
GitHub Repository
    │
    ▼
Push Event
```

---

## Step 2: Workflow Is Triggered

GitHub checks:

```yaml
on:
  push:
    branches:
      - main
```

The push happened on `main`.

The condition matches.

```text
Push to main
     ↓
Trigger matches
     ↓
Workflow starts
```

---

## Step 3: Test Job Starts

GitHub starts the `test` job.

```yaml
test:
  runs-on: ubuntu-latest
```

An Ubuntu runner is assigned to execute the job.

```text
Test Job
    ↓
Ubuntu Runner Created
```

---

## Step 4: Steps Run Sequentially

The runner executes the steps one at a time.

```text
1. Checkout repository
        ↓
2. Set up Python
        ↓
3. Install dependencies
        ↓
4. Run tests
```

The first step:

```yaml
uses: actions/checkout@v4
```

downloads the repository.

The second step:

```yaml
uses: actions/setup-python@v5
```

sets up Python.

The third step:

```yaml
run: pip install -r requirements.txt
```

executes a shell command.

The fourth step:

```yaml
run: pytest
```

executes the tests.

---

## Step 5: Job Completes

If all steps succeed:

```text
Test Job: SUCCESS ✅
```

Because the `deploy` job depends on the `test` job:

```yaml
needs: test
```

GitHub can now start the deployment job.

---

## Step 6: Deployment Job Runs

```text
Test Job ✅
     │
     ▼
Deploy Job Starts
     │
     ▼
Deploy Step Runs
     │
     ▼
Deployment Complete
```

If the test job fails:

```text
Test Job ❌
     │
     ▼
Deploy Job does not run
```

---

# Jobs vs Steps: Important Difference

This is one of the most important distinctions.

## A Job

A job is a **larger unit of work**.

It has its own:

* Runner
* Execution environment
* Steps
* Dependencies
* Conditions

Example:

```yaml
test:
  runs-on: ubuntu-latest
```

---

## A Step

A step is an **individual task inside a job**.

Example:

```yaml
- name: Run tests
  run: pytest
```

---

Think about a restaurant.

```text
Restaurant = Workflow

Kitchen = Job

Individual cooking task = Step
```

Or in GitHub Actions:

```text
Workflow = Complete automation process

Job = Major phase of the process

Step = Individual instruction
```

For example:

```text
CI/CD Workflow
│
├── Test Job
│   ├── Checkout code
│   ├── Install dependencies
│   └── Run tests
│
├── Build Job
│   ├── Build application
│   └── Create Docker image
│
└── Deploy Job
    ├── Push image to ECR
    └── Deploy to ECS
```

---

# Complete Mental Model

The complete hierarchy is:

```text
┌──────────────────────────────────────┐
│           REPOSITORY                 │
│                                      │
│   Contains application code and      │
│   GitHub Actions workflow files      │
│                                      │
└───────────────────┬──────────────────┘
                    │
                    ▼
┌──────────────────────────────────────┐
│             WORKFLOW                 │
│                                      │
│  Automated process triggered by      │
│  an event                            │
│                                      │
│  Contains one or more jobs           │
│                                      │
└───────────────────┬──────────────────┘
                    │
                    ▼
┌──────────────────────────────────────┐
│               JOB                    │
│                                      │
│  Defines the execution environment   │
│  using a runner                      │
│                                      │
│  Contains one or more steps          │
│                                      │
└───────────────────┬──────────────────┘
                    │
                    ▼
┌──────────────────────────────────────┐
│              STEP                    │
│                                      │
│  Performs an individual task         │
│                                      │
│  Can:                                │
│  • Run a shell command               │
│  • Run a script                      │
│  • Use an Action                     │
│                                      │
└──────────────────────────────────────┘
```

---

# Key Summary

```text
Repository
    ↓
Contains one or more Workflows
    ↓
Workflow
    ↓
Triggered by an Event
    ↓
Contains one or more Jobs
    ↓
Job
    ↓
Runs on a Runner
    ↓
Contains one or more Steps
    ↓
Step
    ↓
Runs a command, script, or Action
```

The most important thing to remember is:

> **A workflow is the complete automation pipeline, a job is a major unit of work running on a runner, and a step is an individual task executed inside that job.**

A simple real-world CI/CD example is:

```text
Push Code
    ↓
Workflow Triggered
    ↓
┌─────────────────────────┐
│       Test Job          │
│    Ubuntu Runner        │
│                         │
│  1. Checkout Code       │
│  2. Install Packages    │
│  3. Run Tests           │
└────────────┬────────────┘
             │
             │ Success
             ▼
┌─────────────────────────┐
│       Build Job         │
│    Ubuntu Runner        │
│                         │
│  1. Build Application   │
│  2. Build Docker Image  │
└────────────┬────────────┘
             │
             │ Success
             ▼
┌─────────────────────────┐
│       Deploy Job        │
│    Ubuntu Runner        │
│                         │
│  1. Push to Registry    │
│  2. Deploy Application  │
└─────────────────────────┘
```

This hierarchy—**Repository → Workflow → Job → Step**—is the foundation for understanding GitHub Actions.
