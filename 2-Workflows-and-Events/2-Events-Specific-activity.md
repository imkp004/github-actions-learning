# GitHub Actions: Events on Specific Branches, Activity Types, and Filters

So far, I learned that the `on` keyword determines which events can trigger a GitHub Actions workflow.

For example:

```yaml
on: push
```

This means the workflow listens for every relevant `push` event.

However, in real CI/CD pipelines, I often need **more control**.

I may want a workflow to run only:

* When code is pushed to a specific branch.
* When a pull request targets a specific branch.
* When a specific activity happens, such as a pull request being opened.
* When specific files or folders are changed.
* When an event matches multiple conditions.

This is where **activity types** and **event filters** become important.

---

# My Workflow

I created the following workflow:

```yaml
name: Events Demo 1

on: #[push, workflow_dispatch, ]

  pull_request:

    types:

      - opened

  workflow_dispatch:

jobs:

  deploy:

    runs-on: ubuntu-latest

    steps:

      - name: Output event data

        run: echo "${{ toJSON(github.event) }}"

      - name: Get code

        uses: actions/checkout@v3

      - name: Install dependencies

        run: npm ci

      - name: Test code

        run: npm run test

      - name: Build code

        run: npm run build

      - name: Deploy project

        run: echo "Deploying..."
```

The important part of this workflow is the trigger configuration:

```yaml
on:
  pull_request:
    types:
      - opened

  workflow_dispatch:
```

This means the workflow can start in two ways:

```text
Pull Request Opened
        │
        ▼
   Workflow Starts
        ▲
        │
        │
Manual Trigger
workflow_dispatch
```

---

# 1. Event Activity Types

Some GitHub events can have different **activity types**.

For example, a pull request can be:

```text
opened
closed
reopened
synchronized
edited
labeled
unlabeled
```

These are different activities related to the same `pull_request` event.

Without filtering, a pull request workflow may respond to its default activity behavior.

With activity types, I can tell GitHub exactly which activity I want to trigger my workflow.

My example:

```yaml
pull_request:
  types:
    - opened
```

This means:

> Only trigger this workflow when a pull request is opened.

---

# How My `pull_request` Trigger Works

Suppose I have the following branch:

```text
feature/login
```

I create a pull request:

```text
feature/login
      │
      │ Pull Request
      ▼
     main
```

When I click **Create Pull Request**, GitHub generates:

```text
pull_request event
      │
      ▼
Activity type = opened
      │
      ▼
Matches workflow configuration?
      │
      ▼
YES
      │
      ▼
Workflow runs
```

My workflow starts because:

```yaml
types:
  - opened
```

matches the activity.

---

# What Happens When Another Pull Request Activity Occurs?

My workflow specifically listens for:

```yaml
types:
  - opened
```

Therefore, conceptually:

```text
Pull Request Opened
        │
        ▼
Workflow Runs ✅


Pull Request Closed
        │
        ▼
Workflow Does Not Run ❌


Pull Request Labeled
        │
        ▼
Workflow Does Not Run ❌


Pull Request Reopened
        │
        ▼
Workflow Does Not Run ❌
```

The important idea is:

> **The event identifies the general GitHub activity, while the activity type gives more specific control over which activity should trigger the workflow.**

---

# Multiple Activity Types

I can listen for multiple activities.

For example:

```yaml
on:
  pull_request:
    types:
      - opened
      - reopened
      - synchronize
```

Now the workflow can run when:

```text
Pull Request Opened
        │
        ├──► Run workflow ✅
        │
Pull Request Reopened
        │
        ├──► Run workflow ✅
        │
New commits pushed to the PR
        │
        └──► synchronize
              │
              └──► Run workflow ✅
```

This is useful for a CI workflow because I may want tests to run when:

1. A pull request is first opened.
2. A previously closed pull request is reopened.
3. New commits are pushed to the branch associated with the pull request.

---

# Understanding `synchronize`

One important activity type is:

```yaml
synchronize
```

Suppose I create this pull request:

```text
feature/login
      │
      ▼
Pull Request → main
```

The workflow runs when the pull request is opened.

Later, I make more changes:

```text
git add .
git commit -m "Fix login bug"
git push
```

GitHub updates the existing pull request.

This can generate the `synchronize` activity.

Conceptually:

```text
Pull Request Already Exists
          │
          ▼
New Commit Pushed
          │
          ▼
Pull Request Updated
          │
          ▼
synchronize activity
          │
          ▼
Workflow Runs
```

This is why CI workflows commonly need to consider more than just the `opened` activity.

---

# 2. Events Occurring on Specific Branches

GitHub Actions can also filter events based on branches.

For example:

```yaml
on:
  push:
    branches:
      - main
```

This means:

> Only run the workflow when code is pushed to the `main` branch.

The behavior is:

```text
Push to main
     │
     ▼
Workflow Runs ✅


Push to feature/login
     │
     ▼
Workflow Does Not Run ❌


Push to develop
     │
     ▼
Workflow Does Not Run ❌
```

This gives more control over workflow execution.

---

# Example: Multiple Branches

I can specify multiple branches:

```yaml
on:
  push:
    branches:
      - main
      - develop
```

Now:

```text
Push to main
    │
    └──► Workflow Runs ✅


Push to develop
    │
    └──► Workflow Runs ✅


Push to feature/login
    │
    └──► Workflow Does Not Run ❌
```

This is useful when different branches represent important environments.

For example:

```text
main
  │
  └── Production


develop
  │
  └── Development


feature/*
  │
  └── Individual feature development
```

---

# 3. Pull Request Branch Filters

Branch filtering works slightly differently with `pull_request`.

For example:

```yaml
on:
  pull_request:
    branches:
      - main
```

This means:

> Run the workflow when a pull request targets the `main` branch.

Example:

```text
feature/login
      │
      │ Pull Request
      ▼
     main
      │
      ▼
Workflow Runs ✅
```

Another example:

```text
feature/login
      │
      │ Pull Request
      ▼
   develop
      │
      ▼
Workflow Does Not Run ❌
```

This is important:

> With `pull_request`, the `branches` filter refers to the branch the pull request is targeting, not necessarily the branch where the developer made changes.

---

# Combining Activity Types and Branch Filters

I can combine both controls.

For example:

```yaml
on:
  pull_request:
    types:
      - opened
    branches:
      - main
```

Now the workflow only runs when **both conditions are true**.

```text
Condition 1:
Pull Request activity = opened

AND

Condition 2:
Pull Request target branch = main
```

The flow becomes:

```text
Pull Request Activity
        │
        ▼
Is activity "opened"?
        │
     ┌──┴──┐
     │     │
    No    Yes
     │     │
   Stop    ▼
       Is target main?
            │
         ┌──┴──┐
         │     │
        No    Yes
         │     │
       Stop   Run
```

Example:

```yaml
on:
  pull_request:
    types:
      - opened
    branches:
      - main
```

Behavior:

```text
PR opened → main
Workflow runs ✅


PR opened → develop
Workflow does not run ❌


PR closed → main
Workflow does not run ❌
```

This provides very precise control.

---

# 4. Understanding My Current Workflow

My trigger configuration is:

```yaml
on:
  pull_request:
    types:
      - opened

  workflow_dispatch:
```

This means:

```text
                    Events Demo 1
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
      Pull Request Opened      Manual Trigger
              │                       │
              └───────────┬───────────┘
                          │
                          ▼
                      deploy job
```

The workflow will run if:

### Scenario 1: A pull request is opened

```text
Developer creates feature branch
          │
          ▼
Developer writes code
          │
          ▼
Developer creates Pull Request
          │
          ▼
Activity = opened
          │
          ▼
Workflow starts automatically
```

### Scenario 2: The workflow is manually triggered

```text
GitHub Actions Tab
        │
        ▼
Run Workflow
        │
        ▼
workflow_dispatch event
        │
        ▼
Workflow starts
```

---

# 5. Why Use `workflow_dispatch` Along With Filters?

This is a useful combination.

Suppose my workflow only listens for:

```yaml
pull_request:
  types:
    - opened
```

Once a pull request is already open, I may want to test the workflow again without opening another pull request.

I can use:

```yaml
workflow_dispatch:
```

This gives me a manual option.

```text
Automatic:
PR opened
    ↓
Workflow runs


Manual:
Click Run Workflow
    ↓
Workflow runs
```

This is useful for:

```text
Testing
Troubleshooting
Debugging
Manual deployments
Re-running a process
```

---

# 6. Outputting Event Data

My first step is:

```yaml
- name: Output event data
  run: echo "${{ toJSON(github.event) }}"
```

This is a powerful debugging and learning technique.

To understand it:

```text
github.event
```

contains information about the specific event that triggered the workflow.

For example:

```text
Pull Request Opened
       │
       ▼
GitHub creates event data
       │
       ▼
github.event
       │
       ▼
Workflow can access that data
```

The event data can contain details related to the trigger.

For a pull request, conceptually, this can include information such as:

```text
Pull request number
Pull request title
Pull request state
Source branch
Target branch
Repository information
User who created the pull request
```

The exact data depends on the event.

---

# `github` vs `github.event`

It is important to understand the difference.

## `github`

```yaml
${{ toJSON(github) }}
```

The `github` context provides broader information about the workflow execution.

Conceptually:

```text
github
│
├── actor
├── repository
├── event_name
├── ref
├── workflow
├── run_id
└── event
```

---

## `github.event`

```yaml
${{ toJSON(github.event) }}
```

This focuses specifically on the event payload that triggered the workflow.

Conceptually:

```text
github.event
│
└── Details about the triggering event
```

For my workflow:

```text
Pull Request Opened
       │
       ▼
github.event
       │
       ├── Pull request information
       ├── Repository information
       ├── Sender information
       └── Activity information
```

Therefore, my workflow is printing the details of the actual triggering event.

---

# 7. Why `toJSON()` Is Used

The event data is a structured object.

It is easier to inspect when converted to JSON.

My step:

```yaml
run: echo "${{ toJSON(github.event) }}"
```

works conceptually like:

```text
github.event
     │
     ▼
Structured event object
     │
     ▼
toJSON()
     │
     ▼
JSON text
     │
     ▼
echo
     │
     ▼
Display in workflow logs
```

This allows me to inspect what information GitHub provides when the workflow runs.

---

# 8. Event Payload Changes Depending on the Trigger

My workflow supports two triggers:

```yaml
pull_request
workflow_dispatch
```

The value of:

```yaml
github.event
```

depends on which event started the workflow.

For example:

```text
Workflow triggered by Pull Request
        │
        ▼
github.event contains pull request event data


Workflow triggered manually
        │
        ▼
github.event contains workflow_dispatch event data
```

This is an important concept.

> **The same workflow can receive different event payloads depending on which event triggered it.**

---

# 9. Path Filters

Another important type of filter is the `paths` filter.

Suppose I only want the workflow to run when application code changes.

```yaml
on:
  push:
    paths:
      - 'src/**'
```

Now:

```text
src/app.js changed
       │
       ▼
Workflow runs ✅


README.md changed
       │
       ▼
Workflow does not run ❌
```

I can also specify multiple paths:

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
      - 'package-lock.json'
```

This means the workflow runs if changes affect relevant application files.

---

# Example: Branch and Path Filters Together

I can combine filters.

```yaml
on:
  push:
    branches:
      - main
    paths:
      - 'src/**'
```

Now both conditions must match.

```text
Push to main
       │
       ▼
Did src/** change?
       │
    ┌──┴──┐
    │     │
   No    Yes
    │     │
  Stop   Run
```

Examples:

```text
Push to main + src/app.js changed
→ Workflow runs ✅


Push to main + README.md changed
→ Workflow does not run ❌


Push to feature/login + src/app.js changed
→ Workflow does not run ❌
```

---

# 10. `paths-ignore`

I can also ignore certain file changes.

For example:

```yaml
on:
  push:
    paths-ignore:
      - 'docs/**'
      - 'README.md'
```

Now:

```text
Only documentation changes
        │
        ▼
Workflow does not run


Application code changes
        │
        ▼
Workflow runs
```

This can reduce unnecessary CI runs.

---

# 11. Commented Events in My Workflow

In my original configuration, I wrote:

```yaml
on: #[push, workflow_dispatch, ]
```

The `#` symbol creates a YAML comment.

This means:

```yaml
#[push, workflow_dispatch, ]
```

is not an active trigger configuration.

GitHub ignores that line as a comment.

The actual active triggers are:

```yaml
on:
  pull_request:
    types:
      - opened

  workflow_dispatch:
```

If I wanted to activate the simple list syntax instead, I could write:

```yaml
on: [push, workflow_dispatch]
```

However, if I want to configure activity types or filters, I generally use the expanded YAML format.

For example:

```yaml
on:
  pull_request:
    types:
      - opened

  workflow_dispatch:
```

This is more flexible because each event can have its own configuration.

---

# 12. Event Filters Give More Control

Without filters:

```text
Event occurs
    │
    ▼
Workflow runs
```

With filters:

```text
Event occurs
    │
    ▼
Does event match activity type?
    │
    ▼
Does branch match?
    │
    ▼
Do changed files match?
    │
    ▼
Workflow runs only when required conditions match
```

This gives me more control over:

```text
WHEN a workflow runs
WHERE it runs from
WHAT activity triggers it
WHAT changes are relevant
```

---

# Complete Example: Controlled Pull Request Workflow

Here is an example with more precise control:

```yaml
name: Events Demo

on:
  pull_request:
    types:
      - opened
      - synchronize
    branches:
      - main
    paths:
      - 'src/**'
      - 'package.json'

  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Output event data
        run: echo "${{ toJSON(github.event) }}"

      - name: Get code
        uses: actions/checkout@v3

      - name: Install dependencies
        run: npm ci

      - name: Test code
        run: npm run test

      - name: Build code
        run: npm run build

      - name: Deploy project
        run: echo "Deploying..."
```

This workflow is more controlled.

It requires the following for an automatic pull request trigger:

```text
Pull Request Event
       │
       ▼
Activity is opened or synchronize?
       │
       ▼
Target branch is main?
       │
       ▼
Relevant files changed?
       │
       ▼
YES
       │
       ▼
Workflow runs
```

It can still always be started manually through:

```yaml
workflow_dispatch:
```

---

# Key Takeaways

```text
Event
│
└── The general activity that can trigger a workflow


Activity Type
│
└── A specific action within an event


Branch Filter
│
└── Controls which branches can trigger the workflow


Path Filter
│
└── Controls which file changes can trigger the workflow


workflow_dispatch
│
└── Allows manual execution regardless of automatic event filters
```

The most important mental model is:

```text
GitHub Activity Occurs
         │
         ▼
Does the event match `on`?
         │
         ▼
Does the activity type match?
         │
         ▼
Does the branch filter match?
         │
         ▼
Does the path filter match?
         │
         ▼
      Run Workflow
```

> **Events start workflows, but activity types and filters provide fine-grained control over exactly when the workflow is allowed to run. This helps prevent unnecessary workflow executions and allows CI/CD pipelines to run only for relevant changes and events.**
