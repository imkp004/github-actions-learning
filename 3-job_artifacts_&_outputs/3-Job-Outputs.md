# GitHub Actions: Job Outputs

## 1. What Are Job Outputs?

**Job outputs** allow me to take a value generated inside one job and make that value available to another job.

They are typically used when I need to **reuse a value between different jobs**.

For example, suppose a job generates the name of a file:

```text
Build Job
    │
    ├── Generate file
    │
    └── filename = app-123.zip
              │
              ▼
          Job Output
              │
              ▼
        Deploy Job
              │
              └── Uses app-123.zip
```

The important idea is:

> **Job outputs allow jobs to communicate small pieces of information with each other.**

---

# 2. Why Do We Need Job Outputs?

Remember that each GitHub Actions job normally runs on its own runner.

For example:

```text
Job 1                         Job 2
┌───────────────┐            ┌───────────────┐
│ Build Runner  │            │ Deploy Runner │
│               │            │               │
│ filename = ? │            │               │
└───────────────┘            └───────────────┘
```

Job 2 cannot simply access variables that existed only inside Job 1.

For example:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: |
          filename="app.zip"
```

The variable:

```text
filename
```

belongs to that step's environment.

It does **not automatically become available** to another job.

To pass it to another job, I need to deliberately create an output.

---

# 3. Basic Job Output Flow

The overall process is:

```text
Step
  │
  │ creates a value
  ▼
Step Output
  │
  │ exposed by the job
  ▼
Job Output
  │
  │ accessed by another job
  ▼
Another Job
```

There are therefore two important concepts:

```text
Step Output
     ↓
Job Output
```

A step can create an output, and the job can expose that step output as a **job output**.

---

# 4. Simple Example

Suppose I want one job to generate a filename and another job to use it.

```yaml
name: Job Outputs Demo

on:
  workflow_dispatch:

jobs:

  generate:
    runs-on: ubuntu-latest

    outputs:
      filename: ${{ steps.create-file.outputs.filename }}

    steps:
      - name: Create filename
        id: create-file
        run: echo "filename=app.zip" >> "$GITHUB_OUTPUT"


  deploy:
    needs: generate
    runs-on: ubuntu-latest

    steps:
      - name: Use filename
        run: echo "Deploying ${{ needs.generate.outputs.filename }}"
```

The output flow is:

```text
generate job
     │
     ▼
create-file step
     │
     │ filename=app.zip
     ▼
Step Output
     │
     ▼
Job Output
     │
     ▼
deploy job
     │
     ▼
needs.generate.outputs.filename
```

The Deploy job would output:

```text
Deploying app.zip
```

---

# 5. Understanding `GITHUB_OUTPUT`

This line is extremely important:

```bash
echo "filename=app.zip" >> "$GITHUB_OUTPUT"
```

`GITHUB_OUTPUT` is a special file provided by GitHub Actions.

When I write:

```bash
echo "filename=app.zip" >> "$GITHUB_OUTPUT"
```

I am telling GitHub Actions:

> Create an output named `filename` whose value is `app.zip`.

So:

```text
filename=app.zip
```

becomes a step output.

---

# 6. Why Do We Give the Step an `id`?

Look at:

```yaml
- name: Create filename
  id: create-file
  run: echo "filename=app.zip" >> "$GITHUB_OUTPUT"
```

The important part is:

```yaml
id: create-file
```

The `id` gives the step a name that can be referenced later.

Then I can access its output using:

```text
steps.create-file.outputs.filename
```

Breaking that down:

```text
steps
  │
  └── create-file
        │
        └── outputs
              │
              └── filename
```

Therefore:

```yaml
${{ steps.create-file.outputs.filename }}
```

means:

> Get the `filename` output from the `create-file` step.

---

# 7. Creating the Job Output

Now look at the job:

```yaml
generate:
  runs-on: ubuntu-latest

  outputs:
    filename: ${{ steps.create-file.outputs.filename }}
```

This is where the step output becomes a **job output**.

The relationship is:

```text
Step Output
────────────────────────────
steps.create-file.outputs.filename
              │
              ▼
Job Output
────────────────────────────
generate.outputs.filename
```

The job output is essentially saying:

> Take the `filename` output from my `create-file` step and expose it as my job output called `filename`.

---

# 8. Using the Job Output in Another Job

Now we have:

```yaml
deploy:
  needs: generate
```

This is important.

The Deploy job needs the Generate job.

Then:

```yaml
run: echo "Deploying ${{ needs.generate.outputs.filename }}"
```

The syntax is:

```text
needs.<job-id>.outputs.<output-name>
```

In our example:

```text
needs.generate.outputs.filename
```

Breaking it down:

```text
needs
 │
 └── generate
       │
       └── outputs
             │
             └── filename
```

So:

```yaml
${{ needs.generate.outputs.filename }}
```

means:

> Get the `filename` output produced by the `generate` job.

---

# 9. Why `needs` Is Required

The receiving job needs to depend on the job that produces the output.

For example:

```yaml
deploy:
  needs: generate
```

Without this dependency, GitHub does not have the same job-output relationship available.

The flow becomes:

```text
generate
   │
   │ produces output
   ▼
filename = app.zip
   │
   ▼
deploy
```

If `generate` fails:

```text
generate ❌
    │
    ▼
deploy ⏭️
```

If `generate` succeeds:

```text
generate ✅
    │
    ▼
deploy
```

---

# 10. Example: Generate a File Name Dynamically

The previous example used a hardcoded value:

```bash
echo "filename=app.zip" >> "$GITHUB_OUTPUT"
```

But the real power of outputs is that the value can be generated dynamically.

For example:

```yaml
jobs:

  build:
    runs-on: ubuntu-latest

    outputs:
      filename: ${{ steps.build.outputs.filename }}

    steps:
      - name: Build application
        id: build
        run: |
          VERSION="1.5.2"
          FILE="myapp-${VERSION}.zip"

          echo "Creating $FILE"
          touch "$FILE"

          echo "filename=$FILE" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Deploy
        run: |
          echo "Deploying ${{ needs.build.outputs.filename }}"
```

The Build job dynamically generates:

```text
myapp-1.5.2.zip
```

and exposes it as an output.

The Deploy job receives:

```text
myapp-1.5.2.zip
```

---

# 11. Outputs Can Be Generated From Commands

The value does not have to be manually written.

For example:

```yaml
- name: Get Git commit
  id: commit
  run: |
    SHA=$(git rev-parse --short HEAD)
    echo "sha=$SHA" >> "$GITHUB_OUTPUT"
```

Suppose the Git commit is:

```text
a82f91c
```

The step creates:

```text
sha=a82f91c
```

Then the job can expose it:

```yaml
outputs:
  commit_sha: ${{ steps.commit.outputs.sha }}
```

Another job can use:

```yaml
run: echo "Commit: ${{ needs.build.outputs.commit_sha }}"
```

This is very useful because the value is generated dynamically.

---

# 12. Example: Docker Image Tag

A very common real-world use case is passing a Docker image tag between jobs.

Imagine:

```text
Build Job
   │
   ├── Build Docker image
   │
   └── Determine image tag
            │
            ▼
       image_tag=abc123
            │
            ▼
       Job Output
            │
            ▼
      Deploy Job
            │
            └── Deploy image:abc123
```

Example:

```yaml
jobs:

  build:
    runs-on: ubuntu-latest

    outputs:
      image_tag: ${{ steps.tag.outputs.image_tag }}

    steps:
      - name: Generate image tag
        id: tag
        run: |
          TAG=$(git rev-parse --short HEAD)
          echo "image_tag=$TAG" >> "$GITHUB_OUTPUT"


  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Deploy Docker image
        run: |
          echo "Deploying image:"
          echo "myapp:${{ needs.build.outputs.image_tag }}"
```

If the commit SHA is:

```text
a82f91c
```

the Deploy job gets:

```text
myapp:a82f91c
```

---

# 13. Example: Environment Decision

Outputs can also be used for decisions.

Suppose I want to determine whether a deployment should go to production.

```yaml
jobs:

  determine:
    runs-on: ubuntu-latest

    outputs:
      environment: ${{ steps.check.outputs.environment }}

    steps:
      - name: Determine environment
        id: check
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "environment=production" >> "$GITHUB_OUTPUT"
          else
            echo "environment=development" >> "$GITHUB_OUTPUT"
          fi


  deploy:
    needs: determine
    runs-on: ubuntu-latest

    steps:
      - name: Show environment
        run: |
          echo "Deploying to ${{ needs.determine.outputs.environment }}"
```

The output could be:

```text
main branch
    ↓
production
```

or:

```text
development branch
    ↓
development
```

The second job doesn't need to calculate the value again.

---

# 14. Multiple Job Outputs

A job can expose multiple outputs.

Example:

```yaml
jobs:

  build:
    runs-on: ubuntu-latest

    outputs:
      filename: ${{ steps.info.outputs.filename }}
      version: ${{ steps.info.outputs.version }}
      environment: ${{ steps.info.outputs.environment }}

    steps:
      - name: Generate information
        id: info
        run: |
          echo "filename=app.zip" >> "$GITHUB_OUTPUT"
          echo "version=1.5.2" >> "$GITHUB_OUTPUT"
          echo "environment=production" >> "$GITHUB_OUTPUT"
```

Now the job provides:

```text
build.outputs.filename
build.outputs.version
build.outputs.environment
```

Another job can use them:

```yaml
deploy:
  needs: build

  steps:
    - run: |
        echo "File: ${{ needs.build.outputs.filename }}"
        echo "Version: ${{ needs.build.outputs.version }}"
        echo "Environment: ${{ needs.build.outputs.environment }}"
```

Output:

```text
File: app.zip
Version: 1.5.2
Environment: production
```

---

# 15. Job Outputs vs Artifacts

This is one of the most important distinctions.

Suppose the Build job creates:

```text
app.zip
```

If I need the **actual file**, I use an artifact:

```text
app.zip
   ↓
Artifact
   ↓
Deploy Job
```

If I only need the **name of the file**, I can use an output:

```text
"app.zip"
   ↓
Job Output
   ↓
Deploy Job
```

Therefore:

```text
Artifact
    ↓
Transfers files


Job Output
    ↓
Transfers values
```

A real workflow may use both:

```text
                    BUILD
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
      app.zip                 filename
          │                       │
          ▼                       ▼
      Artifact                Output
          │                       │
          └───────────┬───────────┘
                      ▼
                    DEPLOY
```

The Deploy job downloads the artifact and uses the output to know what file to work with.

---

# 16. Important Syntax to Remember

There are three pieces of syntax to remember.

### Step output

```yaml
${{ steps.<step-id>.outputs.<output-name> }}
```

Example:

```yaml
${{ steps.create-file.outputs.filename }}
```

### Job output definition

```yaml
outputs:
  <output-name>: ${{ steps.<step-id>.outputs.<output-name> }}
```

Example:

```yaml
outputs:
  filename: ${{ steps.create-file.outputs.filename }}
```

### Accessing another job's output

```yaml
${{ needs.<job-id>.outputs.<output-name> }}
```

Example:

```yaml
${{ needs.build.outputs.filename }}
```

The complete relationship is:

```text
Step
 ↓
steps.create-file.outputs.filename
 ↓
Job
 ↓
build.outputs.filename
 ↓
Another Job
 ↓
needs.build.outputs.filename
```

---

# 17. Complete Example

Here is a realistic example combining testing, building, an artifact, and a job output:

```yaml
name: CI/CD

on:
  workflow_dispatch:

jobs:

  test:
    runs-on: ubuntu-latest

    steps:
      - name: Get code
        uses: actions/checkout@v6

      - name: Run tests
        run: echo "Running tests..."


  build:
    needs: test
    runs-on: ubuntu-latest

    outputs:
      filename: ${{ steps.build-info.outputs.filename }}

    steps:
      - name: Get code
        uses: actions/checkout@v6

      - name: Build application
        id: build-info
        run: |
          VERSION="1.0.0"
          FILE="myapp-${VERSION}.zip"

          echo "Building $FILE"
          touch "$FILE"

          echo "filename=$FILE" >> "$GITHUB_OUTPUT"

      - name: Upload artifact
        uses: actions/upload-artifact@v7
        with:
          name: application
          path: "*.zip"


  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Download artifact
        uses: actions/download-artifact@v5
        with:
          name: application

      - name: Deploy
        run: |
          echo "Deploying ${{ needs.build.outputs.filename }}"
          ls -la
```

The workflow now demonstrates both concepts:

```text
TEST
 │
 ▼
BUILD
 │
 ├── Creates myapp-1.0.0.zip
 │        │
 │        └── Artifact
 │
 └── filename=myapp-1.0.0.zip
          │
          └── Job Output
                 │
                 ▼
              DEPLOY
                 │
                 ├── Downloads Artifact
                 │
                 └── Uses Job Output
```

---

# Key Takeaway

The most important concept is:

> **A job output is a value produced by one job that can be reused by another job.**

The pattern to remember is:

```text
Step creates value
       ↓
Write value to GITHUB_OUTPUT
       ↓
Give step an ID
       ↓
Expose step output as a job output
       ↓
Use `needs` in the next job
       ↓
Access it with `needs.<job>.outputs.<output>`
```

For example:

```yaml
# Create step output
- id: create
  run: echo "filename=app.zip" >> "$GITHUB_OUTPUT"

# Expose it as job output
outputs:
  filename: ${{ steps.create.outputs.filename }}

# Use it from another job
run: echo "${{ needs.build.outputs.filename }}"
```

### Mental Model

```text
                 JOB 1
                   │
                   ▼
             Generate value
                   │
                   ▼
             Step Output
                   │
                   ▼
              Job Output
                   │
                   ▼
             ───────────
              JOB 2
             ───────────
                   │
                   ▼
        needs.job1.outputs.value
```

**Artifacts move files. Job outputs move values.**
