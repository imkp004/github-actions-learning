# GitHub Actions: Jobs, Artifacts, and Outputs

As GitHub Actions workflows become more complex, a workflow will often contain multiple jobs that perform different tasks, such as testing, building, and deploying an application.

Because each job runs in its own execution environment, jobs are isolated from one another. This creates an important question:

**How can one job share files or information with another job?**

GitHub Actions provides two important mechanisms for solving this problem:

* **Artifacts** — used to store and share files and directories between jobs or preserve files after a workflow run.
* **Outputs** — used to pass values and information from one job to another.

For example, a CI/CD pipeline might look like:

```text
Test
  ↓
Build
  ↓
Artifact + Outputs
  ↓
Deploy
```

The build job could create a compiled application and upload it as an **artifact**, while also generating information such as a version number or Docker image tag as a **job output**.

The deploy job can then download the artifact and use the output values to perform the deployment.

The basic mental model is:

```text
Artifacts → Files

Outputs   → Values / Information
```

Understanding artifacts and outputs is important for building multi-job GitHub Actions workflows where jobs need to **share files, pass information, and work together as part of a complete CI/CD pipeline**.
