# GitHub Actions: Building My First CI Workflow

I created the following GitHub Actions workflow to automatically test a Node.js project whenever code is pushed to GitHub.

## My Workflow

```yaml
name: test project

on: push

jobs:
  test-job:
    runs-on: ubuntu-latest

    steps:
      - name: getting the code
        uses: actions/checkout@v3

      - name: installing node
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: installing dependencies
        run: npm ci

      - name: running test
        run: npm test
```

This is a basic **Continuous Integration (CI) pipeline**.

The purpose is:

```text
Developer pushes code
        ↓
GitHub detects push
        ↓
GitHub Actions starts workflow
        ↓
Create Ubuntu runner
        ↓
Download project code
        ↓
Install/configure Node.js
        ↓
Install dependencies
        ↓
Run tests
        ↓
PASS ✅ or FAIL ❌
```

---

# 1. `name`

```yaml
name: test project
```

This gives the workflow a name.

GitHub will display this name in the **Actions** tab.

For example:

```text
Actions
│
└── test project
```

The name is for humans. It does not determine what the workflow does.

You could name it:

```yaml
name: Node.js CI
```

or:

```yaml
name: Run Tests
```

or:

```yaml
name: Application CI Pipeline
```

For a real project, something descriptive is usually better:

```yaml
name: Node.js CI
```

---

# 2. `on: push`

```yaml
on: push
```

The `on` section defines **when the workflow should run**.

Here:

```yaml
on: push
```

means:

> Run this workflow whenever a push event occurs in the repository.

For example, I make a change locally:

```bash
git add .
git commit -m "Add login feature"
git push
```

The `git push` sends my changes to GitHub.

GitHub detects the push event:

```text
git push
    ↓
GitHub receives new commit
    ↓
Push event
    ↓
GitHub Actions checks workflows
    ↓
"test project" workflow is triggered
```

---

# What Is an Event?

An **event** is something that happens in GitHub that can trigger a workflow.

Examples include:

```yaml
on: push
```

Run when code is pushed.

```yaml
on: pull_request
```

Run when a pull request is opened or updated.

```yaml
on:
  workflow_dispatch:
```

Allow the workflow to be manually started.

You can also combine events:

```yaml
on:
  push:
  pull_request:
```

Now the workflow can run when:

```text
Code is pushed
       OR
Pull request is created/updated
```

---

# Restricting the Push Trigger

Your current configuration:

```yaml
on: push
```

can run for pushes to any branch.

For example:

```text
push → main       → workflow runs
push → develop    → workflow runs
push → feature-x  → workflow runs
```

You can restrict it:

```yaml
on:
  push:
    branches:
      - main
```

Now:

```text
push → main
   ↓
Workflow runs ✅

push → develop
   ↓
Workflow does not run
```

This becomes very useful in real CI/CD pipelines.

---

# 3. `jobs`

```yaml
jobs:
```

The `jobs` section defines the jobs that belong to the workflow.

A workflow can contain one or many jobs.

Your workflow currently has one:

```yaml
jobs:
  test-job:
```

Conceptually:

```text
Workflow
   │
   └── test-job
```

A larger CI/CD workflow could have:

```yaml
jobs:
  test:
    ...

  build:
    ...

  deploy:
    ...
```

Which gives:

```text
Workflow
│
├── Test Job
├── Build Job
└── Deploy Job
```

---

# 4. `test-job`

```yaml
test-job:
```

This is the **job ID**.

It identifies the job within the workflow.

The job ID can be chosen by you.

For example:

```yaml
test:
```

```yaml
unit-tests:
```

```yaml
node-tests:
```

Your choice:

```yaml
test-job:
```

is perfectly fine.

The important thing is that it describes what the job does.

The job ID becomes particularly useful when one job depends on another.

For example:

```yaml
jobs:
  test:
    ...

  build:
    needs: test
```

This means:

```text
test
 ↓
build
```

The build job waits for the test job.

---

# 5. `runs-on`

```yaml
runs-on: ubuntu-latest
```

This tells GitHub where the job should execute.

`ubuntu-latest` means GitHub provides a GitHub-hosted Ubuntu runner for the job.

Think of the runner as a temporary computer created for your job.

```text
GitHub Actions
       ↓
Creates runner
       ↓
Ubuntu machine
       ↓
Runs your job
```

The runner provides an operating system and tools that your commands can use.

For example:

```yaml
runs-on: ubuntu-latest
```

or:

```yaml
runs-on: windows-latest
```

or:

```yaml
runs-on: macos-latest
```

---

# What Happens to the Runner?

Conceptually:

```text
Workflow starts
      ↓
GitHub assigns a runner
      ↓
Runner prepares environment
      ↓
Your job runs
      ↓
Job finishes
      ↓
Runner environment is discarded
```

This is important.

The runner should generally be treated as a **fresh environment for each job**.

You shouldn't assume that files installed or created on some previous unrelated job will already exist.

For example:

```text
Job 1
Ubuntu Runner
    ↓
Create file.txt
    ↓
Job finishes


Job 2
Different runner
    ↓
file.txt is not automatically there
```

If jobs need to exchange files, you need an explicit mechanism such as artifacts or an external storage system.

---

# 6. `steps`

```yaml
steps:
```

The `steps` section contains the individual tasks performed by the job.

Your job has four steps:

```text
test-job
│
├── 1. getting the code
├── 2. installing node
├── 3. installing dependencies
└── 4. running test
```

Steps within the same job normally execute sequentially:

```text
Step 1
  ↓
Step 2
  ↓
Step 3
  ↓
Step 4
```

If an important step fails, later steps normally do not execute.

For example:

```text
Checkout ✅
   ↓
Setup Node ✅
   ↓
npm ci ❌
   ↓
npm test ⏭️
```

This makes sense because there is no point running tests if the dependencies could not be installed.

---

# 7. Step 1 — Getting the Code

Your first step is:

```yaml
- name: getting the code
  uses: actions/checkout@v3
```

This step uses an existing GitHub Action.

The important part is:

```yaml
uses: actions/checkout@v3
```

---

# What Does `actions/checkout` Do?

When the runner starts, your project code is not simply sitting inside the runner.

The checkout action retrieves your repository's code so that subsequent steps can work with it.

Conceptually:

```text
GitHub Repository
       │
       │ checkout
       ▼
Ubuntu Runner
       │
       ▼
Your project files
```

Before checkout:

```text
Runner
│
└── No project source code
```

After checkout:

```text
Runner
│
└── Your repository
    ├── package.json
    ├── package-lock.json
    ├── src/
    └── test/
```

Now commands such as:

```bash
npm ci
```

can access your project files.

---

# What Does `uses` Mean?

There are two common ways a step performs work.

### `run`

```yaml
run: npm test
```

This executes a shell command.

### `uses`

```yaml
uses: actions/checkout@v3
```

This uses a reusable Action.

So:

```text
run
 ↓
Execute command/script

uses
 ↓
Use reusable Action
```

This distinction is extremely important in GitHub Actions.

---

# `actions/checkout@v3`

Break this down:

```yaml
uses: actions/checkout@v3
```

It has roughly this structure:

```text
actions/checkout
       │
       └── v3
```

`actions/checkout` identifies the Action.

`@v3` specifies the version/ref being used.

For current projects, you may encounter newer major versions as well. The key concept is that Actions are versioned, and you should intentionally choose and maintain the version you use.

---

# 8. Step 2 — Installing Node

Your second step:

```yaml
- name: installing node
  uses: actions/setup-node@v3
  with:
    node-version: 18
```

This uses another reusable Action:

```yaml
actions/setup-node
```

Its purpose is to configure Node.js in the runner environment.

---

# Why Do We Need This?

Your application is a Node.js application.

Your commands later include:

```bash
npm ci
```

and:

```bash
npm test
```

Those commands require Node.js/npm to be available.

So your workflow first prepares the environment.

```text
Ubuntu Runner
      ↓
Setup Node.js
      ↓
Node.js + npm available
      ↓
Run npm commands
```

---

# 9. The `with` Section

This is an important GitHub Actions concept.

You have:

```yaml
with:
  node-version: 18
```

`with` provides **inputs/configuration to an Action**.

Think of it like passing parameters to a function.

For example:

```text
Action:
setup-node

Input:
node-version = 18
```

Conceptually:

```text
actions/setup-node
        │
        │ node-version: 18
        ▼
Configure Node.js 18
```

Without configuration, an Action may use its defaults.

With `with`, you tell the Action exactly how you want it configured.

---

# `with` Is Not a Shell Command

This is important.

This:

```yaml
with:
  node-version: 18
```

does NOT mean GitHub executes:

```bash
node-version 18
```

Instead, it is configuration passed to the Action.

Compare:

```yaml
run: npm ci
```

with:

```yaml
with:
  node-version: 18
```

The first executes a command.

The second provides input to an Action.

---

# Example of `with`

You could configure Node differently:

```yaml
- name: Setup Node
  uses: actions/setup-node@v3
  with:
    node-version: 20
```

Now the Action configures Node.js 20.

You could also use:

```yaml
node-version: "18"
```

or:

```yaml
node-version: "20"
```

depending on your application's requirements.

The important principle is:

> The version used in CI should normally match the Node.js version your application expects.

---

# 10. Step 3 — Installing Dependencies

Your third step:

```yaml
- name: installing dependencies
  run: npm ci
```

This is different from the previous two steps.

Instead of:

```yaml
uses:
```

you use:

```yaml
run:
```

This means:

> Execute a command on the runner.

The command is:

```bash
npm ci
```

---

# What Is `npm ci`?

`npm ci` means **clean install**.

It is designed especially for automated environments such as CI pipelines.

It uses the project's lock file, typically:

```text
package-lock.json
```

to install the exact dependency versions recorded there.

For example:

```text
package.json
       +
package-lock.json
       ↓
     npm ci
       ↓
node_modules/
```

Suppose your project has:

```json
{
  "dependencies": {
    "express": "4.18.2"
  }
}
```

and the lock file records the exact dependency tree.

`npm ci` installs based on that lock file.

---

# Why Use `npm ci` in CI?

Reproducibility.

Imagine your application worked yesterday.

Today a dependency releases a newer version.

If your installation process is allowed to resolve versions differently, you could unexpectedly get different dependency versions.

A lock file helps define the exact dependency tree.

Therefore:

```text
Developer Machine
       ↓
package-lock.json
       ↓
CI Runner
       ↓
npm ci
       ↓
Same locked dependency versions
```

This makes CI environments more predictable.

---

# `npm install` vs `npm ci`

This is an important distinction.

## `npm install`

Usually used during development.

```bash
npm install
```

It can install dependencies and may update the lock file when dependency definitions change.

## `npm ci`

Designed for clean, reproducible installs, especially in CI.

```bash
npm ci
```

It expects the lock file and installs according to it.

A common CI pattern is:

```text
npm ci
   ↓
npm test
```

---

# 11. Step 4 — Running Tests

Your final step:

```yaml
- name: running test
  run: npm test
```

Again, `run` means:

> Execute this command on the runner.

The command is:

```bash
npm test
```

---

# Where Does `npm test` Come From?

Usually, `npm test` is defined by the `scripts` section of `package.json`.

For example:

```json
{
  "scripts": {
    "test": "jest"
  }
}
```

When you run:

```bash
npm test
```

npm looks inside:

```text
package.json
   ↓
scripts
   ↓
test
   ↓
jest
```

So:

```bash
npm test
```

effectively runs:

```bash
jest
```

depending on how your project defines the script.

Another example:

```json
{
  "scripts": {
    "test": "node test.js"
  }
}
```

Then:

```bash
npm test
```

runs:

```bash
node test.js
```

Therefore, `npm test` is not itself a specific testing framework.

It is an npm script defined by your project.

---

# Complete Example `package.json`

For example:

```json
{
  "name": "my-node-app",
  "version": "1.0.0",
  "scripts": {
    "test": "jest"
  },
  "dependencies": {
    "express": "4.18.2"
  },
  "devDependencies": {
    "jest": "29.7.0"
  }
}
```

The workflow executes:

```bash
npm ci
```

which installs dependencies.

Then:

```bash
npm test
```

which executes the project's test script.

Conceptually:

```text
package.json
     │
     ├── dependencies
     │
     ├── devDependencies
     │
     └── scripts
           │
           └── test → jest
```

---

# What Happens When a Test Fails?

Suppose you have:

```javascript
test("addition works", () => {
    expect(2 + 2).toBe(5);
});
```

The test fails because:

```text
2 + 2 = 4
```

not 5.

When GitHub Actions runs:

```bash
npm test
```

the testing framework returns a non-zero exit code.

Conceptually:

```text
npm test
   ↓
Test executes
   ↓
Test fails ❌
   ↓
Command returns failure
   ↓
Step fails
   ↓
Job fails
   ↓
Workflow fails
```

GitHub then shows the workflow as failed.

```text
❌ test project
   └── ❌ test-job
       └── ❌ running test
```

This is one of the major benefits of CI.

A developer can immediately see that their change caused a test failure.

---

# What Happens When All Tests Pass?

If everything works:

```text
Checkout          ✅
Setup Node        ✅
npm ci             ✅
npm test           ✅
```

Then:

```text
Step 1 ✅
   ↓
Step 2 ✅
   ↓
Step 3 ✅
   ↓
Step 4 ✅
   ↓
Job SUCCESS ✅
   ↓
Workflow SUCCESS ✅
```

GitHub reports the workflow as successful.

---

# Complete Execution From Start to Finish

Let's imagine I modify my application.

```bash
git add .
git commit -m "Add login feature"
git push origin main
```

Now the complete process is:

```text
                  Developer
                      │
                      │ git push
                      ▼
              GitHub Repository
                      │
                      │ push event
                      ▼
              GitHub Actions
                      │
                      ▼
              "test project"
                  Workflow
                      │
                      ▼
                 "test-job"
                     Job
                      │
                      ▼
             ubuntu-latest Runner
                      │
                      ▼
          ┌───────────────────────┐
          │ Step 1                │
          │ actions/checkout      │
          │                       │
          │ Get repository code   │
          └───────────┬───────────┘
                      │
                      ▼
          ┌───────────────────────┐
          │ Step 2                │
          │ actions/setup-node    │
          │                       │
          │ Configure Node 18     │
          └───────────┬───────────┘
                      │
                      ▼
          ┌───────────────────────┐
          │ Step 3                │
          │ npm ci                │
          │                       │
          │ Install dependencies  │
          └───────────┬───────────┘
                      │
                      ▼
          ┌───────────────────────┐
          │ Step 4                │
          │ npm test              │
          │                       │
          │ Run automated tests   │
          └───────────┬───────────┘
                      │
                      ▼
                 SUCCESS ✅
```

---

# The Runner's Perspective

Another useful way to understand this is to imagine you are the Ubuntu runner.

GitHub gives you a fresh environment.

### Step 1

```yaml
uses: actions/checkout@v3
```

You receive the repository.

Now you have:

```text
my-project/
├── package.json
├── package-lock.json
├── src/
└── tests/
```

### Step 2

```yaml
uses: actions/setup-node@v3
with:
  node-version: 18
```

Now Node.js is configured.

You can run:

```bash
node --version
```

and:

```bash
npm --version
```

### Step 3

```yaml
run: npm ci
```

Dependencies are installed.

Now you have:

```text
my-project/
├── package.json
├── package-lock.json
├── node_modules/
├── src/
└── tests/
```

### Step 4

```yaml
run: npm test
```

The tests execute.

```text
Tests
│
├── Test 1 ✅
├── Test 2 ✅
├── Test 3 ✅
└── Test 4 ✅
```

The job succeeds.

---

# Why Is This CI?

This workflow is a **CI workflow** because it automatically validates code changes.

The pipeline is:

```text
Code Change
    ↓
Push
    ↓
Automated Environment
    ↓
Install Dependencies
    ↓
Run Tests
    ↓
Feedback
```

The developer does not have to manually:

```text
1. Download code
2. Install Node
3. Install dependencies
4. Run tests
5. Check results
```

GitHub Actions does it automatically.

---

# Turning This Into a More Realistic CI Pipeline

Your current workflow is:

```text
Checkout
   ↓
Setup Node
   ↓
Install Dependencies
   ↓
Run Tests
```

A real-world application might expand this:

```text
Checkout
   ↓
Setup Node
   ↓
Install Dependencies
   ↓
Lint
   ↓
Unit Tests
   ↓
Integration Tests
   ↓
Build Application
   ↓
Security Scan
```

For example:

```yaml
name: Node.js CI

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

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Run lint
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Build application
        run: npm run build
```

Now the pipeline becomes:

```text
             Push
               ↓
        GitHub Actions
               ↓
          Checkout
               ↓
         Setup Node
               ↓
          npm ci
               ↓
          npm run lint
               ↓
           npm test
               ↓
         npm run build
               ↓
          CI SUCCESS
```

---

# CI/CD Version

Eventually, you can add deployment.

For example:

```text
                 Push
                   ↓
              Test Job
                   ↓
            Tests Pass?
             /       \
           No         Yes
           ↓           ↓
         STOP      Build Job
                       ↓
                 Docker Image
                       ↓
                  Push to ECR
                       ↓
                 Deploy to ECS
```

This is where GitHub Actions becomes a complete DevOps CI/CD platform.

---

# Important Concepts Learned

From this workflow, I have now used several fundamental GitHub Actions concepts.

| Concept                 | Example                     | Purpose                             |
| ----------------------- | --------------------------- | ----------------------------------- |
| Workflow name           | `name: test project`        | Identifies the workflow             |
| Event                   | `on: push`                  | Determines when workflow runs       |
| Job                     | `test-job:`                 | Defines a unit of work              |
| Runner                  | `ubuntu-latest`             | Defines where job executes          |
| Step                    | `- name:`                   | Defines an individual task          |
| Action                  | `uses: actions/checkout@v3` | Reuses packaged automation          |
| Action input            | `with:`                     | Provides configuration to an Action |
| Shell command           | `run: npm ci`               | Executes a command on runner        |
| Dependency installation | `npm ci`                    | Installs locked dependencies        |
| Testing                 | `npm test`                  | Executes project's test script      |

---

# The Most Important Mental Model

When looking at this workflow:

```yaml
name: test project

on: push

jobs:
  test-job:
    runs-on: ubuntu-latest

    steps:
      - name: getting the code
        uses: actions/checkout@v3

      - name: installing node
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: installing dependencies
        run: npm ci

      - name: running test
        run: npm test
```

I should mentally translate it into:

```text
NAME
│
└── test project

TRIGGER
│
└── push

JOB
│
└── test-job
      │
      └── RUNNER
            │
            └── Ubuntu
                  │
                  └── STEPS
                        │
                        ├── Checkout repository
                        │      └── Use an Action
                        │
                        ├── Setup Node.js 18
                        │      └── Use an Action
                        │
                        ├── npm ci
                        │      └── Run shell command
                        │
                        └── npm test
                               └── Run shell command
```

Or, even more simply:

```text
REPOSITORY
     ↓
WORKFLOW
     ↓
EVENT: push
     ↓
JOB
     ↓
RUNNER: Ubuntu
     ↓
STEP
     ↓
ACTION / COMMAND
```

---

# Final Takeaway

This workflow is my first real **Continuous Integration pipeline**.

The purpose is to automatically verify that my Node.js project still works whenever I push code.

The entire idea can be summarized as:

```text
Developer writes code
        ↓
Developer pushes code
        ↓
GitHub receives push event
        ↓
Workflow starts
        ↓
Job starts on Ubuntu runner
        ↓
Checkout repository
        ↓
Setup Node.js
        ↓
Install dependencies
        ↓
Run tests
        ↓
       ┌───────────────┐
       │               │
    Tests Pass      Tests Fail
       │               │
       ▼               ▼
   CI Success      CI Failure
```

The key principle of CI is:

> **Every code change should be automatically validated so that problems are discovered as early as possible.**

My workflow is currently only doing **CI**. Later, I can extend it into **CI/CD** by adding build and deployment stages such as building a Docker image, pushing it to Amazon ECR, and deploying it to Amazon ECS.
