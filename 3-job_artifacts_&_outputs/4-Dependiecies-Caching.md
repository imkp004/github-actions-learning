# GitHub Actions: Dependency Caching

## 1. What Is Caching?

In a GitHub Actions workflow, some steps can take a significant amount of time to execute repeatedly.

A common example is installing project dependencies.

For a Node.js project:

```bash
npm ci
```

might download hundreds of packages every time the workflow runs.

For example:

```text
Workflow Run #1
    ↓
npm ci
    ↓
Download dependencies
    ↓
Run tests
```

Then I push another commit:

```text
Workflow Run #2
    ↓
npm ci
    ↓
Download the same dependencies again
    ↓
Run tests
```

If the dependencies have not changed, downloading everything again is unnecessary work.

**Caching** allows GitHub Actions to save data from one workflow run and reuse it in future runs.

The basic idea is:

```text
First Run
   ↓
Download dependencies
   ↓
Save them in cache
   ↓
Future Run
   ↓
Restore cache
   ↓
Skip downloading unchanged dependencies
   ↓
Workflow runs faster
```

---

# 2. Why Is Dependency Caching Useful?

Imagine a project has many dependencies:

```text
package.json
    ↓
npm ci
    ↓
Download
    ├── React
    ├── Express
    ├── Axios
    ├── Jest
    ├── TypeScript
    └── hundreds of other packages
```

Downloading and installing all of these packages can take time.

If two workflow jobs repeatedly need the same dependencies:

```text
Test Job
    ↓
npm ci

Build Job
    ↓
npm ci
```

the same packages may be downloaded repeatedly.

Caching can reduce unnecessary downloads.

The goal is:

```text
Without cache:

Test  → Download dependencies
Build → Download dependencies
Deploy → Download dependencies


With cache:

Test  → Restore cache → Install/use dependencies
Build → Restore cache → Install/use dependencies
Deploy → Restore cache → Install/use dependencies
```

The exact benefit depends on the package manager, project size, cache hit rate, and workflow design.

---

# 3. Cache vs Artifact

Caching is easy to confuse with artifacts, but they serve different purposes.

### Artifact

An artifact stores something produced by the workflow.

```text
Build
  ↓
dist/
  ↓
Artifact
  ↓
Deploy
```

Examples:

```text
dist/
build/
test reports
logs
ZIP files
binaries
```

### Cache

A cache stores data that can be reused to make future workflow runs faster.

```text
npm dependencies
    ↓
Cache
    ↓
Future workflow runs
```

The mental model is:

```text
Artifact
→ "I produced this and want to save/share it."


Cache
→ "I already downloaded/generated this and want to reuse it."
```

---

# 4. A Simple Example Without Caching

Consider this workflow:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Get code
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 18

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

Every time the workflow runs:

```text
checkout
   ↓
setup Node
   ↓
npm ci
   ↓
download/install dependencies
   ↓
npm test
```

If I push another commit:

```text
checkout
   ↓
setup Node
   ↓
npm ci
   ↓
download/install dependencies AGAIN
   ↓
npm test
```

This is where caching can help.

---

# 5. Caching With `setup-node`

For Node.js projects, one of the easiest approaches is to use the caching functionality built into `actions/setup-node`.

Example:

```yaml
- name: Setup Node
  uses: actions/setup-node@v4
  with:
    node-version: 18
    cache: npm
```

Now the workflow can cache npm's package data.

A complete example:

```yaml
name: Node CI

on:
  push:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Get code
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

The important line is:

```yaml
cache: npm
```

This tells `setup-node` to cache npm's global package data.

---

# 6. Important: Cache Does Not Automatically Mean "Skip `npm ci`"

This is a very important beginner concept.

If I have:

```yaml
cache: npm
```

I still normally run:

```yaml
npm ci
```

The cache speeds up the process by avoiding unnecessary downloads from the package registry.

It does **not mean**:

```text
Cache exists
   ↓
Don't install dependencies
```

Instead, the process is closer to:

```text
Cache
  ↓
Restore npm package data
  ↓
npm ci
  ↓
npm can reuse cached packages
  ↓
Install dependencies
```

So caching and installing dependencies are two different operations.

---

# 7. Where Should the Cache Step Go?

The cache must be restored **before the expensive operation that benefits from it**.

For example:

```yaml
- name: Setup Node
  uses: actions/setup-node@v4
  with:
    node-version: 18
    cache: npm

- name: Install dependencies
  run: npm ci
```

The order is important:

```text
Setup Node + Restore Cache
            ↓
       npm ci
            ↓
       Run tests
```

Not:

```text
npm ci
   ↓
Restore cache
```

because the cache needs to be available before the dependency installation can benefit from it.

---

# 8. First Workflow Run: Cache Miss

The first time the workflow runs, there may not be a cache available.

This is called a **cache miss**.

The process is:

```text
Workflow Run #1
      ↓
Look for cache
      ↓
No matching cache
      ↓
Cache MISS
      ↓
npm ci
      ↓
Download dependencies
      ↓
Cache data can be saved
```

Conceptually:

```text
┌──────────────────────┐
│ GitHub Actions Cache │
│                      │
│ No matching cache    │
└──────────────────────┘
          │
          ▼
       MISS
          │
          ▼
    Download packages
```

The first run may therefore not be significantly faster.

---

# 9. Second Workflow Run: Cache Hit

Suppose I push another commit but don't change the dependencies.

The workflow runs again:

```text
Workflow Run #2
      ↓
Look for cache
      ↓
Matching cache found
      ↓
Cache HIT
      ↓
Restore cached package data
      ↓
npm ci
      ↓
Faster dependency installation
```

Conceptually:

```text
┌──────────────────────┐
│ GitHub Actions Cache │
│                      │
│ npm cache available  │
└──────────────────────┘
          │
          ▼
         HIT
          │
          ▼
   Restore cache
          │
          ▼
       npm ci
```

The important difference is:

```text
First run:
Cache MISS → download everything needed

Later run:
Cache HIT → reuse cached package data
```

---

# 10. How Does GitHub Know Which Cache to Use?

GitHub Actions uses a **cache key** to identify cached data.

You can think of a cache key as the cache's unique identity.

For dependencies, I don't want to use the same cache forever.

For example:

```text
package-lock.json
      ↓
Hash
      ↓
Cache Key
```

If my dependencies change, the cache key can change.

This allows GitHub Actions to recognize:

```text
Old dependencies
      ≠
New dependencies
```

and create/use an appropriate cache.

---

# 11. Why `package-lock.json` Is Important

For npm caching, the dependency lock file is important.

For example:

```text
package.json
package-lock.json
```

The `package-lock.json` file describes the exact dependency tree that npm should install.

If I change a dependency:

```text
package-lock.json changes
        ↓
Cache key changes
        ↓
Old cache is no longer the exact match
        ↓
New dependencies are downloaded
        ↓
New cache can be created
```

This is called **cache invalidation**.

---

# 12. Cache Invalidation

Cache invalidation means:

> Deciding when an existing cache should no longer be considered valid.

This is important because dependencies can change.

Suppose the project originally has:

```text
React 18
Express 4
Axios 1
```

and the cache contains those dependencies.

Then I change the project:

```text
React 19
Express 5
Axios 1
```

I don't want GitHub Actions blindly using a cache containing only the old dependency set.

Instead:

```text
Dependencies change
       ↓
Lock file changes
       ↓
Cache key changes
       ↓
Cache miss
       ↓
Download new dependency data
       ↓
New cache
```

---

# 13. Example of Cache Invalidation

Suppose my project initially has:

```text
package-lock.json
        ↓
Hash: ABC123
        ↓
Cache key: npm-ABC123
```

The cache is:

```text
npm-ABC123
```

Later I add a package.

Now:

```text
package-lock.json
        ↓
Hash: XYZ789
        ↓
Cache key: npm-XYZ789
```

The new workflow searches for:

```text
npm-XYZ789
```

but the existing cache is:

```text
npm-ABC123
```

Therefore:

```text
npm-XYZ789
      ↓
No matching cache
      ↓
Cache MISS
      ↓
Install/download new dependencies
      ↓
Save new cache
```

This prevents stale dependency data from being treated as the current cache.

---

# 14. Manual Caching with `actions/cache`

GitHub Actions also provides the `actions/cache` action when I need more control.

Example:

```yaml
- name: Cache npm dependencies
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

Then:

```yaml
- name: Install dependencies
  run: npm ci
```

The workflow becomes:

```text
Checkout
   ↓
Restore npm cache
   ↓
npm ci
   ↓
Run tests
```

---

# 15. Understanding `path`

In:

```yaml
with:
  path: ~/.npm
```

`path` specifies what directory should be cached.

For npm:

```text
~/.npm
```

is npm's cache directory.

So:

```yaml
path: ~/.npm
```

means:

> Save the contents of this directory in the GitHub Actions cache.

Other package managers have different cache locations.

---

# 16. Understanding `key`

Example:

```yaml
key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

This creates a cache key based on:

```text
Operating System
      +
Node
      +
package-lock.json hash
```

For example, conceptually:

```text
Linux + Node + ABC123
```

could produce something like:

```text
Linux-node-ABC123
```

If the lock file changes:

```text
Linux-node-XYZ789
```

the key changes.

That gives us automatic cache invalidation when dependencies change.

---

# 17. `hashFiles()`

This part:

```yaml
${{ hashFiles('**/package-lock.json') }}
```

calculates a hash based on matching files.

In this example:

```text
**/package-lock.json
```

means GitHub Actions looks for `package-lock.json` files in the repository.

The resulting hash can be used as part of the cache key.

The idea is:

```text
package-lock.json
        ↓
     hashFiles()
        ↓
   ABC123
        ↓
Cache key
```

If the file changes:

```text
package-lock.json
        ↓
     hashFiles()
        ↓
   XYZ789
        ↓
Different cache key
```

This is one of the most common ways to invalidate dependency caches.

---

# 18. Using `restore-keys`

`actions/cache` also supports restore keys.

Example:

```yaml
- name: Cache npm dependencies
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

The exact cache key is preferred first.

For example:

```text
Linux-node-ABC123
```

If that exact key isn't available, GitHub can look for a cache matching the restore-key prefix:

```text
Linux-node-
```

This can sometimes allow a partially matching older cache to be restored.

---

# 19. Dependency Caching Across Multiple Jobs

Suppose I have:

```text
Test Job
Build Job
```

Both jobs run:

```bash
npm ci
```

Each job has its own runner.

Therefore:

```text
Test Runner
    ↓
npm ci


Build Runner
    ↓
npm ci
```

They don't share the same filesystem.

But both jobs can use the same dependency cache.

For example:

```yaml
jobs:

  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: npm

      - run: npm ci
      - run: npm test


  build:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: npm

      - run: npm ci
      - run: npm run build
```

Conceptually:

```text
                 GitHub Cache
                 ┌───────────┐
                 │ npm cache │
                 └─────┬─────┘
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
         Test Job            Build Job
             │                   │
           npm ci              npm ci
```

The runners remain separate, but the reusable cache is shared through GitHub's cache service.

---

# 20. Cache Key Depends on More Than Dependencies

A cache should generally account for factors that affect compatibility.

For example:

```text
Operating System
Node version
Dependency lock file
```

A cache key might conceptually be:

```text
OS + Node version + dependency hash
```

For example:

```text
Linux-node18-ABC123
```

This helps prevent using an inappropriate cache across incompatible environments.

---

# 21. Caching Other Package Managers

The same concept applies beyond npm.

Examples include:

```text
npm
Yarn
pnpm
pip
Maven
Gradle
Bundler
Go modules
```

For example, Python workflows commonly cache pip dependencies:

```yaml
- name: Setup Python
  uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: pip
```

Then:

```yaml
- name: Install dependencies
  run: pip install -r requirements.txt
```

The concept is the same:

```text
Restore cache
     ↓
Install dependencies
     ↓
Use cached data where possible
```

---

# 22. When Should I Use Caching?

Caching is most useful when:

* A step takes a long time.
* The same data is downloaded repeatedly.
* The data doesn't change frequently.
* The data can safely be regenerated.
* Multiple workflow runs need the same dependencies.
* The dependency set changes less frequently than the source code.

A good example is:

```text
Source code changes frequently
        ↓
package-lock.json changes less frequently
        ↓
Dependencies can be cached
```

For example:

```text
Commit #1 → cache dependencies
Commit #2 → reuse cache
Commit #3 → reuse cache
Commit #4 → reuse cache
Commit #5 → package-lock.json changes
Commit #5 → new cache
```

---

# 23. When Should I NOT Use a Cache?

Caching isn't automatically beneficial for everything.

If something is:

```text
very small
fast to generate
changes every run
not reusable
```

there may be little benefit.

For example, caching a file that takes:

```text
0.1 seconds
```

to generate probably isn't worth the complexity.

But caching hundreds of megabytes of dependencies that take several minutes to download can provide a significant benefit.

The general principle is:

```text
Cost of generating/downloading
            >
Cost of restoring cache
```

Then caching may be worthwhile.

---

# 24. Cache Lifecycle

A useful mental model is:

```text
                 Workflow Run
                      │
                      ▼
               Look for cache
                      │
             ┌────────┴────────┐
             │                 │
          CACHE HIT         CACHE MISS
             │                 │
             ▼                 ▼
      Restore cached      Download data
          data                  │
             │                  ▼
             │            Create/update
             │               cache
             │                  │
             └────────┬─────────┘
                      ▼
              Continue workflow
```

This is the overall caching lifecycle.

---

# 25. Caching vs Reusing an Artifact

This is another important distinction.

Suppose I build a website:

```text
npm run build
      ↓
dist/
```

I want to deploy `dist/`.

That should generally be an **artifact**:

```text
dist/
  ↓
Artifact
  ↓
Deploy Job
```

But if I want to avoid repeatedly downloading npm packages:

```text
npm dependencies
  ↓
Cache
```

So:

```text
Build output
→ Artifact


Dependencies
→ Cache
```

This distinction is extremely important when designing CI/CD pipelines.

---

# 26. Practical Example: Test + Build

A practical Node.js workflow could look like:

```yaml
name: Node CI

on:
  push:
  pull_request:

jobs:

  test:
    runs-on: ubuntu-latest

    steps:
      - name: Get code
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test


  build:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - name: Get code
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build

      - name: Upload build
        uses: actions/upload-artifact@v7
        with:
          name: website
          path: dist/
```

Now I have both:

```text
                TEST
                  │
                  ▼
           Restore npm cache
                  │
                  ▼
               npm ci
                  │
                  ▼
              npm test
                  │
                  ▼
                BUILD
                  │
                  ▼
           Restore npm cache
                  │
                  ▼
               npm ci
                  │
                  ▼
            npm run build
                  │
                  ▼
             Upload Artifact
                  │
                  ▼
                dist/
```

Notice the two different mechanisms:

```text
npm cache
→ makes dependency installation faster


dist/
→ artifact containing the build output
```

---

# 27. The Most Important Concepts

### 1. Cache expensive, reusable data

```text
Dependencies
    ↓
Cache
```

### 2. Restore the cache before the operation that benefits from it

```text
Restore cache
     ↓
Install dependencies
```

### 3. The first run can be a cache miss

```text
No cache
   ↓
Download
   ↓
Cache
```

### 4. Later runs can be cache hits

```text
Cache exists
   ↓
Restore
   ↓
Faster workflow
```

### 5. Cache keys determine which cache is used

```text
Cache Key
    ↓
Identify cache
```

### 6. Lock-file changes can invalidate dependency caches

```text
package-lock.json changes
        ↓
Different hash
        ↓
Different cache key
        ↓
New cache
```

### 7. Cache is not the same as an artifact

```text
Cache
→ Speed up future workflows


Artifact
→ Store/share workflow output
```

---

# 28. Final Mental Model

The simplest way to understand GitHub Actions caching is:

```text
                EXPENSIVE OPERATION
                       │
                       ▼
               Download dependencies
                       │
                       ▼
                     Cache
                       │
                       │
        ┌──────────────┴──────────────┐
        ▼                             ▼
 Future Workflow                 Future Workflow
        │                             │
        ▼                             ▼
 Restore Cache                  Restore Cache
        │                             │
        ▼                             ▼
 Faster Install                  Faster Install
```

And when dependencies change:

```text
package-lock.json changes
          │
          ▼
     Cache key changes
          │
          ▼
      Cache miss
          │
          ▼
Download new dependencies
          │
          ▼
      New cache
```

The core idea is:

> **Caching allows GitHub Actions to reuse expensive, relatively stable data—especially dependencies—across workflow runs, reducing unnecessary downloads and speeding up CI/CD pipelines.**
