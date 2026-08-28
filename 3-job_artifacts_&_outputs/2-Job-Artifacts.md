# GitHub Actions: Job Artifacts

When working with multiple jobs in GitHub Actions, each job runs in its own runner and therefore has its own filesystem.

For example:

```text
Test Job
   │
   ▼
Build Job
   │
   ├── Creates build files
   │
   ▼
Deploy Job
```

The files created inside the **Build Job** are not automatically available inside the **Deploy Job** because the Deploy Job runs on a different runner.

GitHub Actions provides **artifacts** as a way to store and transfer files produced by a workflow.

An artifact can be thought of as:

> **A file or collection of files produced by a job that can be stored, downloaded, or used by another job.**

Examples of things that can be stored as artifacts include:

```text
dist/
build/
test-results/
coverage reports
logs
.zip files
compiled applications
binaries
deployment packages
```

---

# 1. Build Job Produces an Artifact

A common CI/CD pipeline looks like this:

```text
                 Test
                   │
                   ▼
                 Build
                   │
                   │ produces
                   ▼
              Build Artifact
                   │
                   ▼
                Deploy
```

For example, suppose I have a Node.js application.

The source code might look like:

```text
my-app/
├── src/
├── public/
├── package.json
├── package-lock.json
└── ...
```

When I run:

```bash
npm run build
```

the application might produce:

```text
dist/
├── index.html
├── app.js
├── styles.css
└── assets/
```

The `dist/` directory is the **build output**.

I can upload that directory as a GitHub Actions artifact.

---

# 2. Why Do I Need Artifacts?

Consider two jobs:

```yaml
jobs:

  build:
    runs-on: ubuntu-latest

    steps:
      - run: npm run build

  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - run: echo "Deploying..."
```

The `build` job creates:

```text
dist/
```

But the `deploy` job gets a **new runner**.

Conceptually:

```text
Build Runner
┌──────────────────────┐
│ source code          │
│ node_modules         │
│ dist/                │
└──────────────────────┘
          │
          │ Job finishes
          ▼
       Runner gone


Deploy Runner
┌──────────────────────┐
│ new environment      │
│                      │
│ dist/ does not exist │
└──────────────────────┘
```

Therefore, `needs: build` only creates a **dependency** between jobs. It does not automatically transfer the files.

I need an artifact:

```text
Build Job
    │
    │ upload
    ▼
GitHub Artifact Storage
    │
    │ download
    ▼
Deploy Job
```

---

# 3. Uploading an Artifact

GitHub provides an official action for uploading artifacts:

```yaml
uses: actions/upload-artifact@v4
```

GitHub's current documentation uses the `v4` artifact architecture for the standard upload examples.

A basic example is:

```yaml
- name: Upload artifact
  uses: actions/upload-artifact@v4
  with:
    name: dist-files
    path: dist
```

There are two important pieces here:

```yaml
name: dist-files
```

and:

```yaml
path: dist
```

---

# 4. `name`

```yaml
name: dist-files
```

This is the name I give to the artifact.

I can think of it as the artifact's identifier.

For example:

```text
Artifact:
    dist-files
```

Later, another job can request:

```yaml
name: dist-files
```

to download that specific artifact.

The flow becomes:

```text
Build Job
   │
   ▼
dist/
   │
   ▼
Upload
   │
   ▼
Artifact: dist-files
```

---

# 5. `path`

```yaml
path: dist
```

This tells GitHub Actions **what files or directories to upload**.

If my project contains:

```text
project/
├── src/
├── public/
├── dist/
│   ├── index.html
│   ├── app.js
│   └── styles.css
└── package.json
```

and I use:

```yaml
path: dist
```

GitHub uploads the contents of the `dist` directory as the artifact.

---

# 6. Uploading Multiple Paths

I can upload multiple files or directories.

For example:

```yaml
- name: Upload artifacts
  uses: actions/upload-artifact@v4
  with:
    name: dist-files
    path: |
      dist
      package.json
```

The `|` means that the value contains multiple lines.

So:

```yaml
path: |
  dist
  package.json
```

means:

```text
Upload:
├── dist/
└── package.json
```

The artifact could therefore contain:

```text
dist/
├── index.html
├── app.js
└── styles.css

package.json
```

This is useful when the deployment process needs both the compiled application and some configuration/metadata file.

---

# 7. Your Build Job

Your example is:

```yaml
build:
  needs: test
  runs-on: ubuntu-latest

  steps:
    - name: Get code
      uses: actions/checkout@v3

    - name: Install dependencies
      run: npm ci

    - name: Build website
      run: npm run build

    - name: upload artifacts
      uses: actions/upload-artifact@v7
      with:
        name: dist-files
        path: |
          dist
          package.json
```

The logic is:

```text
test
  │
  │ must succeed
  ▼
build
  │
  ├── Checkout code
  │
  ├── Install dependencies
  │
  ├── Build website
  │
  └── Upload artifact
          │
          ▼
       dist-files
```

There is one important version detail here.

Your example uses:

```yaml
actions/upload-artifact@v7
```

As of the current GitHub Action releases, `upload-artifact` has a `v7` release, so your version is valid for current GitHub.com workflows.

---

# 8. Understanding `needs: test`

Your build job starts with:

```yaml
needs: test
```

This means:

> The `build` job depends on the `test` job.

The execution becomes:

```text
test
 │
 ├── Success
 │
 ▼
build
```

If the tests fail:

```text
test ❌
 │
 ▼
build ⏭️
```

This prevents the build from happening when the required tests have failed.

This gives me a pipeline:

```text
Test
  │
  │ Success
  ▼
Build
  │
  │ Success
  ▼
Artifact
```

---

# 9. The Build Process

The build job first checks out the source code:

```yaml
- name: Get code
  uses: actions/checkout@v3
```

Then installs dependencies:

```yaml
- name: Install dependencies
  run: npm ci
```

Then builds the application:

```yaml
- name: Build website
  run: npm run build
```

Suppose:

```bash
npm run build
```

produces:

```text
dist/
├── index.html
├── app.js
├── styles.css
└── assets/
```

Then the next step:

```yaml
- name: Upload artifacts
  uses: actions/upload-artifact@v7
```

takes those files and stores them as:

```text
dist-files
```

---

# 10. Artifact Storage

The artifact is stored by GitHub for the workflow run.

Conceptually:

```text
Runner
  │
  │ upload
  ▼
GitHub
┌───────────────────────┐
│ Workflow Run           │
│                        │
│ Artifact: dist-files   │
│                        │
│ ├── dist/              │
│ └── package.json       │
└───────────────────────┘
```

The artifact is no longer dependent on the filesystem of the build runner.

This is what makes it possible for another job to retrieve it.

---

# 11. Downloading an Artifact

The next job can download the artifact.

The official action is:

```yaml
uses: actions/download-artifact@v4
```

Current GitHub documentation also shows the download action being used to retrieve artifacts created earlier in the workflow.

Your example has:

```yaml
uses: action/download-artifact@v8
```

There is a small typo here:

```text
action/download-artifact
```

should be:

```text
actions/download-artifact
```

Notice the **`s`** after `action`.

Also, the exact current major version can change over time; GitHub's current documentation shows `v5` examples for `download-artifact`, while the action repository has a v4 architecture.

For documentation, I would use:

```yaml
uses: actions/download-artifact@v5
```

unless your course specifically requires another version.

---

# 12. Your Deploy Job

Your deploy job is:

```yaml
deploy:
  needs: build
  runs-on: ubuntu-latest

  steps:
    - name: get the build artifact
      uses: actions/download-artifact@v5
      with:
        name: dist-files

    - name: output contents
      run: ls

    - name: Deploy
      run: echo "Deploying..."
```

The important part is:

```yaml
with:
  name: dist-files
```

This tells GitHub:

> Download the artifact named `dist-files`.

The flow is:

```text
Build Job
    │
    ▼
Upload
    │
    ▼
dist-files
    │
    │ stored by GitHub
    ▼
Deploy Job
    │
    ▼
Download dist-files
    │
    ▼
Files available on runner
```

---

# 13. `needs: build`

Your deploy job contains:

```yaml
needs: build
```

This creates another dependency.

The entire pipeline becomes:

```text
Test
  │
  │ needs
  ▼
Build
  │
  │ needs
  ▼
Deploy
```

More specifically:

```text
Test
  │
  └──► Build
          │
          └──► Deploy
```

This means:

```text
Test must succeed
      ↓
Build runs
      ↓
Build must succeed
      ↓
Deploy runs
```

If `test` fails:

```text
Test ❌
  ↓
Build skipped
  ↓
Deploy skipped
```

If `build` fails:

```text
Test ✅
  ↓
Build ❌
  ↓
Deploy skipped
```

This creates a safe CI/CD pipeline.

---

# 14. What Happens Inside the Deploy Runner?

When the deploy job starts, it gets a new runner.

Initially:

```text
Deploy Runner
├── Repository workspace
└── No build artifact
```

Then:

```yaml
- name: Get the build artifact
  uses: actions/download-artifact@v5
  with:
    name: dist-files
```

downloads the artifact.

Now the runner contains the artifact files.

Conceptually:

```text
Deploy Runner

├── dist/
│   ├── index.html
│   ├── app.js
│   └── styles.css
│
└── package.json
```

---

# 15. `ls`

Your next step is:

```yaml
- name: output contents
  run: ls
```

This is simply a debugging step.

It allows me to see what files exist in the current directory.

For example, I might see:

```text
dist
package.json
```

That confirms the artifact was downloaded.

I could inspect it further:

```yaml
- name: Output contents
  run: |
    ls -la
    ls -la dist
```

This could show:

```text
.
..
dist
package.json
```

and then:

```text
dist/
index.html
app.js
styles.css
```

---

# 16. Deploying the Artifact

Finally:

```yaml
- name: Deploy
  run: echo "Deploying..."
```

In your example, this only prints a message.

In a real project, this could be replaced by an actual deployment command.

For example:

```text
Artifact
   │
   ▼
Deploy Runner
   │
   ├── Download dist/
   │
   ├── Authenticate
   │
   └── Upload dist/ to hosting platform
```

The important CI/CD principle is:

> **Build once, store the build output as an artifact, and deploy that exact artifact.**

---

# 17. Complete Example

A more complete version of your workflow could look like:

```yaml
name: Build and Deploy

on:
  push:
    branches:
      - main

jobs:

  test:
    runs-on: ubuntu-latest

    steps:
      - name: Get code
        uses: actions/checkout@v6

      - name: Setup Node
        uses: actions/setup-node@v6
        with:
          node-version: 18

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test


  build:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - name: Get code
        uses: actions/checkout@v6

      - name: Setup Node
        uses: actions/setup-node@v6
        with:
          node-version: 18

      - name: Install dependencies
        run: npm ci

      - name: Build website
        run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v7
        with:
          name: dist-files
          path: |
            dist
            package.json


  deploy:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v5
        with:
          name: dist-files

      - name: Inspect artifact
        run: |
          ls -la
          ls -la dist

      - name: Deploy
        run: echo "Deploying..."
```

GitHub's current documentation demonstrates the same general pattern: build files are uploaded with `actions/upload-artifact`, and a later job uses `actions/download-artifact` to retrieve them.

---

# 18. Complete Execution Flow

Now I can visualize the entire pipeline:

```text
                    GitHub Push
                         │
                         ▼
                       TEST
                         │
                    npm test
                         │
                    ┌────┴────┐
                    │         │
                  FAIL      SUCCESS
                    │         │
                    ▼         ▼
                 STOP        BUILD
                              │
                         npm run build
                              │
                              ▼
                           dist/
                              │
                              ▼
                       Upload Artifact
                              │
                              ▼
                        dist-files
                              │
                              ▼
                           DEPLOY
                              │
                     Download Artifact
                              │
                              ▼
                         dist/ files
                              │
                              ▼
                           Deploy
```

This is a very common CI/CD pattern.

---

# 19. Artifacts Can Also Be Downloaded Manually

Artifacts are not only useful between jobs.

After a workflow runs, I can also access its artifacts through the GitHub UI.

Conceptually:

```text
GitHub Repository
      │
      ▼
Actions
      │
      ▼
Workflow Run
      │
      ▼
Artifacts
      │
      ▼
dist-files
      │
      ▼
Download
```

This can be useful when I want to:

* Download a build manually.
* Inspect test results.
* Download logs.
* Download compiled applications.
* Share build output.
* Debug a failed workflow.

---

# 20. Artifacts for Test Reports

Artifacts aren't only for deployment.

For example, a test job might generate:

```text
coverage/
├── index.html
├── report.html
└── coverage.json
```

I can upload it:

```yaml
- name: Upload coverage report
  uses: actions/upload-artifact@v7
  with:
    name: coverage-report
    path: coverage/
```

Now the workflow run contains:

```text
Artifacts
├── dist-files
└── coverage-report
```

This allows me to inspect test results after the workflow finishes.

GitHub's documentation specifically demonstrates uploading build artifacts and code coverage reports as separate artifacts.

---

# 21. Multiple Artifacts

A workflow can produce multiple artifacts.

For example:

```yaml
- name: Upload website
  uses: actions/upload-artifact@v7
  with:
    name: website
    path: dist/

- name: Upload test results
  uses: actions/upload-artifact@v7
  with:
    name: test-results
    path: test-results/

- name: Upload logs
  uses: actions/upload-artifact@v7
  with:
    name: logs
    path: logs/
```

The workflow could produce:

```text
Workflow Run
│
├── website
├── test-results
└── logs
```

Each artifact has its own name.

---

# 22. Artifact Retention

Artifacts are not necessarily stored forever.

I can configure how long an artifact should be retained.

Example:

```yaml
- name: Upload artifact
  uses: actions/upload-artifact@v7
  with:
    name: dist-files
    path: dist/
    retention-days: 5
```

This tells GitHub to retain the artifact for five days.

GitHub supports configuring artifact retention with `retention-days`, subject to the maximum retention policy configured for the repository, organization, or enterprise.

This is useful when artifacts are temporary.

For example:

```text
Build artifact
     ↓
Used for deployment
     ↓
Keep for 5 days
     ↓
Automatically expires
```

---

# 23. Artifact vs Repository Files

An artifact is **not the same thing as committing files to Git**.

For example:

```text
Git repository
│
├── src/
├── package.json
└── README.md
```

These are source-controlled files.

An artifact is workflow output:

```text
Workflow
   │
   ▼
Build
   │
   ▼
dist/
   │
   ▼
Artifact
```

So:

```text
Git repository
→ Source code


Artifact
→ Generated workflow output
```

I generally don't want to commit generated build output to my source repository if it can instead be produced by the CI pipeline.

---

# 24. Artifact vs Cache

Artifacts and caches are also different.

### Artifact

Used to:

```text
Store and share workflow output
```

Examples:

```text
dist/
test reports
compiled binaries
logs
deployment packages
```

### Cache

Used primarily to:

```text
Speed up future workflow runs
```

For example:

```text
npm dependencies
Docker layers
package manager cache
```

Mental model:

```text
Artifact
→ "Here is something my workflow produced."


Cache
→ "Here is something I want to reuse to make future runs faster."
```

They solve different problems.

---

# 25. Important Concept: Artifact ≠ Job Output

This distinction is extremely important before moving into the next section.

Suppose my build job creates:

```text
dist/
```

I should use an **artifact**:

```text
dist/ → Artifact
```

But suppose my build job determines:

```text
version = 1.4.2
```

That is better represented as a **job output**:

```text
version → Output
```

So:

```text
Files
  ↓
Artifacts


Values
  ↓
Outputs
```

Example:

```text
Build Job
│
├── dist/
│     ↓
│   Artifact
│
└── version=1.4.2
      ↓
    Output
```

The next topic, **Job Outputs**, will build on this distinction and show how one job can pass dynamically generated values to another job.

---

# Key Takeaways

The most important things I learned are:

```text
1. Each job gets its own runner/environment.

2. Files created in one job are not automatically available
   in another job.

3. Artifacts allow files to be stored and transferred.

4. actions/upload-artifact uploads files.

5. actions/download-artifact downloads files.

6. `name` identifies the artifact.

7. `path` specifies what files/directories are uploaded.

8. `needs` controls job dependencies but does NOT transfer files.

9. Artifacts can be downloaded by another job or manually
   from the GitHub Actions UI.

10. Artifacts can contain build files, test reports, logs,
    binaries, deployment packages, and other workflow outputs.

11. `retention-days` controls how long an artifact is retained.

12. Artifacts are for FILES; job outputs are for VALUES.
```

The core mental model is:

```text
             BUILD JOB
                 │
                 │
          Creates files
                 │
                 ▼
              dist/
                 │
                 │ upload
                 ▼
        ┌──────────────────┐
        │ GitHub Artifact  │
        │    dist-files    │
        └──────────────────┘
                 │
                 │ download
                 ▼
             DEPLOY JOB
                 │
                 ▼
          Use the files
                 │
                 ▼
             Deploy
```

> **Artifacts provide a way to take the output produced by one job, store it with the workflow run, and make it available for later use. This is especially important in CI/CD because it allows me to build an application once and then pass that exact build output to a later deployment job.**
