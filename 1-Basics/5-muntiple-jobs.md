# GitHub Actions: Multiple Jobs and Parallel Execution

I expanded my CI workflow to contain two jobs:

1. `test-job` — installs the application and runs tests.
2. `deploy` — installs the application, runs tests, builds the project, and deploys it.

My workflow is:

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

  deploy:
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

      - name: build project
        run: npm run build

      - name: deploy
        run: echo "deploying ..."
```

---

# 1. A Workflow Can Have Multiple Jobs

Previously, I had:

```yaml
jobs:
  test-job:
```

Now I have:

```yaml
jobs:
  test-job:
    ...

  deploy:
    ...
```

This means the workflow contains **two separate jobs**.

The structure is:

```text
Workflow
│
├── test-job
│
└── deploy
```

Each job is an independent unit of work.

This is different from having multiple steps.

```text
ONE JOB
│
├── Step 1
├── Step 2
├── Step 3
└── Step 4
```

versus:

```text
MULTIPLE JOBS
│
├── Job 1
│   ├── Step 1
│   ├── Step 2
│   └── Step 3
│
└── Job 2
    ├── Step 1
    ├── Step 2
    └── Step 3
```

This distinction is extremely important.

---

# 2. Each Job Gets Its Own Runner

Both of my jobs contain:

```yaml
runs-on: ubuntu-latest
```

That means GitHub needs an Ubuntu runner for each job.

Conceptually:

```text
GitHub Actions
       │
       ├───────────────┐
       ▼               ▼
   test-job          deploy
       │               │
       ▼               ▼
 Ubuntu Runner      Ubuntu Runner
```

These are separate job execution environments.

I should think of them as:

```text
Runner A → test-job

Runner B → deploy
```

rather than:

```text
One Runner
   ↓
test-job
   ↓
deploy
```

---

# 3. Jobs Are Isolated

This is one of the most important concepts.

Suppose `test-job` creates:

```text
test-job runner
│
├── source code
├── node_modules/
└── test-results/
```

When `test-job` finishes, I cannot assume that `deploy` can see those files.

Why?

Because `deploy` has its own runner.

Conceptually:

```text
test-job
    │
    ▼
Runner A
    │
    ├── source code
    ├── node_modules
    └── build files
    │
    ▼
Job finishes


deploy
    │
    ▼
Runner B
    │
    └── Does NOT automatically contain
        Runner A's files
```

This explains why I have to run:

```yaml
- name: getting the code
  uses: actions/checkout@v3
```

again inside `deploy`.

The deploy job needs its own copy of the repository.

---

# 4. Why Do I Have to Install Node Again?

I already configured Node in `test-job`:

```yaml
- name: installing node
  uses: actions/setup-node@v3
  with:
    node-version: 18
```

But I have to do it again in `deploy`.

Why?

Because the jobs are isolated.

```text
test-job
   ↓
Runner A
   ↓
Node.js configured
```

doesn't mean:

```text
deploy
   ↓
Runner B
   ↓
Node.js automatically configured
```

Therefore:

```yaml
deploy:
  runs-on: ubuntu-latest

  steps:
    - uses: actions/checkout@v3

    - uses: actions/setup-node@v3
      with:
        node-version: 18
```

is necessary.

The same applies to dependencies.

---

# 5. Why Does `deploy` Run `npm ci` Again?

The test job runs:

```yaml
run: npm ci
```

which creates:

```text
node_modules/
```

But those installed dependencies belong to the environment of `test-job`.

The `deploy` job has another runner.

Therefore:

```text
test-job
   ↓
Runner A
   ↓
npm ci
   ↓
node_modules/


deploy
   ↓
Runner B
   ↓
No node_modules yet
   ↓
npm ci
   ↓
node_modules/
```

This is why your deploy job needs:

```yaml
- name: installing dependencies
  run: npm ci
```

---

# 6. The Jobs Run in Parallel by Default

This is another major concept.

Your workflow does **not** automatically mean:

```text
test-job
   ↓
deploy
```

Instead, because there is no dependency between them, GitHub can start both jobs independently.

Conceptually:

```text
                 Workflow
                    │
             ┌──────┴──────┐
             ▼             ▼
         test-job        deploy
             │             │
             ▼             ▼
          Runner A       Runner B
             │             │
             ▼             ▼
           Tests        Tests
             │             │
             │             ▼
             │           Build
             │             │
             │             ▼
             │           Deploy
             ▼
          Complete
```

The important idea is:

> **Jobs without dependencies can run independently and therefore can run in parallel.**

They don't necessarily start at the exact same millisecond, but GitHub is free to execute them concurrently.

---

# 7. Why Parallel Jobs Are Useful

Imagine a workflow with:

```text
Job 1 → Unit Tests       2 minutes
Job 2 → Security Scan    3 minutes
Job 3 → Lint             1 minute
```

If they run sequentially:

```text
Unit Tests
   ↓
2 min

Security Scan
   ↓
3 min

Lint
   ↓
1 min

Total ≈ 6 min
```

If they can run in parallel:

```text
             ┌── Unit Tests ────── 2 min
             │
Start ───────┼── Security Scan ─── 3 min
             │
             └── Lint ──────────── 1 min
                         │
                         ▼
                    All complete
```

The total time can be close to the longest job:

```text
≈ 3 minutes
```

rather than:

```text
≈ 6 minutes
```

This is one of the reasons GitHub Actions supports multiple jobs.

---

# 8. But There Is a Problem With My Current Workflow

There is an important problem in my current configuration.

I have:

```text
test-job
   │
   └── npm test

deploy
   │
   ├── npm test
   ├── npm run build
   └── deploy
```

Because the jobs are independent, the following situation is possible:

```text
test-job
    │
    └── Tests fail ❌


deploy
    │
    ├── Tests pass ✅
    ├── Build ✅
    └── Deploy ✅
```

Or more importantly, if the test job fails, GitHub does not automatically know:

> "The deploy job must wait for test-job."

Why?

Because I haven't told GitHub that `deploy` depends on `test-job`.

---

# 9. `needs` Creates a Dependency Between Jobs

GitHub Actions provides:

```yaml
needs:
```

to establish a dependency.

For example:

```yaml
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

  deploy:
    needs: test-job
    runs-on: ubuntu-latest

    steps:
      - name: getting the code
        uses: actions/checkout@v3

      ...
```

Now the relationship is:

```text
test-job
    │
    │ must finish successfully
    ▼
deploy
```

Instead of:

```text
test-job ──────┐
               │
               ├── independent
               │
deploy ────────┘
```

we now have:

```text
test-job
    │
    ▼
deploy
```

---

# 10. What Happens When `needs` Is Used?

Consider:

```yaml
deploy:
  needs: test-job
```

GitHub understands:

> The deploy job depends on the successful completion of `test-job`.

Execution becomes:

```text
Workflow starts
      │
      ▼
  test-job
      │
      ▼
   Tests
      │
      ├────────── FAIL ❌
      │             │
      │             ▼
      │        deploy skipped
      │
      └────────── PASS ✅
                    │
                    ▼
                  deploy
                    │
                    ▼
                  Build
                    │
                    ▼
                 Deploy
```

This is much closer to what we normally want from CI/CD.

---

# 11. Parallel vs Sequential Jobs

Without `needs`:

```yaml
jobs:
  test:
    ...

  deploy:
    ...
```

Conceptually:

```text
       Workflow
       /       \
      ▼         ▼
    test      deploy
```

The jobs are independent.

With:

```yaml
jobs:
  test:
    ...

  deploy:
    needs: test
    ...
```

the relationship becomes:

```text
test
  │
  ▼
deploy
```

So `needs` is how I define the **dependency graph** between jobs.

---

# 12. Jobs Form a Dependency Graph

This concept becomes very powerful when there are many jobs.

For example:

```yaml
jobs:
  test:
    ...

  security:
    ...

  build:
    needs: [test, security]
    ...

  deploy:
    needs: build
    ...
```

The workflow becomes:

```text
             ┌── test ───────┐
             │               │
             │               ▼
Start ───────┤             build
             │               │
             │               ▼
             └── security ─ deploy
```

Here:

```text
test
  \
   → build → deploy
  /
security
```

`test` and `security` can run in parallel.

But `build` waits for both.

Then `deploy` waits for `build`.

This is how complex CI/CD pipelines are constructed.

---

# 13. Example: Real CI/CD Pipeline

A common architecture could be:

```text
                    Push
                      │
                      ▼
              ┌──────────────┐
              │   Checkout   │
              └──────────────┘
                      │
             ┌────────┴────────┐
             ▼                 ▼
        Unit Tests       Security Scan
             │                 │
             └────────┬────────┘
                      ▼
                    Build
                      │
                      ▼
                 Docker Image
                      │
                      ▼
               Push to Registry
                      │
                      ▼
                   Deploy
```

The corresponding jobs might be:

```yaml
jobs:

  test:
    ...

  security:
    ...

  build:
    needs: [test, security]
    ...

  deploy:
    needs: build
    ...
```

This creates a controlled pipeline.

---

# 14. Important: Jobs vs Steps

I need to keep this distinction very clear.

## Steps

Steps belong to a job:

```yaml
job:
  steps:
    - ...
    - ...
    - ...
```

They normally execute sequentially.

```text
Step 1
  ↓
Step 2
  ↓
Step 3
```

## Jobs

Jobs belong to a workflow:

```yaml
jobs:
  test:
    ...

  build:
    ...

  deploy:
    ...
```

Jobs can run in parallel unless dependencies are defined.

```text
Job 1 ──────────┐
                │
Job 2 ──────────┼── parallel
                │
Job 3 ──────────┘
```

or can be chained:

```text
Job 1
  ↓
Job 2
  ↓
Job 3
```

using `needs`.

---

# 15. My Current Workflow Has Duplicate Work

There is another thing I should notice about my workflow.

Both jobs perform:

```text
checkout
setup Node
npm ci
npm test
```

So I'm doing the testing twice:

```text
test-job
   ↓
npm test

deploy
   ↓
npm test
```

This is not necessarily wrong, but it may be unnecessary.

A cleaner architecture could be:

```yaml
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

  deploy:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 18

      - run: npm ci
      - run: npm run build
      - run: echo "deploying ..."
```

Now the responsibility is clearer:

```text
test job
    ↓
Validate application
    ↓
PASS
    ↓
deploy job
    ↓
Build application
    ↓
Deploy
```

---

# 16. Why Doesn't `deploy` Automatically Receive the Result of `test`?

Another important concept:

`needs` controls **execution dependency**, but it does not automatically copy the entire filesystem from one job to another.

For example:

```yaml
deploy:
  needs: test
```

does NOT mean:

```text
test runner files
       ↓
automatically copied
       ↓
deploy runner
```

It means:

> Don't start `deploy` until `test` has completed successfully.

If I need to transfer files between jobs, I need something such as **artifacts**.

---

# 17. Example Using Artifacts

Suppose the build job creates:

```text
dist/
├── index.html
├── app.js
└── styles.css
```

I could upload the build output as an artifact.

Conceptually:

```text
Build Job
   │
   ├── npm run build
   │
   └── dist/
          │
          ▼
      Upload Artifact
          │
          ▼
      GitHub storage
          │
          ▼
      Deploy Job
          │
          ▼
      Download Artifact
          │
          ▼
      Deploy dist/
```

This is different from `needs`.

Think:

```text
needs
  =
execution dependency
```

while:

```text
artifact
  =
file/data transfer between jobs
```

---

# 18. A Practical CI/CD Example

Imagine I have a Node.js application.

I want:

1. Test the application.
2. Build it.
3. Deploy it.

I could design the workflow as:

```text
              Push
                │
                ▼
              Test
                │
             PASS?
             /   \
           NO     YES
           │       │
         STOP    Build
                   │
                   ▼
                 Deploy
```

The jobs:

```yaml
jobs:

  test:
    ...

  build:
    needs: test
    ...

  deploy:
    needs: build
    ...
```

This gives me a controlled pipeline:

```text
test
 ↓
build
 ↓
deploy
```

If testing fails:

```text
test ❌
 ↓
build skipped
 ↓
deploy skipped
```

That is an important CI/CD safety mechanism.

---

# 19. Multiple Independent Jobs

Sometimes I don't want jobs to depend on each other.

For example:

```yaml
jobs:

  unit-tests:
    ...

  lint:
    ...

  security-scan:
    ...
```

These can run independently:

```text
             ┌── Unit Tests
             │
Start ───────┼── Lint
             │
             └── Security Scan
```

This is useful when the jobs don't need each other's output.

---

# 20. Multiple Jobs With Dependencies

I can combine parallel and sequential execution.

For example:

```text
                  Start
                    │
             ┌──────┴──────┐
             ▼             ▼
           Test          Security
             │             │
             └──────┬──────┘
                    ▼
                  Build
                    │
                    ▼
                 Deploy
```

This is often the architecture I want in a real CI/CD pipeline.

The first two jobs can run concurrently, saving time.

Then:

```text
Build
```

waits for both.

Finally:

```text
Deploy
```

waits for Build.

---

# 21. My Mental Model Going Forward

When I see:

```yaml
jobs:
  test:
    ...

  deploy:
    ...
```

I should think:

```text
Two separate jobs
      │
      ├── Each gets its own runner
      │
      ├── Each has its own steps
      │
      ├── Each has its own execution environment
      │
      └── Can run independently
```

When I see:

```yaml
deploy:
  needs: test
```

I should think:

```text
deploy depends on test
       ↓
test must succeed
       ↓
deploy can start
```

When I see:

```yaml
run:
```

I should think:

```text
Execute a shell command/script
```

When I see:

```yaml
uses:
```

I should think:

```text
Use a reusable GitHub Action
```

When I see:

```yaml
with:
```

I should think:

```text
Pass configuration/input to the Action
```

---

# Key New Concepts From This Workflow

| Concept        | Meaning                                                   |
| -------------- | --------------------------------------------------------- |
| Multiple jobs  | A workflow can contain multiple independent units of work |
| Job isolation  | Each job executes in its own runner environment           |
| Parallel jobs  | Independent jobs can execute concurrently                 |
| `needs`        | Creates a dependency between jobs                         |
| Job dependency | A job waits for another job to finish successfully        |
| Artifacts      | Used to transfer files between jobs                       |
| Steps vs jobs  | Steps belong to jobs; jobs belong to workflows            |
| CI pipeline    | Validate code before progressing                          |
| CD pipeline    | Build/release/deploy validated code                       |

---

# The Big Picture

My workflow has introduced an important evolution:

### Before

```text
Workflow
   ↓
One Job
   ↓
Steps
```

### Now

```text
Workflow
   │
   ├── Job 1
   │    └── Steps
   │
   └── Job 2
        └── Steps
```

And with dependencies:

```text
Workflow
   │
   ├── Test Job
   │      │
   │      ▼
   │   Build Job
   │      │
   │      ▼
   └── Deploy Job
```

The most important thing to remember is:

> **Jobs are independent execution units. Each job gets its own runner. Jobs without dependencies can run in parallel. If one job must wait for another, use `needs`. If files must move between jobs, use artifacts or another external storage mechanism.**

This is the foundation for designing more advanced GitHub Actions CI/CD pipelines.
