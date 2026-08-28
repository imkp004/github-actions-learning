# My First GitHub Actions Workflows

I created my first GitHub Actions workflows to understand the basic structure of:

```text
Workflow
   ↓
Trigger
   ↓
Job
   ↓
Runner
   ↓
Steps
   ↓
Commands
```

These examples are simple, but they demonstrate the fundamental architecture of GitHub Actions.

---

# Workflow 1: Hello and Goodbye

## Corrected Workflow

```yaml
name: my first workflow

on:
  workflow_dispatch:

jobs:
  first-job:
    runs-on: ubuntu-latest

    steps:
      - name: hello from Kirtan
        run: echo "Hello Kirtan"

      - name: goodbye
        run: echo "Goodbye Kirtan"
```

---

# Complete Breakdown

## 1. `name`

```yaml
name: my first workflow
```

The `name` gives the workflow a human-readable name.

This is the name that appears in the GitHub Actions interface.

Conceptually:

```text
GitHub Repository
       ↓
Actions Tab
       ↓
my first workflow
```

The workflow name is mainly used to make the workflow easy to identify.

For example, a repository might have several workflows:

```yaml
name: CI Pipeline
```

```yaml
name: Deploy to Production
```

```yaml
name: Terraform Validation
```

```yaml
name: Security Scan
```

These names make it easier to understand what each workflow does.

---

# 2. `on`

```yaml
on:
  workflow_dispatch:
```

The `on` keyword defines the **event or trigger** that starts the workflow.

In this case:

```yaml
workflow_dispatch:
```

means the workflow can be started **manually**.

The flow looks like this:

```text
You
 │
 │ Open GitHub Repository
 ▼
Actions Tab
 │
 │ Select Workflow
 ▼
Click "Run workflow"
 │
 ▼
GitHub Actions starts the workflow
```

Unlike a `push` trigger, this workflow will not automatically run every time you push code.

For comparison:

### Manual trigger

```yaml
on:
  workflow_dispatch:
```

```text
You click Run workflow
        ↓
Workflow starts
```

### Push trigger

```yaml
on:
  push:
```

```text
Developer pushes code
        ↓
Workflow starts automatically
```

### Pull request trigger

```yaml
on:
  pull_request:
```

```text
Pull request created or updated
        ↓
Workflow starts automatically
```

Your first workflow uses `workflow_dispatch` because you want to manually run the workflow while learning.

---

# 3. `jobs`

```yaml
jobs:
```

The `jobs` section contains all the jobs that belong to the workflow.

A workflow can have:

```text
One Workflow
     │
     ├── One Job
     │
     └── Multiple Jobs
```

For example:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

  build:
    runs-on: ubuntu-latest

  deploy:
    runs-on: ubuntu-latest
```

Your workflow contains one job:

```yaml
jobs:
  first-job:
```

---

# 4. `first-job`

```yaml
first-job:
```

This is the **job ID**.

The job ID identifies the job inside the workflow.

```text
Workflow
    │
    ▼
Job ID: first-job
```

The job ID is important because other jobs can reference it.

For example:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

  deploy:
    needs: test
    runs-on: ubuntu-latest
```

Here:

```text
Job: test
    ↓
must complete first
    ↓
Job: deploy
```

The `deploy` job references the `test` job using:

```yaml
needs: test
```

In your workflow:

```yaml
first-job:
```

is simply the identifier for your job.

You could name it:

```yaml
hello-job:
```

or:

```yaml
greeting-job:
```

The important thing is that the job ID should clearly describe its purpose.

---

# 5. `runs-on`

```yaml
runs-on: ubuntu-latest
```

Every job needs an execution environment.

This is called a **runner**.

Your job tells GitHub:

> Run this job on an Ubuntu runner.

Conceptually:

```text
GitHub Actions
       │
       ▼
Creates or assigns a runner
       │
       ▼
Ubuntu Environment
       │
       ▼
Executes your job
```

The runner executes the commands defined in your steps.

For example:

```yaml
runs-on: ubuntu-latest
```

means your commands will run on an Ubuntu environment.

Other examples include:

```yaml
runs-on: windows-latest
```

or:

```yaml
runs-on: macos-latest
```

You can also use your own infrastructure with self-hosted runners:

```yaml
runs-on: self-hosted
```

A useful mental model is:

```text
Job = What work needs to be done

Runner = Where the work is executed
```

Your workflow says:

```text
Job: first-job
        ↓
Runner: ubuntu-latest
        ↓
Execute the steps
```

---

# 6. `steps`

```yaml
steps:
```

A job contains one or more steps.

Each step performs an individual task.

Your workflow has two steps:

```text
first-job
    │
    ├── Step 1: Hello from Kirtan
    │
    └── Step 2: Goodbye
```

Steps inside the same job normally run **sequentially**.

This means:

```text
Step 1
   ↓
Step 1 finishes
   ↓
Step 2
   ↓
Step 2 finishes
   ↓
Job completes
```

GitHub does not normally run these two steps in parallel.

---

# 7. First Step: Hello

```yaml
- name: hello from Kirtan
  run: echo "Hello Kirtan"
```

The `-` indicates that this is an item in the list of steps.

The step contains two parts:

```text
Step
 │
 ├── name
 │
 └── run
```

---

## `name`

```yaml
name: hello from Kirtan
```

This gives the individual step a readable name.

GitHub displays this name in the workflow logs.

Conceptually:

```text
first-job
│
├── ✅ hello from Kirtan
│
└── ⏳ goodbye
```

The `name` is mainly for humans.

It makes the workflow easier to read and debug.

Compare these two workflows:

### Less descriptive

```yaml
steps:
  - run: echo "Hello Kirtan"
  - run: echo "Goodbye Kirtan"
```

### More descriptive

```yaml
steps:
  - name: Print greeting message
    run: echo "Hello Kirtan"

  - name: Print goodbye message
    run: echo "Goodbye Kirtan"
```

The second version is easier to understand when looking at the GitHub Actions logs.

---

# 8. `run`

```yaml
run: echo "Hello Kirtan"
```

The `run` keyword executes a command on the runner.

Because your job is running on Ubuntu:

```yaml
runs-on: ubuntu-latest
```

GitHub executes the command in the Ubuntu environment.

Conceptually:

```text
GitHub Actions Step
        │
        ▼
Ubuntu Runner
        │
        ▼
Shell executes:
echo "Hello Kirtan"
        │
        ▼
Output:
Hello Kirtan
```

The `echo` command prints text to the terminal.

So:

```bash
echo "Hello Kirtan"
```

produces:

```text
Hello Kirtan
```

You can see this output in the GitHub Actions logs.

---

# 9. Second Step: Goodbye

```yaml
- name: goodbye
  run: echo "Goodbye Kirtan"
```

This is another step.

Because it comes after the first step, it runs after the greeting step completes.

The complete execution is:

```text
first-job starts
        ↓
Runner starts
        ↓
Step 1: hello from Kirtan
        ↓
Output: Hello Kirtan
        ↓
Step 1 completes
        ↓
Step 2: goodbye
        ↓
Output: Goodbye Kirtan
        ↓
Step 2 completes
        ↓
Job completes successfully
```

The workflow output would conceptually look like:

```text
Hello Kirtan
Goodbye Kirtan
```

---

# Workflow 2: Greeting and Goodbye

## Corrected Workflow

```yaml
name: my second workflow

on:
  workflow_dispatch:

jobs:
  second-job:
    runs-on: ubuntu-latest

    steps:
      - name: greeting and goodbye
        run: |
          echo "Hello"
          echo "Goodbye"
```

I corrected:

```text
seocnd-job → second-job
goodbuye → Goodbye
```

These corrections are mainly spelling improvements. The important structural parts of the workflow remain the same.

---

# What Is Different in the Second Workflow?

Your second workflow teaches an important concept:

```yaml
run: |
```

Instead of using two separate steps, you placed multiple commands inside **one step**.

---

# The `|` Symbol

Look at this:

```yaml
run: |
  echo "Hello"
  echo "Goodbye"
```

The `|` allows you to write a **multi-line command block**.

Conceptually:

```text
One Step
    │
    ├── Command 1: echo "Hello"
    │
    └── Command 2: echo "Goodbye"
```

Both commands belong to the same step.

The runner executes:

```bash
echo "Hello"
echo "Goodbye"
```

The output is:

```text
Hello
Goodbye
```

---

# First Workflow vs Second Workflow

Your first workflow uses **two steps**:

```yaml
steps:
  - name: hello from Kirtan
    run: echo "Hello Kirtan"

  - name: goodbye
    run: echo "Goodbye Kirtan"
```

Architecture:

```text
Job
│
├── Step 1
│   └── echo "Hello Kirtan"
│
└── Step 2
    └── echo "Goodbye Kirtan"
```

Your second workflow uses **one step containing multiple commands**:

```yaml
steps:
  - name: greeting and goodbye
    run: |
      echo "Hello"
      echo "Goodbye"
```

Architecture:

```text
Job
│
└── Step 1
    │
    ├── echo "Hello"
    │
    └── echo "Goodbye"
```

---

# Why Use Multiple Steps?

Suppose your CI pipeline has:

```yaml
steps:
  - name: Install dependencies
    run: pip install -r requirements.txt

  - name: Run tests
    run: pytest

  - name: Build application
    run: python build.py
```

This is useful because each major task is separate.

In GitHub Actions, you can clearly see which task failed:

```text
✅ Install dependencies
❌ Run tests
⏭️ Build application
```

You immediately know:

> The tests failed.

This makes debugging easier.

---

# Why Use Multiple Commands in One Step?

Sometimes several commands are part of the same logical task.

For example:

```yaml
- name: Set up application
  run: |
    mkdir app
    cd app
    touch config.txt
    echo "environment=development" > config.txt
```

These commands are all part of one task:

```text
Set up application
    │
    ├── Create directory
    ├── Enter directory
    ├── Create configuration file
    └── Add configuration
```

Putting them in one step can make sense because they belong together.

---

# Important Difference: Step Failure

This is an important concept.

Consider this:

```yaml
- name: Greeting and goodbye
  run: |
    echo "Hello"
    false
    echo "Goodbye"
```

If a command causes the step to fail, the step can stop and the job can fail.

Conceptually:

```text
Step starts
    ↓
echo "Hello" ✅
    ↓
false ❌
    ↓
Step fails
    ↓
Job can fail
```

This behavior is useful because CI/CD pipelines should stop when an important command fails.

For example:

```text
Install dependencies
       ↓
Run tests ❌
       ↓
Do not deploy broken application
```

---

# Execution Flow of My First Workflow

```text
MANUAL TRIGGER
workflow_dispatch
        │
        ▼
┌───────────────────────────────┐
│      my first workflow        │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│         first-job             │
│                               │
│ Runner: ubuntu-latest         │
└───────────────┬───────────────┘
                │
                ▼
        Step 1 starts
                │
                ▼
   echo "Hello Kirtan"
                │
                ▼
        Step 1 succeeds
                │
                ▼
        Step 2 starts
                │
                ▼
  echo "Goodbye Kirtan"
                │
                ▼
        Step 2 succeeds
                │
                ▼
          Job succeeds
                │
                ▼
       Workflow succeeds
```

---

# Execution Flow of My Second Workflow

```text
MANUAL TRIGGER
workflow_dispatch
        │
        ▼
┌───────────────────────────────┐
│      my second workflow       │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│         second-job            │
│                               │
│ Runner: ubuntu-latest         │
└───────────────┬───────────────┘
                │
                ▼
     Step: greeting and goodbye
                │
                ▼
        Command 1 runs
        echo "Hello"
                │
                ▼
        Command 2 runs
       echo "Goodbye"
                │
                ▼
          Step succeeds
                │
                ▼
           Job succeeds
                │
                ▼
       Workflow succeeds
```

---

# Complete Comparison

| Concept           | First Workflow       | Second Workflow               |   |
| ----------------- | -------------------- | ----------------------------- | - |
| Trigger           | `workflow_dispatch`  | `workflow_dispatch`           |   |
| Job               | `first-job`          | `second-job`                  |   |
| Runner            | `ubuntu-latest`      | `ubuntu-latest`               |   |
| Number of Steps   | 2                    | 1                             |   |
| Commands          | One command per step | Multiple commands in one step |   |
| Multi-line script | No                   | Yes, using `                  | ` |

---

# Important YAML Concept: Indentation

YAML is sensitive to indentation.

For example:

```yaml
jobs:
  first-job:
    runs-on: ubuntu-latest

    steps:
      - name: hello from Kirtan
        run: echo "Hello Kirtan"
```

The indentation tells GitHub the relationship between the configuration values.

```text
jobs
│
└── first-job
    │
    ├── runs-on
    │
    └── steps
        │
        ├── step 1
        │   ├── name
        │   └── run
        │
        └── step 2
            ├── name
            └── run
```

For this reason, YAML errors can happen even when the commands themselves are correct.

For example:

```yaml
steps:
  - name: hello
  run: echo "Hello"
```

This indentation is incorrect because `run` should belong to the step.

The correct version is:

```yaml
steps:
  - name: hello
    run: echo "Hello"
```

The structure is:

```text
Step
├── name
└── run
```

---

# What I Learned From These Two Workflows

These two simple workflows demonstrate the basic architecture of GitHub Actions.

```text
Repository
    │
    ▼
Workflow YAML File
    │
    ▼
Workflow
    │
    ├── Has a trigger
    │
    └── Contains jobs
            │
            ▼
           Job
            │
            ├── Runs on a runner
            │
            └── Contains steps
                    │
                    ├── Run commands
                    ├── Run scripts
                    └── Use reusable Actions
```

My workflows specifically use:

```text
Trigger
   ↓
workflow_dispatch
   ↓
Manual execution
   ↓
Job
   ↓
ubuntu-latest runner
   ↓
Steps
   ↓
Shell commands using run
```

---

# Key Takeaways

## 1. A Workflow Is the Complete Automation Process

```yaml
name: my first workflow
```

The workflow contains the complete automation configuration.

---

## 2. `workflow_dispatch` Allows Manual Execution

```yaml
on:
  workflow_dispatch:
```

The workflow waits for me to manually start it from GitHub.

---

## 3. A Workflow Contains Jobs

```yaml
jobs:
  first-job:
```

A job represents a major unit of work.

---

## 4. A Job Runs on a Runner

```yaml
runs-on: ubuntu-latest
```

The runner is the execution environment where the job runs.

---

## 5. A Job Contains Steps

```yaml
steps:
```

Steps are the individual tasks performed by the job.

---

## 6. A Step Can Run a Command

```yaml
run: echo "Hello Kirtan"
```

The command is executed on the runner.

---

## 7. One Step Can Run Multiple Commands

```yaml
run: |
  echo "Hello"
  echo "Goodbye"
```

The `|` creates a multi-line command block.

---

# Final Mental Model

When I manually run one of my workflows, the process is:

```text
I click "Run workflow"
          │
          ▼
workflow_dispatch event occurs
          │
          ▼
GitHub starts the workflow
          │
          ▼
GitHub starts the job
          │
          ▼
An Ubuntu runner executes the job
          │
          ▼
Steps execute in sequence
          │
          ▼
Each step runs commands or Actions
          │
          ▼
Job completes
          │
          ▼
Workflow completes
```

The fundamental hierarchy is:

```text
REPOSITORY
    │
    ▼
WORKFLOW
    │
    ▼
JOB
    │
    ▼
RUNNER
    │
    ▼
STEPS
    │
    ├── Shell Commands
    ├── Scripts
    └── Actions
```

These two workflows are my first practical examples of GitHub Actions and demonstrate how a manually triggered workflow executes jobs and steps on a GitHub-hosted Ubuntu runner.
