# GitHub Actions Summary — What I Have Learned So Far

So far, I have learned the fundamental building blocks of **GitHub Actions** and how they work together to create automated **CI/CD pipelines**.

## 1. What GitHub Actions Is

GitHub Actions is a CI/CD and automation platform built into GitHub. It allows me to automate tasks when specific events occur in my repository.

For example:

```text
Developer pushes code
        ↓
GitHub event occurs
        ↓
Workflow starts
        ↓
Code is tested
        ↓
Application is built
        ↓
Application can be deployed
```

---

## 2. Workflows

A **workflow** is an automated process defined in a YAML file inside the repository.

A workflow contains:

```text
Workflow
   │
   ├── Trigger/Event
   │
   └── Jobs
         │
         └── Steps
```

For example:

```yaml
name: Test Project

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running tests"
```

---

## 3. Workflow Triggers

The `on` keyword determines **when a workflow starts**.

For example:

```yaml
on: push
```

Automatically runs the workflow when code is pushed.

I also learned:

```yaml
on: workflow_dispatch
```

which allows me to manually start a workflow from the GitHub Actions page.

Both can be used together:

```yaml
on: [push, workflow_dispatch]
```

This means the workflow can start either automatically after a push or manually from GitHub.

---

## 4. Jobs

A workflow can contain one or more jobs.

```yaml
jobs:
  test-job:
    ...

  deploy:
    ...
```

Each job is a separate unit of work with its own execution environment.

For example:

```text
Workflow
   │
   ├── test-job
   │
   └── deploy
```

Each job normally requires its own runner.

---

## 5. Runners

A runner is the machine or execution environment that performs the job.

For example:

```yaml
runs-on: ubuntu-latest
```

means GitHub provides an Ubuntu environment to execute the job.

A key concept I learned is that jobs are isolated.

```text
test-job → Runner A

deploy   → Runner B
```

Files, installed software, and dependencies from one job are not automatically available to another job.

---

## 6. Steps

A job contains one or more steps.

```yaml
steps:
  - name: Install dependencies
    run: npm ci

  - name: Run tests
    run: npm test
```

Steps within a job normally run sequentially:

```text
Step 1
   ↓
Step 2
   ↓
Step 3
```

Each step performs a specific task.

---

## 7. `run` vs `uses`

I learned that there are two common ways to execute work in a step.

### `run`

Used to execute shell commands or scripts:

```yaml
- name: Run tests
  run: npm test
```

### `uses`

Used to run a reusable GitHub Action:

```yaml
- name: Get repository code
  uses: actions/checkout@v3
```

I also used:

```yaml
uses: actions/setup-node@v3
```

to install and configure Node.js.

---

## 8. The `with` Keyword

The `with` keyword provides configuration or inputs to an action.

For example:

```yaml
- uses: actions/setup-node@v3
  with:
    node-version: 18
```

Here, `with` passes the Node.js version to the setup action.

---

## 9. Building a CI Workflow

I created a Node.js testing workflow that performs the following process:

```text
Push Code
    ↓
Get Repository Code
    ↓
Install Node.js
    ↓
Install Dependencies
    ↓
Run Tests
```

The main commands were:

```yaml
run: npm ci
```

to install dependencies using the project's lock file, and:

```yaml
run: npm test
```

to execute the application's tests.

---

## 10. Multiple Jobs

I expanded the workflow to contain multiple jobs:

```text
Workflow
   │
   ├── test-job
   │
   └── deploy
```

Jobs without dependencies can run independently and potentially in parallel.

```text
              Workflow
              /      \
             ▼        ▼
        test-job    deploy
```

This is useful when different tasks can be executed at the same time.

---

## 11. The Importance of `needs`

I learned that `needs` creates a dependency between jobs.

For example:

```yaml
deploy:
  needs: test-job
```

This means:

```text
test-job
    │
    ├── Tests fail ❌ → deploy is skipped
    │
    └── Tests pass ✅
             │
             ▼
          deploy
```

This is important for CI/CD because I usually do not want to deploy an application before testing has successfully completed.

A job can also depend on multiple jobs:

```yaml
needs: [test-job, security-job, lint-job]
```

In this case, all required jobs must complete successfully before the dependent job runs.

---

## 12. Job Dependencies and Parallel Execution

I learned that I can combine parallel and sequential execution.

For example:

```text
              ┌── Test ──────┐
              │              │
Start ────────┼── Security ──┤
              │              │
              └── Lint ──────┘
                     │
                     ▼
                   Build
                     │
                     ▼
                   Deploy
```

The independent jobs can run in parallel, while later jobs wait for required jobs using `needs`.

---

## 13. GitHub Actions Context and Expressions

I learned that GitHub provides information about the current workflow execution through **contexts**.

For example:

```yaml
${{ github.event_name }}
```

can tell me what event triggered the workflow.

Examples:

```text
push
workflow_dispatch
```

I also learned the expression syntax:

```yaml
${{ }}
```

which tells GitHub Actions to evaluate an expression.

Examples of useful GitHub context information include:

```text
github.actor
github.repository
github.event_name
github.ref
```

These can provide information such as:

```text
Who triggered the workflow?
Which repository is running?
What event triggered the workflow?
Which branch is involved?
```

---

## 14. Inspecting the GitHub Context

I created a workflow to output the GitHub context:

```yaml
- name: Output GitHub context
  run: echo "${{ toJson(github) }}"
```

Here:

```text
github
   ↓
GitHub context information

toJson(github)
   ↓
Converts the context into JSON

echo
   ↓
Prints the information in the workflow logs
```

This is useful for understanding what information GitHub makes available during a workflow run.

---

# Current Overall Mental Model

At this point, I understand GitHub Actions using this structure:

```text
GitHub Event
      │
      ▼
┌───────────────┐
│   WORKFLOW    │
│               │
│ on: push      │
│ or            │
│ workflow_     │
│ dispatch      │
└───────┬───────┘
        │
        ▼
┌─────────────────────────┐
│          JOBS           │
│                         │
│  test-job ── needs ──►  │
│                  deploy │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│         RUNNERS         │
│                         │
│ Each job gets its own   │
│ execution environment   │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│         STEPS           │
│                         │
│ checkout                │
│ setup Node.js           │
│ npm ci                  │
│ npm test                │
│ npm run build           │
│ deploy                  │
└─────────────────────────┘
```

# Key Takeaway

The main concepts I have learned so far are:

> **`on` controls when a workflow starts. A workflow contains jobs. Jobs run on runners and contain steps. Steps can execute commands using `run` or reusable actions using `uses`. Independent jobs can run in parallel, while `needs` creates dependencies between jobs. GitHub contexts provide information about the current workflow run and can be accessed using expressions such as `${{ }}`.**

This gives me the foundation needed to start building more realistic GitHub Actions CI/CD pipelines.
