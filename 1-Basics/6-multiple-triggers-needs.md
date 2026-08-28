# GitHub Actions: Multiple Triggers, `needs`, and GitHub Context

In this section, I learned three important GitHub Actions concepts:

1. A workflow can have **multiple triggers**.
2. The `needs` keyword controls **job dependencies and execution flow**.
3. GitHub provides **context information** that can be accessed using expressions such as `${{ github }}`.

---

# 1. Multiple Workflow Triggers

My workflow uses:

```yaml
on: [push, workflow_dispatch]
```

This means the workflow can be triggered in **two different ways**.

```text
                Workflow
                    │
           ┌────────┴────────┐
           │                 │
           ▼                 ▼
         push        workflow_dispatch
           │                 │
           └────────┬────────┘
                    ▼
             Workflow starts
```

## Trigger 1: `push`

```yaml
push
```

The workflow starts automatically when code is pushed to the repository.

For example:

```text
Developer changes code
        ↓
git add .
        ↓
git commit
        ↓
git push
        ↓
Push event occurs
        ↓
GitHub Actions starts workflow automatically
```

This is useful for normal Continuous Integration because every code change can automatically be tested.

---

## Trigger 2: `workflow_dispatch`

```yaml
workflow_dispatch
```

This allows me to manually start the workflow from the GitHub Actions interface.

```text
GitHub Repository
       ↓
Actions tab
       ↓
Select workflow
       ↓
Run workflow
       ↓
Workflow starts
```

This is useful when I want to run the workflow **without making a new commit or pushing new code**.

For example, imagine the last push happened yesterday, but today I want to manually test the workflow again.

With only:

```yaml
on: push
```

I would need a new push event.

With:

```yaml
on: [push, workflow_dispatch]
```

I can simply manually start it.

---

# Why Multiple Triggers Are Useful

The workflow now supports both:

```text
AUTOMATIC EXECUTION
        │
        └── push


MANUAL EXECUTION
        │
        └── workflow_dispatch
```

This gives me flexibility.

For example:

```text
Normal development:
Push code
    ↓
Workflow runs automatically


Troubleshooting:
No code changes
    ↓
Manually run workflow
    ↓
Workflow runs
```

The same workflow definition can respond to different GitHub events.

---

# Short Form vs Long Form

My current syntax is the short form:

```yaml
on: [push, workflow_dispatch]
```

This is equivalent to:

```yaml
on:
  push:
  workflow_dispatch:
```

Both mean the workflow is triggered by either event.

The short form is convenient when I do not need to configure the triggers.

---

# When the Long Form Is Better

Suppose I only want the workflow to run automatically when code is pushed to the `main` branch.

I can configure the `push` trigger:

```yaml
on:
  push:
    branches:
      - main

  workflow_dispatch:
```

Now:

```text
Push to main
     ↓
Workflow runs automatically ✅


Push to feature branch
     ↓
Workflow does not automatically run ❌


Manual Run Workflow
     ↓
Workflow runs from the GitHub UI ✅
```

The important concept is:

> **The `on` section defines one or more events that can start a workflow.**

---

# 2. The Importance of `needs`

My workflow contains:

```yaml
deploy:
  needs: test-job
```

This is extremely important because it creates a dependency between jobs.

The deploy job is saying:

> Do not start me until `test-job` completes successfully.

The execution flow becomes:

```text
test-job
    │
    ▼
Tests execute
    │
    ├── FAIL ❌
    │      │
    │      ▼
    │   deploy does not run
    │
    └── PASS ✅
           │
           ▼
        deploy starts
```

---

# `needs` Protects the Deployment Process

Without `needs`, the workflow could look like this:

```text
             Workflow Starts
                  │
          ┌───────┴───────┐
          ▼               ▼
      test-job          deploy
          │               │
          │               ├── Build
          │               └── Deploy
          │
          └── Run tests
```

The two jobs are independent.

This means deployment could potentially begin while testing is still happening.

That is usually not what I want in a CI/CD pipeline.

With:

```yaml
needs: test-job
```

the flow becomes:

```text
Workflow
    │
    ▼
test-job
    │
    ▼
Are tests successful?
    │
    ├── No ❌
    │     │
    │     └── Stop pipeline
    │
    └── Yes ✅
          │
          ▼
        deploy
          │
          ├── Build
          │
          └── Deploy
```

This is why `needs` is a fundamental CI/CD safety mechanism.

---

# `needs` Creates a Gate

A good way to think about `needs` is as a **gate**.

```text
             TEST GATE
                 │
        ┌────────┴────────┐
        │                 │
     FAIL ❌            PASS ✅
        │                 │
        ▼                 ▼
      STOP             DEPLOY
```

The deployment job is blocked until the required job succeeds.

For example:

```yaml
deploy:
  needs: test-job
```

means:

```text
deploy
  ↑
  │ Permission to start
  │
test-job must succeed
```

---

# Multiple Job Dependencies

I can make one job depend on multiple jobs.

For example:

```yaml
deploy:
  needs: [test-job, security-scan, lint]
```

This means:

> The deploy job must wait for all three jobs.

The workflow can look like:

```text
                 Workflow
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
      test-job    security     lint
          │          │          │
          └──────────┼──────────┘
                     │
              All must succeed
                     │
                     ▼
                   deploy
```

This is useful for production pipelines.

For example:

```text
Unit Tests          → Must pass
Security Scan       → Must pass
Code Quality Check  → Must pass
           │
           ▼
        Deploy
```

The deploy job acts as the final stage.

---

# Example: Multiple `needs`

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

  lint:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Checking code quality"

  deploy:
    needs: [test, security, lint]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying application"
```

Execution:

```text
Start
  │
  ├─────────────┬──────────────┐
  ▼             ▼              ▼
Test         Security         Lint
  │             │              │
  └─────────────┴──────────────┘
                │
         All successful?
                │
              Yes
                │
                ▼
              Deploy
```

---

# `needs` Can Also Create a Complex Workflow

A workflow does not have to be one straight line.

For example:

```text
                     Start
                       │
              ┌────────┴────────┐
              ▼                 ▼
           Backend            Frontend
              │                 │
              ▼                 ▼
          API Tests         UI Tests
              │                 │
              └────────┬────────┘
                       ▼
                    Build
                       │
                       ▼
                    Deploy
```

GitHub Actions workflows can be designed as a **dependency graph**.

`needs` is one of the main tools used to define that graph.

---

# Important Rule: Default `needs` Behavior

If a job listed in `needs` fails, the dependent job is normally skipped.

For example:

```yaml
deploy:
  needs: test
```

If:

```text
test ❌
```

then normally:

```text
deploy ⏭️ skipped
```

The deploy job does not run because its required job did not complete successfully.

This is exactly what makes `needs` valuable for deployment pipelines.

---

# 3. Understanding the GitHub Context

I created another workflow:

```yaml
name: output info

on: workflow_dispatch

jobs:
  info:
    runs-on: ubuntu-latest

    steps:
      - name: output github context
        run: echo "${{ toJson(github) }}"
```

This workflow is designed to inspect information provided by GitHub.

The important line is:

```yaml
${{ toJson(github) }}
```

To understand this, I need to understand **contexts** and **expressions**.

---

# What Is a GitHub Actions Context?

A context is information that GitHub makes available to a workflow while it is running.

Think of GitHub providing information like:

```text
Who triggered the workflow?
Which repository is running?
Which branch is involved?
Which commit triggered the workflow?
What event triggered the workflow?
What is the run ID?
```

The `github` context provides GitHub-related information about the current workflow run.

Conceptually:

```text
GitHub Event
     │
     ▼
GitHub creates context information
     │
     ▼
Workflow can access that information
```

---

# What Is `${{ }}`?

The syntax:

```yaml
${{ }}
```

is called a GitHub Actions expression.

It tells GitHub:

> Evaluate what is inside these braces and replace it with its value.

For example:

```yaml
${{ github.repository }}
```

might resolve to something conceptually like:

```text
username/my-project
```

Another example:

```yaml
${{ github.ref }}
```

might resolve to:

```text
refs/heads/main
```

Another:

```yaml
${{ github.actor }}
```

might resolve to the user or account that triggered the workflow.

So:

```text
${{ expression }}
        │
        ▼
GitHub evaluates expression
        │
        ▼
Actual value is inserted
```

---

# Understanding `${{ toJson(github) }}`

Let's break it down:

```yaml
${{ toJson(github) }}
```

## `github`

```yaml
github
```

This refers to the GitHub context.

It contains information about the current workflow execution.

Conceptually:

```text
github
│
├── actor
├── repository
├── event_name
├── ref
├── sha
├── workflow
├── run_id
└── additional event information
```

The exact information available can depend on the event and workflow environment.

---

## `toJson()`

```yaml
toJson(github)
```

converts the context object into JSON so it can be printed more easily.

Without converting the entire object, trying to inspect a complex context would be more difficult.

Conceptually:

```text
github context
      │
      ▼
toJson()
      │
      ▼
JSON representation
      │
      ▼
Print to workflow logs
```

Your command:

```yaml
run: echo "${{ toJson(github) }}"
```

means approximately:

```text
Take the github context
        ↓
Convert it to JSON
        ↓
Insert the JSON into the command
        ↓
Execute echo
        ↓
Display information in logs
```

---

# Example Information From the `github` Context

When the workflow runs, the GitHub context can provide information such as:

```json
{
  "event_name": "workflow_dispatch",
  "repository": "example-user/example-project",
  "workflow": "output info",
  "ref": "refs/heads/main"
}
```

The exact values will depend on the repository and the event that triggered the workflow.

Your workflow prints information related to the current execution.

---

# Why Is This Useful?

Contexts allow a workflow to make decisions based on what is happening.

For example:

```yaml
run: echo "Triggered by ${{ github.actor }}"
```

Conceptually:

```text
Workflow triggered by:
Kirtan
```

Or:

```yaml
run: echo "Repository: ${{ github.repository }}"
```

Output:

```text
Repository: username/project
```

Or:

```yaml
run: echo "Event: ${{ github.event_name }}"
```

Output:

```text
Event: push
```

or:

```text
Event: workflow_dispatch
```

depending on how the workflow was started.

---

# My Multiple Triggers Affect the Context

This is a useful connection between my two workflows.

I have:

```yaml
on: [push, workflow_dispatch]
```

The same workflow can be started by different events.

Therefore:

```yaml
${{ github.event_name }}
```

can tell me which event triggered the current run.

For example:

### When I push code

```text
git push
   ↓
github.event_name
   ↓
push
```

### When I manually run the workflow

```text
Run workflow button
   ↓
github.event_name
   ↓
workflow_dispatch
```

This means one workflow can change its behavior based on its trigger.

---

# Example: Different Behavior Based on the Trigger

I can use conditions with contexts.

For example:

```yaml
- name: Run only after a push
  if: github.event_name == 'push'
  run: echo "This workflow was triggered by a push"
```

And:

```yaml
- name: Run only manually
  if: github.event_name == 'workflow_dispatch'
  run: echo "This workflow was started manually"
```

Conceptually:

```text
Workflow starts
       │
       ▼
What triggered it?
       │
       ├── push
       │     └── Run push-specific step
       │
       └── workflow_dispatch
             └── Run manual-specific step
```

This is one reason contexts are powerful.

---

# Example: Deploy Only From the `main` Branch

Contexts can also be used for conditional deployment.

For example:

```yaml
- name: deploy
  if: github.ref == 'refs/heads/main'
  run: echo "Deploying production..."
```

Conceptually:

```text
Current branch?
      │
      ├── main
      │     │
      │     ▼
      │  Deploy ✅
      │
      └── feature branch
            │
            ▼
        Skip deployment ⏭️
```

This can prevent accidental production deployment from a feature branch.

---

# 4. Connecting `on`, `needs`, and Context

These three concepts work together.

## `on`

Determines:

```text
WHEN does the workflow start?
```

Example:

```yaml
on: [push, workflow_dispatch]
```

---

## Context

Provides:

```text
WHAT is happening during this workflow run?
```

Example:

```yaml
${{ github.event_name }}
```

This can tell the workflow whether it was started by:

```text
push
```

or:

```text
workflow_dispatch
```

---

## `needs`

Determines:

```text
WHAT must finish before another job can start?
```

Example:

```yaml
deploy:
  needs: test-job
```

Together:

```text
                 EVENT
                   │
                   ▼
            on: push/manual
                   │
                   ▼
               WORKFLOW
                   │
                   ▼
                test-job
                   │
            needs dependency
                   │
                   ▼
                 deploy
                   │
                   ▼
            Use GitHub context
                   │
                   ▼
          Make decisions/actions
```

---

# Example: A Safer Deployment Workflow

A more realistic workflow could look like this:

```yaml
name: CI and Deploy

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Run tests
        run: echo "Running tests..."

  security:
    runs-on: ubuntu-latest

    steps:
      - name: Run security scan
        run: echo "Running security scan..."

  deploy:
    needs: [test, security]
    runs-on: ubuntu-latest

    steps:
      - name: Show deployment information
        run: |
          echo "Repository: ${{ github.repository }}"
          echo "Triggered by: ${{ github.actor }}"
          echo "Event: ${{ github.event_name }}"

      - name: Deploy
        run: echo "Deploying..."
```

The execution flow is:

```text
                 Workflow Trigger
                        │
             ┌──────────┴──────────┐
             │                     │
           push              workflow_dispatch
             │                     │
             └──────────┬──────────┘
                        ▼
                 ┌─────────────┐
                 │    Test     │
                 └──────┬──────┘
                        │
                 ┌──────┴──────┐
                 │  Security   │
                 └──────┬──────┘
                        │
                 Both must pass
                        │
                        ▼
                     Deploy
                        │
                        ▼
              Read GitHub context
                        │
                        ▼
                  Deploy application
```

---

# 5. Important Mental Model

I should think about a workflow in three different layers.

## Layer 1: Trigger

```yaml
on:
```

Answers:

> **Why did this workflow start?**

Examples:

```text
push
pull_request
workflow_dispatch
schedule
```

---

## Layer 2: Job Flow

```yaml
needs:
```

Answers:

> **When is this job allowed to start?**

Example:

```text
test
  │
  ▼
deploy
```

---

## Layer 3: Runtime Information

```yaml
${{ github.* }}
```

Answers:

> **What information does GitHub know about this workflow run?**

Examples:

```text
Who triggered it?
Which repository?
Which branch?
Which event?
Which commit?
```

---

# Key Takeaways

```text
on
│
└── Starts the workflow based on events


on: [push, workflow_dispatch]
│
└── Workflow can run automatically or manually


needs
│
└── Creates job dependencies


needs: test-job
│
└── deploy waits for test-job


needs: [test, security, lint]
│
└── deploy waits for all required jobs


${{ }}
│
└── GitHub Actions expression syntax


github
│
└── Provides information about the current workflow run


toJson(github)
│
└── Converts the GitHub context into JSON for inspection
```

---

# Final Mental Model

```text
┌─────────────────────────────────────────────┐
│                  TRIGGER                    │
│                                             │
│   push  OR  workflow_dispatch              │
└──────────────────────┬──────────────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│                 WORKFLOW                    │
└──────────────────────┬──────────────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│                  TEST JOB                   │
│                                             │
│   Runs validation and tests                 │
└──────────────────────┬──────────────────────┘
                       │
                       │ needs
                       ▼
┌─────────────────────────────────────────────┐
│                 DEPLOY JOB                  │
│                                             │
│   Starts only if required jobs succeed      │
└──────────────────────┬──────────────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│               GITHUB CONTEXT                │
│                                             │
│   github.event_name                         │
│   github.actor                              │
│   github.repository                         │
│   github.ref                                │
└─────────────────────────────────────────────┘
```

The most important lesson is:

> **`on` controls when a workflow starts, `needs` controls the order and dependency between jobs, and contexts such as `github` provide information about the current workflow execution that can be used to inspect the run or control workflow behavior.**
