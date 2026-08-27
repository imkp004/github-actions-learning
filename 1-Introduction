# GitHub Actions

## What Is GitHub Actions?

**GitHub Actions** is a CI/CD and automation platform built directly into GitHub.

It allows you to create automated workflows that perform specific tasks when something happens in your GitHub repository.

For example, you can configure GitHub Actions to automatically:

* Run tests when you push code
* Build your application
* Check code quality
* Create Docker images
* Deploy an application to AWS
* Run Terraform commands
* Send notifications
* Publish packages

Instead of manually performing these tasks every time you make a change, GitHub Actions can automate them for you.

### Simple Example

Imagine you have a Python application.

Without automation, your workflow might look like this:

```text
1. Write code
        ↓
2. Run tests manually
        ↓
3. Build the application
        ↓
4. Deploy manually
```

With GitHub Actions:

```text
Developer pushes code to GitHub
                ↓
        GitHub Actions starts
                ↓
         Run automated tests
                ↓
        Build the application
                ↓
          Deploy to AWS
```

This is where **CI/CD** comes in.

---

# What Is CI/CD?

CI/CD stands for:

* **CI = Continuous Integration**
* **CD = Continuous Delivery or Continuous Deployment**

CI/CD is a software development practice that automates the process of integrating, testing, building, and deploying applications.

---

# Continuous Integration (CI)

**Continuous Integration** means developers frequently merge or push their code changes into a shared repository.

Every time new code is added, automated processes verify that the new changes do not break the application.

For example:

```text
Developer writes new code
            ↓
     git push origin main
            ↓
     GitHub Actions starts
            ↓
        Install dependencies
            ↓
          Run tests
            ↓
      Check code quality
            ↓
      Build the application
```

The main goal of CI is to **catch problems early**.

### Example

Suppose your application has a login function:

```python
def login(username, password):
    if username == "admin" and password == "password":
        return True

    return False
```

You have automated tests to verify that the login function works correctly.

When a developer changes the code and pushes it to GitHub:

```text
git push origin main
```

GitHub Actions can automatically:

1. Start a virtual machine
2. Download your repository
3. Install Python
4. Install project dependencies
5. Run your tests
6. Report whether the tests passed or failed

If the tests fail, GitHub will show that the workflow failed.

This helps prevent broken code from being deployed.

---

# Continuous Delivery and Continuous Deployment (CD)

The second part of CI/CD is **CD**.

CD can mean either:

* **Continuous Delivery**
* **Continuous Deployment**

Although they are similar, there is an important difference.

## Continuous Delivery

With **Continuous Delivery**, the application is automatically built and prepared for deployment.

However, a human may need to approve the final deployment.

Example:

```text
Push code
    ↓
Run tests
    ↓
Build application
    ↓
Create Docker image
    ↓
Push image to container registry
    ↓
Wait for manual approval
    ↓
Deploy to production
```

This gives organizations more control over production deployments.

---

## Continuous Deployment

With **Continuous Deployment**, the entire process is automated.

If the code passes all required tests and checks, it is automatically deployed to production.

Example:

```text
Developer pushes code
            ↓
       Run tests
            ↓
      Tests pass?
       ↙      ↘
     No        Yes
     ↓          ↓
 Stop      Build application
                  ↓
          Create Docker image
                  ↓
          Push image to registry
                  ↓
          Deploy automatically
                  ↓
             Production
```

There is no manual approval required between the successful tests and deployment.

---

# GitHub Actions and CI/CD

GitHub Actions is the tool that can implement a CI/CD pipeline inside GitHub.

For example:

```text
GitHub Repository
        │
        │ Developer pushes code
        ▼
┌───────────────────────┐
│    GitHub Actions     │
│       Workflow        │
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│        Test           │
│    Run Unit Tests     │
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│        Build          │
│  Build Application    │
└───────────────────────┘
        │
        ▼
┌───────────────────────┐
│        Deploy         │
│    Deploy to AWS      │
└───────────────────────┘
```

This entire process can happen automatically when an event occurs in your repository.

---

# Real-World Example: Python Application

Imagine you have this repository:

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
        └── ci.yml
```

The important directory for GitHub Actions is:

```text
.github/workflows/
```

GitHub Actions looks inside this directory for workflow files.

The workflow files are usually written in **YAML**.

For example:

```text
.github/workflows/ci.yml
```

---

# Example GitHub Actions Workflow

```yaml
name: Python CI Pipeline

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
```

Now let's break down exactly what happens.

---

# Step 1: Workflow Name

```yaml
name: Python CI Pipeline
```

This gives the workflow a name.

You will see this name inside the **Actions** section of your GitHub repository.

For example:

```text
Actions
└── Python CI Pipeline
```

The name does not control how the workflow works. It simply makes the workflow easier to identify.

---

# Step 2: The Trigger

```yaml
on:
  push:
    branches:
      - main
```

The `on` section defines the **event that triggers the workflow**.

In this example, the workflow starts whenever someone pushes code to the `main` branch.

For example:

```text
Developer
    │
    │ git push origin main
    ▼
GitHub Repository
    │
    │ Push event detected
    ▼
GitHub Actions Workflow Starts
```

You can trigger workflows from many different events.

Examples include:

```yaml
on: push
```

Run when code is pushed.

```yaml
on: pull_request
```

Run when a pull request is created or updated.

```yaml
on: workflow_dispatch
```

Allow a user to manually start the workflow.

```yaml
on:
  schedule:
    - cron: "0 0 * * *"
```

Run on a schedule.

This could be useful for tasks such as:

* Daily backups
* Scheduled security scans
* Running Terraform checks
* Generating reports

---

# Step 3: Jobs

```yaml
jobs:
  test:
```

A **job** is a collection of steps that perform a specific task.

In this example:

```text
jobs
└── test
```

The job is named `test`.

You can have multiple jobs in a workflow.

For example:

```text
jobs:
  test:
    # Run tests

  build:
    # Build application

  deploy:
    # Deploy application
```

A complete CI/CD pipeline could look like this:

```text
Workflow
│
├── Test Job
│
├── Build Job
│
└── Deploy Job
```

Jobs can run in parallel or depend on other jobs.

For example:

```text
Test
  ↓
Build
  ↓
Deploy
```

We can define those dependencies later using:

```yaml
needs:
```

---

# Step 4: Runner

```yaml
runs-on: ubuntu-latest
```

A **runner** is the machine that executes your workflow.

In this example, GitHub provides a temporary machine running the latest supported Ubuntu environment.

The workflow runs inside that machine.

Conceptually:

```text
GitHub Actions
      │
      ▼
Creates Ubuntu Runner
      │
      ▼
Downloads Your Repository
      │
      ▼
Runs Workflow Commands
      │
      ▼
Workflow Completes
      │
      ▼
Runner Is Removed
```

Other runner options can include:

```yaml
runs-on: ubuntu-latest
```

```yaml
runs-on: windows-latest
```

```yaml
runs-on: macos-latest
```

You can also configure your own machine as a **self-hosted runner**.

---

# Step 5: Steps

```yaml
steps:
```

A job contains one or more **steps**.

Each step performs an individual task.

For example:

```text
Test Job
│
├── Step 1: Download repository
│
├── Step 2: Install Python
│
├── Step 3: Install dependencies
│
└── Step 4: Run tests
```

Let's examine each step.

---

# Step 6: Checkout the Repository

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
```

The runner starts as a fresh machine.

Your repository code is **not automatically available** on the runner.

This step downloads your repository onto the runner.

```text
GitHub Repository
        │
        │ actions/checkout
        ▼
GitHub Actions Runner
        │
        ▼
Your application files
```

The `uses` keyword means we are using an existing GitHub Action.

```yaml
uses: actions/checkout@v4
```

This action is provided by GitHub and performs the repository checkout process.

Think of it similar to running:

```bash
git clone <repository>
```

but the GitHub Action handles the process for the workflow.

---

# Step 7: Set Up Python

```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: "3.12"
```

This step installs or configures Python 3.12 on the runner.

The workflow is using another pre-built GitHub Action:

```text
actions/setup-python
```

The `with` section provides configuration to that action.

```yaml
with:
  python-version: "3.12"
```

Conceptually:

```text
GitHub Action
      │
      ▼
actions/setup-python
      │
      ▼
Configuration:
Python version = 3.12
      │
      ▼
Python is ready
```

---

# Step 8: Install Dependencies

```yaml
- name: Install dependencies
  run: pip install -r requirements.txt
```

The `run` keyword executes a command on the runner.

This is similar to opening a terminal and running:

```bash
pip install -r requirements.txt
```

The command installs the dependencies required by the application.

For example:

```text
requirements.txt
│
├── flask
├── pytest
└── requests
```

GitHub Actions installs these packages before running the tests.

---

# Step 9: Run Tests

```yaml
- name: Run tests
  run: pytest
```

This command executes your automated tests.

Conceptually:

```text
Run pytest
    │
    ├── Test 1 → PASS
    ├── Test 2 → PASS
    ├── Test 3 → PASS
    └── Test 4 → PASS
```

If all tests pass:

```text
Workflow Status: SUCCESS ✅
```

If one or more tests fail:

```text
Workflow Status: FAILED ❌
```

This gives developers immediate feedback about their changes.

---

# Complete Flow

Let's put everything together.

A developer runs:

```bash
git add .
git commit -m "Add new login feature"
git push origin main
```

Then the following happens automatically:

```text
1. Developer pushes code to main
                ↓
2. GitHub detects the push event
                ↓
3. GitHub Actions starts the workflow
                ↓
4. GitHub creates an Ubuntu runner
                ↓
5. Runner downloads the repository
                ↓
6. Python 3.12 is configured
                ↓
7. Dependencies are installed
                ↓
8. Automated tests are executed
                ↓
9. Results are reported in GitHub
```

This is a basic **Continuous Integration pipeline**.

---

# CI/CD Example with Docker and AWS

A more realistic DevOps pipeline might look like this:

```text
Developer pushes code
        │
        ▼
┌───────────────────┐
│    GitHub Push    │
└───────────────────┘
        │
        ▼
┌───────────────────┐
│   GitHub Actions  │
│     CI Pipeline   │
└───────────────────┘
        │
        ▼
   Run Tests
        │
        ▼
   Tests Pass?
    │       │
   No      Yes
    │       │
    ▼       ▼
  STOP   Build Docker Image
                │
                ▼
        Push Image to Amazon ECR
                │
                ▼
        Deploy New Version
                │
                ▼
            Amazon ECS
                │
                ▼
          Application Updated
```

In this example:

### CI:

```text
Push Code
    ↓
Run Tests
    ↓
Build Application
    ↓
Build Docker Image
```

### CD:

```text
Push Docker Image to ECR
    ↓
Deploy to ECS
    ↓
Application Updated
```

This is a complete example of how GitHub Actions can automate a modern DevOps CI/CD pipeline.

---

# Important GitHub Actions Concepts

As you continue learning, these are the main concepts you should understand:

| Concept      | Description                                     |
| ------------ | ----------------------------------------------- |
| **Workflow** | An automated process defined in a YAML file     |
| **Event**    | Something that triggers a workflow              |
| **Trigger**  | The condition that starts a workflow            |
| **Job**      | A group of related steps                        |
| **Step**     | An individual task inside a job                 |
| **Runner**   | The machine that executes the job               |
| **Action**   | Reusable automation that performs a task        |
| **CI**       | Automatically integrates and tests code changes |
| **CD**       | Automatically delivers or deploys applications  |

---

# Mental Model

A simple way to remember GitHub Actions is:

```text
EVENT
  ↓
WORKFLOW
  ↓
JOB
  ↓
RUNNER
  ↓
STEPS
  ↓
ACTIONS / COMMANDS
```

For example:

```text
Push to GitHub
      ↓
Workflow: CI Pipeline
      ↓
Job: Test Application
      ↓
Runner: Ubuntu
      ↓
Step 1: Checkout Code
Step 2: Set Up Python
Step 3: Install Dependencies
Step 4: Run Tests
```

---

# Key Takeaway

> **GitHub Actions is GitHub's automation platform that allows you to build CI/CD pipelines and automate tasks directly from your repository.**

The most important idea is:

```text
Something happens in GitHub
          ↓
      Event occurs
          ↓
Workflow is triggered
          ↓
Jobs are created
          ↓
Jobs run on runners
          ↓
Steps execute Actions or commands
          ↓
Task completes
```

For a DevOps workflow, this commonly becomes:

```text
Code
  ↓
Push
  ↓
Test
  ↓
Build
  ↓
Package
  ↓
Deploy
  ↓
Monitor
```

This automation allows teams to release software faster, reduce manual work, and catch problems earlier in the development process.
