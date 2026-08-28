# Summary: GitHub Actions Events and Workflow Control

## 1. What Are Events in GitHub Actions?

An **event** is an activity that occurs in a GitHub repository and can trigger a GitHub Actions workflow.

The `on` keyword defines which events trigger a workflow.

Example:

```yaml
name: Event Demo

on: push
```

This means:

```text
Code is pushed
      ↓
push event occurs
      ↓
Workflow starts
```

Events are the starting point of a GitHub Actions workflow.

---

# 2. Multiple Events

A workflow can listen for more than one event.

Example:

```yaml
on: [push, workflow_dispatch]
```

This workflow can start in two ways:

```text
Code pushed ──────────────┐
                          │
                          ▼
                    Workflow starts
                          ▲
                          │
Manual trigger ───────────┘
```

Another way to configure multiple events is:

```yaml
on:
  push:
  workflow_dispatch:
```

The expanded syntax is especially useful when an event needs additional configuration such as activity types or filters.

---

# 3. `workflow_dispatch`

`workflow_dispatch` allows a workflow to be started manually.

Example:

```yaml
on:
  workflow_dispatch:
```

The flow is:

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

This is useful for:

* Testing workflows
* Debugging
* Troubleshooting
* Manual deployments
* Running a workflow without creating a new push or pull request

---

# 4. Event Activity Types

Some events have different **activity types**.

For example, the `pull_request` event can have activities such as:

```text
opened
closed
reopened
synchronize
edited
labeled
unlabeled
```

I can filter the event to listen only for specific activities.

Example:

```yaml
on:
  pull_request:
    types:
      - opened
```

This means:

```text
Pull request opened
       ↓
Workflow runs ✅
```

But:

```text
Pull request closed
       ↓
Workflow does not run ❌
```

Activity types provide more precise control over workflow triggers.

---

# 5. Multiple Activity Types

I can configure multiple activity types.

Example:

```yaml
on:
  pull_request:
    types:
      - opened
      - reopened
      - synchronize
```

The workflow can now run when:

```text
PR opened
    ↓
Run workflow ✅

PR reopened
    ↓
Run workflow ✅

New commits pushed to an existing PR
    ↓
synchronize
    ↓
Run workflow ✅
```

This is commonly useful for CI pipelines.

---

# 6. Branch Filters

Branch filters control which branches can trigger a workflow.

Example:

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


Push to develop
     ↓
Workflow does not run ❌
```

Multiple branches can also be configured:

```yaml
branches:
  - main
  - develop
```

---

# 7. Branch Patterns

GitHub Actions supports patterns for matching branch names.

Example:

```yaml
branches:
  - main
  - 'dev-*'
  - 'feat/**'
```

### Exact branch match

```yaml
- main
```

Matches:

```text
main ✅
```

### Wildcard pattern

```yaml
- 'dev-*'
```

Can match branches such as:

```text
dev-test
dev-staging
dev-kirtan
```

### Recursive pattern

```yaml
- 'feat/**'
```

Can match branches such as:

```text
feat/login
feat/payment
feat/api/v2
feat/user/profile
```

This allows teams to create branch naming conventions and trigger workflows based on those patterns.

---

# 8. Important Difference: `push` vs `pull_request` Branch Filters

The `branches` filter means something different depending on the event.

## With `push`

```yaml
on:
  push:
    branches:
      - main
```

This means:

> Run when code is pushed **to `main`**.

```text
git push → main
      ↓
Workflow runs
```

## With `pull_request`

```yaml
on:
  pull_request:
    branches:
      - main
```

This means:

> Run when a pull request is **targeting `main`**.

```text
feature/login
      │
      │ Pull Request
      ▼
     main
      ↓
Workflow runs
```

The mental model is:

```text
push:
branches = Where was code pushed?


pull_request:
branches = Which branch is the PR targeting?
```

---

# 9. Combining Activity Types and Branch Filters

Filters can be combined.

Example:

```yaml
on:
  pull_request:
    types:
      - opened
    branches:
      - main
```

Both conditions must match.

```text
Pull Request
     ↓
Is activity "opened"?
     ↓
Is target branch "main"?
     ↓
Workflow runs
```

Examples:

```text
PR opened → main
Workflow runs ✅


PR opened → develop
Workflow does not run ❌


PR closed → main
Workflow does not run ❌
```

This gives precise control over when workflows run.

---

# 10. Path Filters

Path filters control workflow execution based on which files are changed.

Example:

```yaml
on:
  push:
    paths:
      - 'src/**'
```

This means:

```text
src/app.js changed
       ↓
Workflow runs ✅


README.md changed
       ↓
Workflow does not run ❌
```

Multiple paths can be configured:

```yaml
paths:
  - 'src/**'
  - 'package.json'
  - 'package-lock.json'
```

The workflow can run when relevant application or dependency files change.

---

# 11. `paths-ignore`

`paths-ignore` specifies changes that should not trigger the workflow.

Example:

```yaml
on:
  push:
    paths-ignore:
      - 'docs/**'
      - 'README.md'
```

Now:

```text
Only README changed
      ↓
Workflow does not run


Only documentation changed
      ↓
Workflow does not run


Application code changed
      ↓
Workflow can run
```

The difference is:

```text
paths
│
└── What changes DO I care about?


paths-ignore
│
└── What changes do I NOT care about?
```

---

# 12. Combining Branch and Path Filters

Filters work together to provide more control.

Example:

```yaml
on:
  push:
    branches:
      - main
    paths:
      - 'src/**'
```

The workflow requires the configured conditions to match.

```text
Push event
    ↓
Was it pushed to main?
    ↓
Did relevant source files change?
    ↓
Workflow runs
```

Examples:

```text
Push to main + src/app.js changed
→ Workflow runs ✅


Push to main + README.md changed
→ Workflow does not run ❌


Push to feature branch + src/app.js changed
→ Workflow does not run ❌
```

---

# 13. Event Data and `github.event`

GitHub provides information about the event that triggered the workflow.

Example:

```yaml
- name: Output event data
  run: echo "${{ toJSON(github.event) }}"
```

The process is:

```text
GitHub event occurs
       ↓
GitHub creates event payload
       ↓
github.event
       ↓
toJSON()
       ↓
Information displayed in workflow logs
```

The event data changes depending on the trigger.

```text
push
  ↓
Push-related event data


pull_request
  ↓
Pull-request-related event data


workflow_dispatch
  ↓
Manual-trigger-related event data
```

This is useful for debugging and learning what information GitHub provides.

---

# 14. Workflow Trigger Flow

A more complex event configuration can look like:

```yaml
on:
  pull_request:
    types:
      - opened
    branches:
      - main
      - 'dev-*'
      - 'feat/**'

  workflow_dispatch:

  push:
    branches:
      - main
      - 'dev-*'
      - 'feat/**'
```

Conceptually:

```text
                    GitHub Activity
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
   Pull Request         Push         Manual Run
          │               │               │
          ▼               ▼               │
   Activity filter   Branch filter        │
          │               │               │
          ▼               ▼               │
   Target branch?    Path filters?        │
          │               │               │
          └───────────────┼───────────────┘
                          │
                          ▼
                    Workflow Starts
```

---

# 15. What Happens When a Step Fails?

Steps normally run sequentially.

Example:

```text
Get code
   ↓
Install dependencies
   ↓
Run tests
   ↓
Build
   ↓
Deploy
```

If a step fails:

```text
Get code               ✅
Install dependencies   ✅
Run tests              ❌
Build                  ⏭️
Deploy                 ⏭️
```

By default:

```text
Step fails
    ↓
Job fails
```

Most commands communicate success or failure using an exit code:

```text
Exit code 0
    ↓
Success ✅


Non-zero exit code
    ↓
Failure ❌
```

For example:

```yaml
- name: Run tests
  run: npm test
```

If the tests fail, the step normally fails.

---

# 16. Job Failure and Workflow Failure

With a simple workflow:

```text
Workflow
    │
    ▼
Job
    │
    ▼
Step fails ❌
```

The result is usually:

```text
Step fails
    ↓
Job fails
    ↓
Workflow fails
```

However, independent jobs may continue running even if another parallel job fails.

---

# 17. `needs` and Failure

The `needs` keyword creates job dependencies.

Example:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

  deploy:
    needs: test
    runs-on: ubuntu-latest
```

The dependency flow is:

```text
test
  ↓
Must succeed
  ↓
deploy
```

If tests pass:

```text
test ✅
   ↓
deploy runs ✅
```

If tests fail:

```text
test ❌
   ↓
deploy is skipped ⏭️
```

This is important in CI/CD because I usually do not want to deploy an application that has failed its tests.

---

# 18. Manually Cancelling a Workflow

A workflow can be cancelled while it is running.

```text
Workflow Running
       ↓
Cancel Workflow
       ↓
Workflow Cancelled ⛔
```

This can be useful when:

* The workflow was triggered accidentally.
* The wrong code was pushed.
* A deployment should be stopped.
* The workflow is taking too long.
* The current workflow run is no longer needed.

Cancellation is different from failure.

```text
Failure
│
└── Something went wrong.


Cancellation
│
└── The workflow was intentionally stopped.
```

---

# 19. Automatically Cancelling Older Runs

GitHub Actions can automatically cancel outdated workflow runs using concurrency.

Example:

```yaml
concurrency:
  group: production-deployment
  cancel-in-progress: true
```

The concept is:

```text
Commit A
   ↓
Workflow #1 starts


Commit B pushed before Workflow #1 finishes
   ↓
Workflow #1 cancelled ⛔
   ↓
Workflow #2 runs with the latest code ✅
```

This prevents resources from being wasted on outdated commits.

---

# 20. Skipping a Workflow

A workflow can be skipped even when an event would normally trigger it.

For example:

```yaml
on: push
```

Normally:

```text
git push
   ↓
Workflow runs
```

But I can add a supported skip annotation to a commit message:

```bash
git commit -m "Update README [skip ci]"
```

Conceptually:

```text
Push event occurs
       ↓
GitHub checks commit message
       ↓
Skip annotation detected
       ↓
Workflow is skipped
```

Common supported annotations include:

```text
[skip ci]
[ci skip]
[no ci]
[skip actions]
[actions skip]
```

Example:

```bash
git commit -m "Fix documentation typo [skip ci]"
```

---

# 21. When Should I Skip a Workflow?

Skipping may be useful for changes such as:

```text
README typo
Documentation formatting
Non-functional comments
Minor markdown changes
```

For example:

```bash
git commit -m "Fix markdown formatting [no ci]"
```

This is a one-time decision for that commit.

---

# 22. Skip Annotation vs `paths-ignore`

These are different approaches.

## Skip Annotation

```bash
git commit -m "Fix typo [skip ci]"
```

This means:

```text
Skip this particular workflow trigger/run
```

It is a manual, temporary decision.

---

## `paths-ignore`

```yaml
paths-ignore:
  - 'docs/**'
  - 'README.md'
```

This means:

```text
Always ignore these types of changes
```

It is a permanent rule in the workflow configuration.

The mental model is:

```text
Skip annotation
    ↓
One-time decision


paths-ignore
    ↓
Permanent workflow rule
```

---

# 23. Skipping a Workflow, Job, or Step

Skipping can happen at different levels.

## Entire Workflow

A workflow is prevented from running.

Example:

```text
[skip ci]
```

---

## Job

A job can be skipped using a condition.

Example:

```yaml
deploy:
  if: github.ref == 'refs/heads/main'
```

Conceptually:

```text
Workflow starts
      ↓
Is branch main?
      │
   ┌──┴──┐
   │     │
  No    Yes
   │     │
Skip    Run
```

---

## Step

An individual step can also be skipped.

Example:

```yaml
- name: Deploy
  if: github.ref == 'refs/heads/main'
  run: echo "Deploying..."
```

The hierarchy is:

```text
Workflow
│
├── Job
│   │
│   ├── Step
│   ├── Step
│   └── Step
│
└── Job
```

A condition can control execution at different levels.

---

# 24. Failure vs Cancellation vs Skipping

| Action             | Meaning                                         |
| ------------------ | ----------------------------------------------- |
| Step fails         | A command or action encounters an error         |
| Job fails          | A required step fails                           |
| Workflow fails     | One or more required jobs fail                  |
| Job skipped        | Job condition or dependency prevents execution  |
| Step skipped       | Step condition prevents execution               |
| Workflow skipped   | Trigger is intentionally prevented from running |
| Workflow cancelled | A running workflow is intentionally stopped     |

---

# Final Mental Model

Everything learned so far can be understood as a workflow lifecycle:

```text
GitHub Activity
       │
       ▼
Does it match the configured event?
       │
       ▼
Do activity type filters match?
       │
       ▼
Do branch filters match?
       │
       ▼
Do path filters match?
       │
       ▼
Is there a skip instruction?
       │
       ├── Yes → Workflow skipped ⏭️
       │
       └── No
            │
            ▼
       Workflow starts
            │
            ▼
        Jobs execute
            │
            ├── Condition false → Job skipped ⏭️
            ├── Dependency fails → Dependent job skipped ⏭️
            ├── Job fails → Workflow may fail ❌
            └── Job succeeds
                    │
                    ▼
               Steps execute
                    │
                    ├── Condition false → Step skipped ⏭️
                    ├── Step fails → Job fails ❌
                    ├── Workflow cancelled → Work stops ⛔
                    └── All required steps succeed
                            │
                            ▼
                       Success ✅
```

## Key Takeaway

> **Events determine what can start a workflow. Activity types, branch filters, and path filters control when it starts. Skip rules can prevent unnecessary runs. Conditions and dependencies control which jobs and steps execute. Failures stop dependent work, while cancellation intentionally stops work that is already running.**

At this point, I understand the full basic lifecycle of a GitHub Actions workflow: **what triggers it, how triggers are filtered, how execution can be skipped, how failures affect jobs and workflows, and how running workflows can be cancelled.**
