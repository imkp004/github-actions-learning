# GitHub Actions: More Complex Event Configuration

This workflow builds on the basic event concepts by adding **multiple triggers, activity types, branch filters, branch patterns, and path filters**.

The most important idea is that GitHub Actions allows me to control **exactly when a workflow should run**.

My workflow is:

```yaml
name: Events Demo 1

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

    paths:
      # Path patterns can be added here

    paths-ignore:
      # Ignored path patterns can be added here

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

---

# 1. The Big Picture

This workflow has **three possible triggers**:

```text
                    Events Demo 1
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
   pull_request   workflow_dispatch       push
          │               │                │
          └───────────────┼────────────────┘
                          │
                          ▼
                      deploy job
```

The workflow can start because:

1. A matching pull request is opened.
2. The workflow is manually started.
3. Code is pushed to a matching branch and passes the configured path filters.

The `on` section is becoming more powerful because each event has its **own rules**.

---

# 2. `pull_request` Event with Multiple Controls

My pull request configuration is:

```yaml
pull_request:
  types:
    - opened
  branches:
    - main
    - 'dev-*'
    - 'feat/**'
```

For this workflow to run automatically from a pull request, the event must satisfy the configured conditions.

---

# 3. `types: opened`

```yaml
types:
  - opened
```

This controls the **activity type**.

A pull request can have multiple activities:

```text
Pull Request
│
├── opened
├── closed
├── reopened
├── synchronize
├── edited
├── labeled
└── unlabeled
```

My workflow specifically listens for:

```text
opened
```

Therefore:

```text
Pull Request opened
        │
        ▼
Workflow may run ✅
```

But:

```text
Pull Request closed
        │
        ▼
Workflow does not run ❌
```

And:

```text
New commits pushed to an existing PR
        │
        ▼
synchronize activity
        │
        ▼
Workflow does not run with this configuration ❌
```

This is important because my workflow is intentionally restricted to the moment when a pull request is first created.

---

# 4. Pull Request Branch Filters

My workflow also contains:

```yaml
branches:
  - main
  - 'dev-*'
  - 'feat/**'
```

For a `pull_request` event, the `branches` filter refers to the **target branch of the pull request**.

For example:

```text
Source Branch                     Target Branch

feature/login  ───── PR ─────►       main
```

Since `main` is listed:

```yaml
branches:
  - main
```

the workflow matches.

However:

```text
feature/login  ───── PR ─────► production
```

does not match the configured branch patterns, so the workflow does not run automatically.

The important rule is:

> **For `pull_request`, `branches` filters the branch the pull request is targeting.**

---

# 5. Exact Branch Match: `main`

```yaml
- main
```

This matches exactly:

```text
main
```

Examples:

```text
PR → main        ✅ Match
PR → Main        ❌ No match
PR → main-dev    ❌ No match
PR → develop     ❌ No match
```

Branch matching is based on the configured pattern.

---

# 6. Single-Level Wildcard: `dev-*`

My configuration contains:

```yaml
- 'dev-*'
```

The `*` wildcard represents a sequence of characters.

This can match branches such as:

```text
dev-test         ✅
dev-staging      ✅
dev-kirtan       ✅
dev-123          ✅
```

But:

```text
development      ❌
main             ❌
feature/login    ❌
```

The pattern is useful when my branches follow a naming convention such as:

```text
dev-test
dev-staging
dev-production
```

Conceptually:

```text
dev-*
  │
  └── dev- followed by matching characters
```

---

# 7. Recursive Branch Pattern: `feat/**`

My workflow also contains:

```yaml
- 'feat/**'
```

The `**` pattern is useful for matching branch names that can contain multiple levels.

Examples:

```text
feat/login              ✅
feat/payment            ✅
feat/api/v2             ✅
feat/user/profile       ✅
```

Conceptually:

```text
feat/**
  │
  ├── feat/login
  ├── feat/payment
  ├── feat/api/v2
  └── feat/user/profile
```

This is useful when a team uses structured branch naming.

For example:

```text
feat/
├── authentication
├── payments
└── api/
    ├── v1
    └── v2
```

The `feat/**` pattern can match paths or names below `feat/`.

---

# 8. Combining Pull Request Conditions

My `pull_request` trigger combines multiple conditions.

Conceptually:

```text
Pull Request Event
       │
       ▼
Is activity "opened"?
       │
   ┌───┴────┐
   │        │
  No       Yes
   │        │
 Stop       ▼
     Does target branch match?
             │
        ┌────┴────┐
        │         │
       No        Yes
        │         │
       Stop      Run
```

For example:

### Example 1

```text
PR opened
Target: main
```

Result:

```text
Workflow runs ✅
```

### Example 2

```text
PR opened
Target: dev-testing
```

Result:

```text
Matches dev-* → Workflow runs ✅
```

### Example 3

```text
PR opened
Target: feat/api/v2
```

Result:

```text
Matches feat/** → Workflow runs ✅
```

### Example 4

```text
PR opened
Target: production
```

Result:

```text
No branch pattern matches → Workflow does not run ❌
```

### Example 5

```text
PR closed
Target: main
```

Result:

```text
Branch matches
BUT activity is not opened
        ↓
Workflow does not run ❌
```

Both configured requirements must match.

---

# 9. `workflow_dispatch`

My workflow also contains:

```yaml
workflow_dispatch:
```

This allows me to start the workflow manually.

```text
GitHub Repository
       │
       ▼
Actions Tab
       │
       ▼
Select Events Demo 1
       │
       ▼
Run workflow
       │
       ▼
Workflow starts
```

This is independent of the automatic `pull_request` and `push` triggers.

For example, I can manually run the workflow even if:

```text
No pull request was opened
```

and:

```text
No new code was pushed
```

This is useful for:

```text
Testing the workflow
Debugging
Troubleshooting
Manual deployments
Learning and experimentation
```

---

# 10. `push` Event with Branch Filters

My push configuration is:

```yaml
push:
  branches:
    - main
    - 'dev-*'
    - 'feat/**'
```

This means the workflow can automatically run when code is pushed to matching branches.

Conceptually:

```text
git push
    │
    ▼
Which branch received the push?
    │
    ├── main?
    ├── dev-*?
    └── feat/**?
            │
        Match found?
            │
        ┌───┴────┐
        │        │
       No       Yes
        │        │
      Stop     Continue
```

Examples:

```text
Push to main

main → Match → Workflow runs ✅
```

```text
Push to dev-testing

dev-* → Match → Workflow runs ✅
```

```text
Push to feat/login

feat/** → Match → Workflow runs ✅
```

```text
Push to hotfix/login

No configured pattern matches → Workflow does not run ❌
```

---

# 11. Difference Between `push` and `pull_request` Branch Filters

This is one of the most important concepts.

The same configuration:

```yaml
branches:
  - main
```

means something different depending on the event.

## With `push`

```yaml
on:
  push:
    branches:
      - main
```

It means:

> Run when code is pushed **to `main`**.

```text
git push → main
      │
      ▼
Workflow runs
```

---

## With `pull_request`

```yaml
on:
  pull_request:
    branches:
      - main
```

It means:

> Run when a pull request is targeting `main`.

```text
feature/login
      │
      │ Pull Request
      ▼
     main
      │
      ▼
Workflow runs
```

So the mental model is:

```text
push:
  branches = Where was code pushed?


pull_request:
  branches = Which branch is the PR targeting?
```

---

# 12. Path Filters

My workflow contains placeholders for:

```yaml
paths:
```

and:

```yaml
paths-ignore:
```

These allow me to control workflow execution based on **which files changed**.

This is useful because sometimes a branch matches, but the changed files are not relevant to the workflow.

---

# 13. Using `paths`

Suppose I configure:

```yaml
push:
  branches:
    - main

  paths:
    - 'src/**'
```

Now the workflow needs a push to `main` and a matching file change.

Example:

```text
Push to main
     │
     ▼
Did a file in src/** change?
     │
 ┌───┴────┐
 │        │
No       Yes
 │        │
Stop     Run
```

Examples:

```text
Branch: main
Changed: src/app.js

Result: Workflow runs ✅
```

```text
Branch: main
Changed: README.md

Result: Workflow does not run ❌
```

```text
Branch: feature/login
Changed: src/app.js

Result: Workflow does not run ❌
```

The branch and path conditions work together.

---

# 14. Multiple Path Patterns

I can configure multiple relevant paths:

```yaml
paths:
  - 'src/**'
  - 'package.json'
  - 'package-lock.json'
```

Now the workflow runs if changes match any of those patterns.

```text
Changed File

src/app.js              → Run ✅
src/api/users.js        → Run ✅
package.json            → Run ✅
package-lock.json       → Run ✅
README.md               → No match ❌
docs/setup.md           → No match ❌
```

This is useful because my CI workflow should run when application code or dependencies change.

---

# 15. Using `paths-ignore`

`paths-ignore` does the opposite.

Instead of specifying files that should trigger the workflow, I specify files that should be ignored.

For example:

```yaml
paths-ignore:
  - 'docs/**'
  - 'README.md'
```

Conceptually:

```text
Push occurs
     │
     ▼
Are all relevant changes ignored?
     │
     ├── Documentation only
     │        │
     │        ▼
     │     Do not run
     │
     └── Application code changed
              │
              ▼
           Run workflow
```

Examples:

```text
Changed: README.md

Ignored → Workflow does not run
```

```text
Changed: docs/setup.md

Ignored → Workflow does not run
```

```text
Changed: src/app.js

Not ignored → Workflow runs
```

---

# 16. `paths` vs `paths-ignore`

These provide two different approaches.

## `paths`

```yaml
paths:
  - 'src/**'
```

Means:

> Run only when these relevant files change.

```text
Relevant files change → Run
Other files change    → Do not run
```

---

## `paths-ignore`

```yaml
paths-ignore:
  - 'docs/**'
```

Means:

> Run for changes except ignored files.

```text
Documentation only → Do not run
Other changes      → Run
```

A useful way to think about them is:

```text
paths
│
└── What changes DO I care about?


paths-ignore
│
└── What changes DO I NOT care about?
```

---

# 17. Example of My `push` Configuration

A completed version could look like:

```yaml
push:
  branches:
    - main
    - 'dev-*'
    - 'feat/**'

  paths:
    - 'src/**'
    - 'package.json'
    - 'package-lock.json'

  paths-ignore:
    - 'docs/**'
    - 'README.md'
```

The intention is:

```text
Push Event
    │
    ▼
Does branch match?
    │
    ▼
Does the file change match my rules?
    │
    ▼
If all trigger requirements match
    │
    ▼
Workflow runs
```

> In practice, `paths` and `paths-ignore` should be used according to the exact trigger logic needed. I should carefully validate the patterns because they determine whether a workflow runs or is skipped.

---

# 18. Outputting Event Data

My first workflow step is:

```yaml
- name: Output event data
  run: echo "${{ toJSON(github.event) }}"
```

This prints information about the event that triggered the workflow.

The flow is:

```text
GitHub Event
     │
     ▼
Event Payload Created
     │
     ▼
github.event
     │
     ▼
toJSON()
     │
     ▼
echo
     │
     ▼
Workflow Logs
```

Since my workflow supports multiple events:

```text
pull_request
workflow_dispatch
push
```

the event data can be different for each workflow run.

For example:

```text
Push triggered workflow
        │
        ▼
github.event contains push-related data
```

```text
Pull request triggered workflow
        │
        ▼
github.event contains pull-request-related data
```

```text
Manual trigger
        │
        ▼
github.event contains workflow_dispatch-related data
```

This makes the output step useful for learning and debugging.

---

# 19. The Job Execution Flow

Once one of the triggers successfully matches, the workflow starts the `deploy` job.

```text
Matching Event
       │
       ▼
deploy job
       │
       ▼
Get code
       │
       ▼
Install dependencies
       │
       ▼
Test code
       │
       ▼
Build code
       │
       ▼
Deploy project
```

The steps run sequentially.

This means:

```text
Install dependencies must complete
            ↓
Tests run
            ↓
Tests must complete successfully
            ↓
Build runs
            ↓
Build must complete successfully
            ↓
Deployment runs
```

By default, if an earlier step fails, later steps are normally not executed.

For example:

```text
npm ci       ✅
npm run test ❌
npm run build ⏭️ Skipped
Deploy        ⏭️ Skipped
```

This creates a basic safety mechanism in the pipeline.

---

# 20. Complete Trigger Decision Flow

My workflow can be visualized as:

```text
                    GitHub Activity
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
      Pull Request      Push        Manual Run
             │            │            │
             ▼            ▼            │
       Is it opened?  Branch match?    │
             │            │            │
             ▼            ▼            │
       Target branch?  Path rules?     │
             │            │            │
             └────────────┼────────────┘
                          │
                   Trigger matches
                          │
                          ▼
                     deploy job
                          │
                          ▼
                 Output event data
                          │
                          ▼
                     Checkout code
                          │
                          ▼
                Install dependencies
                          │
                          ▼
                     Run tests
                          │
                          ▼
                    Build project
                          │
                          ▼
                   Deploy project
```

---

# Key Concepts Learned

```text
Event
│
└── The general GitHub activity that can trigger a workflow


Activity Type
│
└── A specific action within an event

Example:
pull_request → opened


Branch Filter
│
└── Restricts workflow execution to matching branches

Examples:
main
dev-*
feat/**


Path Filter
│
└── Runs the workflow only when specified files change


Path Ignore
│
└── Prevents the workflow from running for specified file changes


workflow_dispatch
│
└── Allows the workflow to be started manually
```

---

# Final Mental Model

The `on` section acts like a **filtering system**.

```text
GitHub Activity
       │
       ▼
Is it a configured event?
       │
       ▼
Does the activity type match?
       │
       ▼
Does the branch match?
       │
       ▼
Do the changed files match the path rules?
       │
       ▼
Start Workflow
```

The more configuration I add to an event, the more precisely I can control when the workflow executes.

> **Events provide the trigger, activity types narrow down specific actions, branch filters control where the event occurs, and path filters control which code changes are relevant. Together, these features allow me to build efficient CI/CD workflows that run only when the required conditions are met.**
