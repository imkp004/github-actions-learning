# GitHub Actions — Jobs: Artifacts, Outputs & Caching

## 1. Job Artifacts

An **artifact** is a file or collection of files produced during a workflow that I want to save and use later.

For example, a build job might produce:

```text
dist/
├── index.html
├── app.js
└── style.css
```

The `dist/` directory is a **build output**, which can be uploaded as an artifact.

### Why Artifacts?

Each GitHub Actions job normally runs on its own runner. Therefore, files created in one job are not automatically available in another job.

For example:

```text
Build Job
    │
    └── creates dist/
             │
             ▼
          Artifact
             │
             ▼
       Deploy Job
```

The artifact acts as a way to transfer files between jobs or preserve them after the workflow finishes.

### Uploading an Artifact

```yaml
- name: Upload artifacts
  uses: actions/upload-artifact@v7
  with:
    name: dist-files
    path: |
      dist
      package.json
```

Important parts:

```yaml
name: dist-files
```

This gives the artifact a name.

```yaml
path: |
  dist
  package.json
```

This specifies which files/directories should be uploaded.

### Downloading an Artifact

Another job can download it:

```yaml
- name: Get build artifact
  uses: actions/download-artifact@v8
  with:
    name: dist-files
```

The overall flow becomes:

```text
Build Job
    │
    ├── npm run build
    │
    ├── dist/
    │
    └── Upload Artifact
             │
             ▼
        GitHub Artifact
             │
             ▼
       Deploy Job
             │
             └── Download Artifact
```

### Artifact vs Cache

An artifact is generally something I **produced and want to save/share**.

Examples:

```text
dist/
build/
ZIP files
test reports
binaries
```

A cache is something I **want to reuse to make future workflow runs faster**.

Examples:

```text
npm dependencies
pip cache
Maven dependencies
Gradle dependencies
```

A simple mental model:

```text
Artifact → Save/share output

Cache → Speed up future work
```

---

# 2. Job Outputs

A **job output** is a value created by one job and made available to another job.

Unlike artifacts, outputs are generally used for **small pieces of information**, rather than files.

For example:

```text
Build Job
    │
    └── generates:
        filename = app-1.0.zip
                 │
                 ▼
             Job Output
                 │
                 ▼
            Deploy Job
```

Job outputs are useful for values such as:

```text
filename
version
Docker image tag
environment name
resource ID
deployment URL
```

---

## 3. Step Outputs → Job Outputs

A common pattern is:

```text
Step
 ↓
Step Output
 ↓
Job Output
 ↓
Another Job
```

Example:

```yaml
jobs:

  build:
    runs-on: ubuntu-latest

    outputs:
      filename: ${{ steps.create-file.outputs.filename }}

    steps:
      - name: Create filename
        id: create-file
        run: echo "filename=app.zip" >> "$GITHUB_OUTPUT"
```

The step creates an output:

```bash
echo "filename=app.zip" >> "$GITHUB_OUTPUT"
```

The step has an ID:

```yaml
id: create-file
```

Therefore the step output can be referenced as:

```yaml
${{ steps.create-file.outputs.filename }}
```

The job exposes that value:

```yaml
outputs:
  filename: ${{ steps.create-file.outputs.filename }}
```

Now another job can use it:

```yaml
deploy:
  needs: build

  steps:
    - run: echo "Deploying ${{ needs.build.outputs.filename }}"
```

The syntax for accessing another job's output is:

```text
needs.<job-id>.outputs.<output-name>
```

For example:

```text
needs.build.outputs.filename
```

---

# 4. Why `needs` Matters for Outputs

The receiving job should depend on the job producing the output:

```yaml
deploy:
  needs: build
```

This creates the relationship:

```text
build
  │
  │ produces output
  ▼
deploy
```

If the Build job fails:

```text
build ❌
   │
   ▼
deploy ⏭️
```

By default, the Deploy job will not run.

If Build succeeds:

```text
build ✅
   │
   ▼
deploy
```

This is also useful because the Deploy job can safely consume the output produced by Build.

---

# 5. Artifacts vs Job Outputs

This distinction is extremely important.

### Artifact

Use an artifact when I need to transfer or preserve **files**.

```text
Build
 ↓
dist/
 ↓
Artifact
 ↓
Deploy
```

### Job Output

Use a job output when I need to transfer a **value**.

```text
Build
 ↓
filename = app.zip
 ↓
Job Output
 ↓
Deploy
```

Therefore:

```text
Artifact → Files

Job Output → Values
```

They can also be used together:

```text
                     BUILD
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
           dist/           filename
              │                 │
              ▼                 ▼
          Artifact          Output
              │                 │
              └────────┬────────┘
                       ▼
                     DEPLOY
```

The Deploy job can download the actual build files using the artifact while using the job output to know information about those files.

---

# 6. Dependency Caching

**Caching** is used to speed up workflow execution by reusing data that is expensive to download or generate.

A very common example is project dependencies.

Without caching:

```text
Workflow Run #1
    ↓
npm ci
    ↓
Download dependencies

Workflow Run #2
    ↓
npm ci
    ↓
Download dependencies AGAIN

Workflow Run #3
    ↓
npm ci
    ↓
Download dependencies AGAIN
```

This can waste time when the dependencies haven't changed.

With caching:

```text
First Run
    ↓
Download dependencies
    ↓
Save cache

Future Run
    ↓
Restore cache
    ↓
npm ci
    ↓
Reuse cached package data
    ↓
Faster
```

---

# 7. Caching With `setup-node`

For Node.js projects, `actions/setup-node` provides an easy way to enable npm caching.

```yaml
- name: Setup Node
  uses: actions/setup-node@v4
  with:
    node-version: 18
    cache: npm
```

Then I still install dependencies:

```yaml
- name: Install dependencies
  run: npm ci
```

The important point is:

> **Caching does not replace dependency installation.**

Instead, caching allows the package manager to reuse previously downloaded package data.

The workflow is:

```text
Setup Node
    ↓
Restore npm cache
    ↓
npm ci
    ↓
Run tests
```

---

# 8. Cache Hit and Cache Miss

The first time a workflow runs, there may be no matching cache.

This is a **cache miss**:

```text
Workflow
   ↓
Search cache
   ↓
No cache
   ↓
CACHE MISS
   ↓
Download dependencies
   ↓
Cache data
```

On a later run:

```text
Workflow
   ↓
Search cache
   ↓
Matching cache found
   ↓
CACHE HIT
   ↓
Restore cache
   ↓
npm ci
```

So:

```text
Cache MISS → Build the cache

Cache HIT → Reuse the cache
```

---

# 9. Cache Invalidation

A cache should not be used forever if the underlying dependencies change.

This is called **cache invalidation**.

For example:

```text
package-lock.json
        ↓
      Hash
        ↓
    Cache Key
```

If the dependency lock file changes:

```text
package-lock.json changes
        ↓
Different hash
        ↓
Different cache key
        ↓
Cache miss
        ↓
Download updated dependencies
        ↓
New cache
```

This prevents an old dependency cache from being treated as the current dependency set.

---

# 10. Manual Dependency Cache

I can also use `actions/cache` directly when I need more control.

```yaml
- name: Cache npm dependencies
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

- name: Install dependencies
  run: npm ci
```

Here:

```yaml
path: ~/.npm
```

specifies what should be cached.

And:

```yaml
key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

creates a cache key based on the operating system, Node environment, and dependency lock file.

---

# 11. `hashFiles()`

`hashFiles()` can generate a hash based on files in the repository.

For example:

```yaml
${{ hashFiles('**/package-lock.json') }}
```

Conceptually:

```text
package-lock.json
       ↓
   hashFiles()
       ↓
    ABC123
       ↓
Cache Key
```

If the lock file changes:

```text
package-lock.json
       ↓
   hashFiles()
       ↓
    XYZ789
       ↓
Different Cache Key
```

This is a common way to automatically invalidate dependency caches when dependencies change.

---

# 12. Caching Across Multiple Jobs

Each job gets its own runner:

```text
Test Job
    ↓
Runner A

Build Job
    ↓
Runner B
```

The filesystem isn't automatically shared between them.

However, both jobs can use the same dependency cache.

```text
                 GitHub Cache
                ┌─────────────┐
                │ npm cache   │
                └──────┬──────┘
                       │
              ┌────────┴────────┐
              ▼                 ▼
          Test Job           Build Job
              │                 │
            npm ci             npm ci
```

This allows each job to benefit from cached dependency data even though the jobs have separate runners.

---

# 13. Complete CI/CD Mental Model

At this point, I can think about these three concepts like this:

```text
                    CI/CD WORKFLOW
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
          Artifact      Output       Cache
             │            │            │
             │            │            │
          Files         Values       Reusable
             │            │          data
             │            │            │
             ▼            ▼            ▼
          dist/        version      npm cache
          build/       filename     pip cache
          reports/     image tag    Maven cache
```

### Artifact

```text
"What did my workflow produce?"
```

Example:

```text
dist/
app.zip
test-report.html
```

### Job Output

```text
"What information does another job need?"
```

Example:

```text
version=1.2.0
filename=app.zip
environment=production
```

### Cache

```text
"What expensive data can I reuse to make the workflow faster?"
```

Example:

```text
npm dependencies
pip dependencies
Maven dependencies
```

---

# 14. Quick Reference

| Feature               | Purpose                                 | Example                       |
| --------------------- | --------------------------------------- | ----------------------------- |
| **Artifact**          | Save/share files produced by a workflow | `dist/`                       |
| **Job Output**        | Pass values between jobs                | `version=1.2.0`               |
| **Cache**             | Speed up future workflow runs           | npm dependencies              |
| `upload-artifact`     | Upload files                            | Build → Artifact              |
| `download-artifact`   | Download files                          | Artifact → Deploy             |
| `$GITHUB_OUTPUT`      | Create a step output                    | `filename=app.zip`            |
| `needs`               | Create job dependency                   | `needs: build`                |
| `needs.<job>.outputs` | Access another job's output             | `needs.build.outputs.version` |
| `cache: npm`          | Enable npm caching                      | `setup-node`                  |
| `hashFiles()`         | Generate a hash for cache keys          | `package-lock.json`           |

## Final Mental Model

```text
                 GITHUB ACTIONS
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
     ARTIFACT        OUTPUT          CACHE
        │              │              │
        ▼              ▼              ▼
      FILES           VALUES      REUSABLE DATA
        │              │              │
        ▼              ▼              ▼
     dist.zip       version=1.0    npm packages
        │              │              │
        ▼              ▼              ▼
   Deploy Job      Deploy Job      Faster CI
```

The three concepts answer three different questions:

**Artifact:** *What files do I want to save or transfer?*

**Job Output:** *What value do I want another job to know?*

**Cache:** *What expensive data can I reuse to make future runs faster?*
