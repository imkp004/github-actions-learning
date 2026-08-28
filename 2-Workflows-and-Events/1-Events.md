# GitHub Actions Events

## What Are Events in GitHub Actions?

An **event** is a specific activity or change that happens in GitHub.

GitHub Actions can listen for these events and automatically start a workflow when they occur.

The basic flow is:

```text
Something happens in GitHub
        ↓
GitHub generates an event
        ↓
GitHub Actions detects the event
        ↓
A matching workflow is triggered
        ↓
Jobs begin running
```

For example:

```text
Developer pushes code
        ↓
push event
        ↓
GitHub Actions checks workflows
        ↓
Workflow with `on: push` is triggered
        ↓
Tests run automatically
```

The `on` keyword is used to define which event or events can trigger a workflow.

---

# Basic Event Syntax

The simplest example is:

```yaml
on: push
```

This means:

> Run this workflow whenever a `push` event occurs.

For example:

```yaml
name: Run Tests

on: push

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - run: echo "Running tests..."
```

The flow is:

```text
git push
    ↓
push event
    ↓
Workflow triggered
    ↓
test job starts
    ↓
Tests run
```

---

# Multiple Events

A workflow can listen for more than one event.

For example:

```yaml
on: [push, workflow_dispatch]
```

This means:

```text
          push event
              │
              │
              ▼
        ┌─────────────┐
        │  WORKFLOW   │
        └─────────────┘
              ▲
              │
              │
    workflow_dispatch
```

The workflow can be started either:

* Automatically when code is pushed.
* Manually from the GitHub Actions UI.

---

# Main Types of GitHub Actions Events

There are many GitHub events, but the most commonly used ones can be grouped into the following categories:

```text
GitHub Events
│
├── Code Events
│   ├── push
│   ├── pull_request
│   └── pull_request_target
│
├── Manual Events
│   └── workflow_dispatch
│
├── Scheduled Events
│   └── schedule
│
├── Workflow Events
│   ├── workflow_run
│   └── workflow_call
│
├── Issue and Project Events
│   ├── issues
│   └── issue_comment
│
└── Repository Events
    ├── release
    └── create
```

The most important events to learn first are:

```text
1. push
2. pull_request
3. workflow_dispatch
4. schedule
5. workflow_call
6. workflow_run
7. release
```

---

# 1. `push` Event

The `push` event is triggered when commits or tags are pushed to a repository.

```yaml
on: push
```

Example:

```text
Developer changes code
        ↓
git add .
        ↓
git commit -m "Add login feature"
        ↓
git push
        ↓
push event occurs
        ↓
Workflow starts
```

A common use case is Continuous Integration:

```yaml
name: CI

on: push

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
```

Every push automatically triggers the tests.

---

## `push` on Specific Branches

Often, I do not want a workflow to run on every branch.

For example:

```yaml
on:
  push:
    branches:
      - main
```

This means:

```text
Push to main
    ↓
Workflow runs ✅


Push to feature branch
    ↓
Workflow does not run ❌
```

I can also specify multiple branches:

```yaml
on:
  push:
    branches:
      - main
      - develop
```

Now:

```text
main      → Workflow runs
develop   → Workflow runs
feature/* → Does not run
```

---

## Branch Patterns

I can also use patterns.

For example:

```yaml
on:
  push:
    branches:
      - 'feature/**'
```

This can match branches such as:

```text
feature/login
feature/payment
feature/api/v2
```

This is useful when I want different workflows for different types of branches.

---

## Ignoring Branches

I can also ignore branches:

```yaml
on:
  push:
    branches-ignore:
      - main
```

This means:

```text
Push to main
    ↓
Do not run


Push to another branch
    ↓
Run workflow
```

---

# 2. `pull_request` Event

The `pull_request` event is one of the most important events for CI.

It can run a workflow when activity occurs on a pull request.

Basic syntax:

```yaml
on: pull_request
```

Typical workflow:

```text
Developer creates feature branch
        ↓
Developer writes code
        ↓
Developer pushes changes
        ↓
Developer creates Pull Request
        ↓
pull_request event
        ↓
CI workflow runs
        ↓
Tests / Linting / Security checks
        ↓
Pull Request can be reviewed
```

Example:

```yaml
name: Pull Request Checks

on: pull_request

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 18

      - run: npm ci
      - run: npm test
```

This is useful because it checks the code **before it is merged**.

---

## `pull_request` for Specific Target Branches

For example:

```yaml
on:
  pull_request:
    branches:
      - main
```

This does **not** mean the workflow only runs when the source branch is `main`.

It means the workflow runs when a pull request targets the `main` branch.

For example:

```text
feature/login ────── Pull Request ──────► main
                                             │
                                             ▼
                                      Workflow runs
```

But:

```text
feature/login ────── Pull Request ──────► develop
                                             │
                                             ▼
                                  Workflow does not run
```

This is an important distinction.

---

# `push` vs `pull_request`

These two events are commonly used together.

```yaml
on: [push, pull_request]
```

However, they serve slightly different purposes.

## `push`

```text
Code is pushed
      ↓
Run CI
```

Useful for checking changes after commits are pushed.

## `pull_request`

```text
Pull Request activity
      ↓
Run CI
      ↓
Validate code before merge
```

A common workflow is:

```text
Developer
   │
   ▼
Feature Branch
   │
   ▼
Push Code ────────────────► push event
   │                            │
   │                            ▼
   │                         Run tests
   │
   ▼
Create Pull Request ──────► pull_request event
                                │
                                ▼
                             Run tests
                                │
                                ▼
                             Review
                                │
                                ▼
                               Merge
```

---

# 3. `workflow_dispatch` Event

`workflow_dispatch` allows a workflow to be manually triggered.

```yaml
on: workflow_dispatch
```

The process is:

```text
GitHub Repository
        ↓
Actions Tab
        ↓
Select Workflow
        ↓
Run Workflow
        ↓
Workflow starts manually
```

This is useful when I do not want to create a new commit just to run a workflow.

For example:

```yaml
name: Manual Deployment

on: workflow_dispatch

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - run: echo "Starting manual deployment..."
```

---

## Manual Inputs

`workflow_dispatch` can also accept inputs.

For example:

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment to deploy to"
        required: true
        type: choice
        options:
          - development
          - staging
          - production
```

When I manually run the workflow, GitHub can ask me:

```text
Which environment?

[development ▼]
```

The selected value can then be used in the workflow.

For example:

```yaml
steps:
  - name: Show environment
    run: echo "Deploying to ${{ inputs.environment }}"
```

If I select:

```text
production
```

the output would be conceptually:

```text
Deploying to production
```

This makes `workflow_dispatch` very useful for controlled manual deployments.

---

# 4. `schedule` Event

The `schedule` event runs workflows automatically at scheduled times.

It uses **cron syntax**.

Example:

```yaml
on:
  schedule:
    - cron: '0 0 * * *'
```

This runs at midnight according to GitHub's cron schedule behavior.

The workflow concept is:

```text
Scheduled Time
      ↓
GitHub triggers workflow
      ↓
Job runs automatically
```

Example use cases:

```text
Daily backup
Daily security scan
Weekly cleanup
Monthly report
Dependency checks
```

Example:

```yaml
name: Daily Security Scan

on:
  schedule:
    - cron: '0 0 * * *'

jobs:
  security:
    runs-on: ubuntu-latest

    steps:
      - run: echo "Running scheduled security scan..."
```

---

## Understanding Cron

Cron uses five fields:

```text
┌──────────── minute
│ ┌────────── hour
│ │ ┌──────── day of month
│ │ │ ┌────── month
│ │ │ │ ┌──── day of week
│ │ │ │ │
* * * * *
```

For example:

```text
0 0 * * *
```

means:

```text
At minute 0
At hour 0
Every day
Every month
Every day of the week
```

Another example:

```text
0 9 * * 1
```

means:

```text
9:00
Every Monday
```

A schedule can also have multiple cron entries:

```yaml
on:
  schedule:
    - cron: '0 9 * * 1-5'
    - cron: '0 18 * * 1-5'
```

Conceptually:

```text
Monday-Friday

9:00  → Run workflow
18:00 → Run workflow
```

---

# 5. `workflow_call` Event

`workflow_call` allows one workflow to be called by another workflow.

This is used to create **reusable workflows**.

Instead of copying the same CI logic into multiple workflow files, I can create one reusable workflow.

Example reusable workflow:

```yaml
name: Reusable Test Workflow

on:
  workflow_call:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
```

Conceptually:

```text
Workflow A
    │
    │ calls
    ▼
Reusable Workflow
    │
    ▼
Run Tests
```

This is useful in large organizations.

For example:

```text
Application A ──┐
                │
Application B ──┼──► Reusable CI Workflow
                │
Application C ──┘
```

Instead of maintaining the same workflow logic in many places, the CI process can be standardized.

---

# 6. `workflow_run` Event

`workflow_run` allows a workflow to start after another workflow runs.

For example:

```yaml
on:
  workflow_run:
    workflows: ["CI"]
    types:
      - completed
```

This means:

```text
CI Workflow
     │
     ▼
Completes
     │
     ▼
workflow_run event
     │
     ▼
Second workflow starts
```

Example:

```text
Workflow 1
    │
    ▼
Run Tests
    │
    ▼
Complete
    │
    ▼
Workflow 2
    │
    ▼
Deploy
```

This is useful when I want to separate a large CI/CD process into multiple workflow files.

---

# 7. `release` Event

The `release` event occurs when a GitHub release is created or published.

Example:

```yaml
on:
  release:
    types:
      - published
```

Conceptually:

```text
New Release Published
        │
        ▼
release event
        │
        ▼
Workflow starts
        │
        ▼
Build / Publish / Deploy
```

Example use cases:

```text
Publish package
Build release files
Deploy production application
Create release documentation
```

---

# 8. `issues` Event

GitHub Actions can also react to issue activity.

For example:

```yaml
on:
  issues:
    types:
      - opened
```

This means:

```text
New Issue Created
       ↓
issues event
       ↓
Workflow runs
```

Example:

```yaml
jobs:
  issue-info:
    runs-on: ubuntu-latest

    steps:
      - run: echo "A new issue was created"
```

This can be used for automation such as:

```text
New Issue
    ↓
Automatically add label
    ↓
Assign team
    ↓
Send notification
```

---

# 9. `issue_comment` Event

This event can trigger a workflow when someone comments on an issue or pull request.

For example:

```yaml
on:
  issue_comment:
    types:
      - created
```

Conceptually:

```text
User creates comment
        ↓
issue_comment event
        ↓
Workflow starts
```

This can be used to create command-based automation.

For example:

```text
Comment:

/deploy staging
```

The workflow could detect the command and perform a deployment.

---

# 10. `create` Event

The `create` event occurs when certain GitHub resources, such as branches or tags, are created.

```yaml
on: create
```

Conceptually:

```text
New branch or tag created
          ↓
create event
          ↓
Workflow starts
```

A common use case is reacting to new version tags.

For example:

```text
Create tag: v1.0.0
        ↓
create event
        ↓
Build release
```

---

# Event Activity Types

Some events support **activity types**.

Instead of triggering for every possible activity, I can select specific actions.

For example:

```yaml
on:
  pull_request:
    types:
      - opened
      - closed
```

The workflow runs when a pull request is:

```text
Opened
  OR
Closed
```

I can visualize it as:

```text
Pull Request Events
│
├── opened      → Run workflow
├── closed      → Run workflow
├── edited      → Ignore
├── labeled     → Ignore
└── other       → Ignore
```

Another example:

```yaml
on:
  issues:
    types:
      - opened
      - labeled
```

The workflow only responds to those selected activities.

---

# Combining Different Events

A real workflow can use multiple events and configure each one.

For example:

```yaml
name: Application CI

on:
  push:
    branches:
      - main

  pull_request:
    branches:
      - main

  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
```

This workflow can start in three ways:

```text
Push to main
     │
     ├──────────────┐
     │              │
Pull Request        │
to main             │
     │              │
     └──────┬───────┘
            │
            ▼
       CI Workflow
            ▲
            │
     Manual Trigger
```

This is a common CI workflow design.

---

# Event Filtering

GitHub Actions events can often be filtered so workflows only run when relevant changes occur.

This helps avoid unnecessary workflow runs.

## Branch Filtering

```yaml
on:
  push:
    branches:
      - main
```

Only run when pushing to `main`.

---

## Path Filtering

I can also trigger workflows only when certain files change.

Example:

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
```

The workflow runs when:

```text
src/app.js changed        → Run
package.json changed      → Run
README.md changed         → Do not run
```

This is useful in large repositories.

For example:

```text
Repository
│
├── frontend/
├── backend/
└── infrastructure/
```

I could create separate workflows.

Frontend workflow:

```yaml
on:
  push:
    paths:
      - 'frontend/**'
```

Backend workflow:

```yaml
on:
  push:
    paths:
      - 'backend/**'
```

Infrastructure workflow:

```yaml
on:
  push:
    paths:
      - 'infrastructure/**'
```

Now a frontend change does not unnecessarily trigger every pipeline.

---

# Event and Context Connection

When an event triggers a workflow, GitHub makes information about that event available through contexts.

For example:

```yaml
on: [push, workflow_dispatch]
```

I can check what started the workflow:

```yaml
${{ github.event_name }}
```

Conceptually:

```text
Workflow starts
       │
       ▼
What triggered it?
       │
       ├── push
       │     │
       │     ▼
       │  github.event_name = push
       │
       └── workflow_dispatch
             │
             ▼
    github.event_name = workflow_dispatch
```

This allows the workflow to make decisions based on the event.

---

# Example: Different Behavior for Different Events

```yaml
steps:
  - name: Push message
    if: github.event_name == 'push'
    run: echo "Workflow started because code was pushed"

  - name: Manual message
    if: github.event_name == 'workflow_dispatch'
    run: echo "Workflow was started manually"
```

The workflow changes behavior depending on the event that triggered it.

---

# Most Important Events to Remember

| Event               | What Triggers It                | Common Use                   |
| ------------------- | ------------------------------- | ---------------------------- |
| `push`              | Code or tags are pushed         | Continuous Integration       |
| `pull_request`      | Pull request activity occurs    | Test code before merge       |
| `workflow_dispatch` | Manual user action              | Manual testing or deployment |
| `schedule`          | A defined time                  | Backups, scans, cleanup      |
| `workflow_call`     | Another workflow calls it       | Reusable workflows           |
| `workflow_run`      | Another workflow runs/completes | Multi-workflow pipelines     |
| `release`           | Release activity occurs         | Production releases          |
| `issues`            | Issue activity occurs           | Issue automation             |
| `issue_comment`     | Comment activity occurs         | Command-based automation     |
| `create`            | Branch or tag is created        | Branch/tag automation        |

---

# My Mental Model for Events

I should think of an event as the **starting signal** for a workflow.

```text
                    Something happens
                           │
                           ▼
                       GitHub Event
                           │
                           ▼
                    Does `on:` match?
                       /       \
                     Yes        No
                      │          │
                      ▼          ▼
              Start Workflow   Do nothing
                      │
                      ▼
                    Jobs
                      │
                      ▼
                    Steps
```

The most important concept is:

> **Events determine when a workflow starts. The `on` keyword tells GitHub Actions which events to listen for. Events can be configured and filtered by branches, paths, and activity types to ensure workflows run only when necessary.**

For most CI/CD workflows, the events I will use most frequently are:

```text
push
pull_request
workflow_dispatch
schedule
workflow_call
workflow_run
```

These events provide the foundation for automatically running tests, validating pull requests, scheduling tasks, manually triggering deployments, and creating reusable CI/CD workflows.
