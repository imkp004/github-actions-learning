# Cancelling, Failing, and Skipping GitHub Actions Workflows

GitHub Actions gives me several ways to stop work from continuing:

```text
Workflow starts
      │
      ├── Step fails
      │      ↓
      │   Job fails
      │      ↓
      │ Workflow may fail
      │
      ├── Workflow is manually cancelled
      │      ↓
      │ Workflow stops
      │
      └── Workflow is skipped before running
             ↓
          No workflow run starts
```

Although these concepts are related, they are different:

* **Failure** → Something went wrong while the workflow was running.
* **Cancellation** → A running workflow is intentionally stopped.
* **Skipping** → A workflow that would normally trigger is prevented from running.

Understanding the difference is important when building CI/CD pipelines.

---

# 1. What Happens When a Step Fails?

A job contains one or more steps:

```text
Job
│
├── Step 1: Get code
├── Step 2: Install dependencies
├── Step 3: Run tests
├── Step 4: Build application
└── Step 5: Deploy
```

By default, steps run sequentially.

For example:

```yaml
steps:
  - name: Get code
    uses: actions/checkout@v3

  - name: Install dependencies
    run: npm ci

  - name: Run tests
    run: npm test

  - name: Build project
    run: npm run build

  - name: Deploy
    run: echo "Deploying..."
```

The normal execution flow is:

```text
Get code
   ↓
Success ✅
   ↓
Install dependencies
   ↓
Success ✅
   ↓
Run tests
   ↓
Success ✅
   ↓
Build project
   ↓
Success ✅
   ↓
Deploy
```

However, what happens if `npm test` fails?

```text
Get code
   ↓
Success ✅
   ↓
Install dependencies
   ↓
Success ✅
   ↓
Run tests
   ↓
FAILURE ❌
```

By default, GitHub Actions stops executing the remaining steps in that job.

```text
Build project
    ↓
Skipped ⏭️

Deploy
    ↓
Skipped ⏭️
```

The job is then marked as failed.

```text
Job Status: Failure ❌
```

---

# 2. Step Failure → Job Failure

The basic default behavior is:

```text
Step fails
    ↓
Job fails
```

For example:

```yaml
- name: Run tests
  run: npm test
```

If `npm test` returns a non-zero exit code, the step fails.

Conceptually:

```text
npm test
   │
   ├── Exit code 0
   │       ↓
   │    Success ✅
   │
   └── Non-zero exit code
           ↓
        Failure ❌
```

Most command-line tools use exit codes to communicate success or failure.

Generally:

```text
0       = Success
Non-zero = Failure
```

For example:

```bash
npm test
```

If all tests pass:

```text
Exit code: 0
```

If one or more tests fail:

```text
Exit code: 1
```

GitHub Actions detects the failed exit code and marks the step as failed.

---

# 3. Job Failure → Workflow Failure

Consider a simple workflow with one job:

```text
Workflow
   │
   ▼
test job
```

If the only job fails:

```text
Workflow
   │
   ▼
test job ❌
   │
   ▼
Workflow ❌
```

So:

```text
Step fails
    ↓
Job fails
    ↓
Workflow fails
```

Example:

```yaml
name: Test Application

on: push

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - run: npm ci

      - run: npm test
```

If:

```text
npm test ❌
```

then:

```text
test job ❌
```

and the workflow run is normally marked:

```text
Failure ❌
```

---

# 4. Multiple Jobs and Failure

The behavior becomes more interesting when a workflow has multiple jobs.

Example:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
```

Because there is no `needs` relationship:

```text
test job     build job
    │             │
    ▼             ▼
Runs           Runs
```

The jobs can run independently and, when capacity is available, in parallel.

If the test job fails:

```text
test ❌       build
               │
               ▼
             Success ✅
```

The workflow is still considered unsuccessful overall because one job failed:

```text
Workflow ❌
```

However, the `build` job does not automatically stop just because the independent `test` job failed.

This is different from dependent jobs.

---

# 5. Failure with `needs`

Consider:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - run: echo "Deploying..."
```

The execution flow is:

```text
test
  │
  ▼
Must succeed
  │
  ▼
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

This is extremely important in CI/CD.

For example:

```text
Run Tests
    ↓
Tests fail ❌
    ↓
DO NOT deploy
```

The `needs` keyword creates a dependency between jobs.

---

# 6. Failure Does Not Always Mean the Entire Workflow Immediately Stops

This is an important distinction.

Suppose I have:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

  security:
    runs-on: ubuntu-latest

  deploy:
    needs: [test, security]
    runs-on: ubuntu-latest
```

The workflow looks like:

```text
              ┌── test ──────┐
              │              │
Workflow ─────┤              ├──► deploy
              │              │
              └── security ──┘
```

If:

```text
test ❌
security ✅
```

then:

```text
deploy ⏭️
```

because `deploy` depends on both jobs.

The workflow ultimately fails because one of the required jobs failed.

---

# 7. Manually Cancelling a Workflow

A workflow can also be manually cancelled while it is running.

For example:

```text
Workflow Running
       │
       ▼
GitHub Actions Page
       │
       ▼
Cancel Workflow
       │
       ▼
Workflow stops
```

Conceptually:

```text
Running
   │
   ▼
Manual cancellation
   │
   ▼
Cancelled ⛔
```

This is useful when:

* I accidentally triggered the workflow.
* I pushed incorrect code.
* A deployment should no longer continue.
* The workflow is taking too long.
* I started the wrong workflow.
* I want to stop unnecessary CI/CD resource usage.

---

# Example: Cancelling During a Long-Running Job

Suppose my workflow is:

```text
Install dependencies
        ↓
Run tests
        ↓
Run security scan
        ↓
Build project
        ↓
Deploy
```

The security scan is taking a long time:

```text
Run security scan...
Status: Running
```

I realize the workflow was triggered by mistake.

I manually cancel it:

```text
Workflow Running
       ↓
Cancel
       ↓
Workflow Cancelled ⛔
```

The remaining work does not continue normally.

For example:

```text
Install dependencies ✅
Run tests             ✅
Security scan         ⛔ Cancelled
Build                 ⏭️ Not completed
Deploy                ⏭️ Not completed
```

---

# 8. Automatic Cancellation Using Concurrency

GitHub Actions can also cancel older workflow runs automatically.

This is useful when multiple commits are pushed quickly.

For example:

```text
Commit 1
   ↓
Workflow #1 starts
   ↓
Still running...

Commit 2
   ↓
Workflow #2 starts
```

Now I may not need Workflow #1 anymore because it is testing older code.

I can use concurrency:

```yaml
concurrency:
  group: production-deployment
  cancel-in-progress: true
```

The concept is:

```text
Workflow #1 Running
        │
        ▼
New Workflow #2 starts
        │
        ▼
Workflow #1 cancelled ⛔
        │
        ▼
Workflow #2 continues with newest code ✅
```

This is useful because:

```text
Older commit
     ↓
Old CI run
     ↓
No longer needed

Newer commit
     ↓
New CI run
     ↓
This is the one I care about
```

A common use case is:

```text
Developer pushes commit A
        ↓
CI starts

Developer quickly pushes commit B
        ↓
Cancel CI for commit A
        ↓
Run CI for commit B
```

This prevents wasting resources on outdated commits.

---

# 9. What Does Skipping a Workflow Mean?

Skipping is different from failure and cancellation.

A skipped workflow is prevented from running even though the event would normally trigger it.

For example:

```yaml
on: push
```

Normally:

```text
git push
   ↓
push event
   ↓
Workflow runs
```

But suppose I only changed a comment in the documentation and do not need CI to run.

I can use a special annotation in the commit message.

For example:

```bash
git commit -m "Fix documentation typo [skip ci]"
```

Then:

```text
git push
   ↓
push event occurs
   ↓
GitHub checks commit message
   ↓
[skip ci] detected
   ↓
Workflow is skipped
```

---

# 10. Supported Skip Annotations

GitHub recognizes special phrases in commit messages or pull request commit messages.

Common supported annotations include:

```text
[skip ci]
[ci skip]
[no ci]
[skip actions]
[actions skip]
```

For example:

```bash
git commit -m "Update README [skip ci]"
```

or:

```bash
git commit -m "Fix typo [ci skip]"
```

Conceptually:

```text
Commit Message
      │
      ▼
"Update documentation [skip ci]"
      │
      ▼
GitHub detects skip annotation
      │
      ▼
Workflow does not run
```

> `[skip action]` is not the standard GitHub Actions skip annotation. Use a recognized annotation such as `[skip ci]` or `[skip actions]`.

---

# 11. Why Would I Skip a Workflow?

Suppose my workflow runs tests every time I push:

```yaml
on: push
```

I make a change like:

```text
Fix spelling in README
```

The change does not affect:

```text
Application code
Dependencies
Tests
Build process
```

I may decide that running the full CI pipeline is unnecessary.

Instead of:

```bash
git commit -m "Fix README typo"
```

I could use:

```bash
git commit -m "Fix README typo [skip ci]"
```

The result is:

```text
Push
  │
  ▼
Workflow would normally trigger
  │
  ▼
Skip annotation found
  │
  ▼
Workflow skipped
```

This can save unnecessary workflow runs.

---

# 12. More Skip Annotation Examples

### Example 1: Documentation Change

```bash
git commit -m "Update project documentation [skip ci]"
```

Meaning:

```text
Documentation updated
        ↓
CI intentionally skipped
```

---

### Example 2: Typo Fix

```bash
git commit -m "Fix typo in README [ci skip]"
```

Meaning:

```text
Minor documentation change
        ↓
No need to run full pipeline
```

---

### Example 3: Add Comments

```bash
git commit -m "Add comments to Terraform files [skip actions]"
```

Conceptually:

```text
Non-functional change
       ↓
Skip GitHub Actions
```

---

### Example 4: Markdown Formatting

```bash
git commit -m "Fix markdown formatting [no ci]"
```

This may be appropriate when the change does not require application testing.

---

# 13. Skip Annotation vs Path Filtering

Skip annotations are useful for an individual commit.

For example:

```bash
git commit -m "Fix typo [skip ci]"
```

This is a **manual decision for one commit**.

Path filtering is different.

Example:

```yaml
on:
  push:
    paths-ignore:
      - 'docs/**'
      - 'README.md'
```

This creates a permanent workflow rule.

```text
Documentation change
        ↓
Workflow automatically does not run
```

The difference is:

```text
Skip Annotation
│
└── Temporary decision made in a commit message


Path Filter
│
└── Permanent workflow configuration rule
```

---

# Example Comparison

Suppose I have this repository:

```text
project/
├── src/
├── tests/
├── docs/
└── README.md
```

## Option 1: Skip Annotation

My workflow triggers on every push:

```yaml
on: push
```

I make a documentation change:

```text
docs/setup.md
```

I can manually skip CI:

```bash
git commit -m "Update setup guide [skip ci]"
```

The next commit does not automatically skip unless I add the annotation again.

---

## Option 2: `paths-ignore`

```yaml
on:
  push:
    paths-ignore:
      - 'docs/**'
      - 'README.md'
```

Now:

```text
docs/setup.md changed
       ↓
Workflow automatically skipped
```

No special commit message is needed.

---

# 14. Skipping a Job vs Skipping an Entire Workflow

These are also different concepts.

A job can be conditionally skipped.

Example:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - run: npm test

  deploy:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

    steps:
      - run: echo "Deploying..."
```

Conceptually:

```text
Workflow starts
      │
      ├── test
      │     ↓
      │   Runs
      │
      └── deploy
            ↓
      Is branch main?
          │
       ┌──┴───┐
       │      │
      No     Yes
       │      │
    Skipped   Runs
```

If the workflow runs on a feature branch:

```text
test   → Runs ✅
deploy → Skipped ⏭️
```

The workflow itself still ran.

This is different from a skip annotation, where the workflow run is prevented from proceeding normally.

---

# 15. Skipping a Step

I can also conditionally skip an individual step.

Example:

```yaml
steps:
  - name: Run tests
    run: npm test

  - name: Deploy
    if: github.ref == 'refs/heads/main'
    run: echo "Deploying..."
```

On a feature branch:

```text
Run tests
   ↓
Success ✅

Deploy
   ↓
Condition is false
   ↓
Skipped ⏭️
```

The workflow and job continue, but that specific step does not run.

This creates three different levels of skipping:

```text
Workflow Level
    │
    └── Entire workflow is skipped


Job Level
    │
    └── Workflow runs, but a job is skipped


Step Level
    │
    └── Job runs, but a specific step is skipped
```

---

# 16. Failure vs Cancellation vs Skipping

This is the most important comparison.

| Situation              | What Happens?                      | Example               |
| ---------------------- | ---------------------------------- | --------------------- |
| Step failure           | Step returns an error              | `npm test` fails      |
| Job failure            | Job is marked failed               | A required step fails |
| Workflow failure       | Workflow has failed jobs           | Tests fail            |
| Manual cancellation    | Running workflow is stopped        | User clicks Cancel    |
| Automatic cancellation | Older run is stopped               | Newer run replaces it |
| Workflow skip          | Workflow is intentionally not run  | `[skip ci]`           |
| Job skip               | One job does not meet a condition  | `if:` is false        |
| Step skip              | One step does not meet a condition | `if:` is false        |

---

# 17. Complete Mental Model

I can think about the lifecycle like this:

```text
GitHub Event
     │
     ▼
Should workflow trigger?
     │
     ├── Skip annotation found
     │        │
     │        ▼
     │   Workflow skipped ⏭️
     │
     └── No skip annotation
              │
              ▼
        Workflow starts
              │
              ▼
           Jobs start
              │
              ├── Job condition false
              │       │
              │       ▼
              │    Job skipped ⏭️
              │
              └── Job runs
                      │
                      ▼
                    Steps
                      │
                      ├── Step condition false
                      │       │
                      │       ▼
                      │    Step skipped ⏭️
                      │
                      ├── Step fails
                      │       │
                      │       ▼
                      │    Job fails ❌
                      │
                      ├── User cancels
                      │       │
                      │       ▼
                      │   Workflow cancelled ⛔
                      │
                      └── All required steps succeed
                              │
                              ▼
                        Workflow succeeds ✅
```

---

# Key Takeaways

> **A failed step normally causes the current job to fail. If the failed job is required for the workflow, the workflow is marked as failed, and jobs that depend on it through `needs` are normally skipped.**

> **A workflow can be cancelled manually while it is running, or older workflow runs can be automatically cancelled using concurrency settings.**

> **Skipping is different from cancelling. Skipping prevents a workflow, job, or step from running, while cancellation stops work that has already started or is in progress.**

> **For one-time workflow skipping, GitHub recognizes annotations such as `[skip ci]`, `[ci skip]`, `[no ci]`, `[skip actions]`, and `[actions skip]`.**

My final mental model is:

```text
Before execution:
Skip or Trigger?

During execution:
Run, Fail, or Cancel?

Inside the workflow:
Run or Skip specific Jobs/Steps?
```

Understanding these controls is important because real CI/CD pipelines should not only know **how to start**, but also know **when to stop, what to skip, and how to prevent unnecessary or unsafe work from continuing**.
