# Task 2: Simple Jenkins Pipeline for CI/CD

A small Flask app with a complete CI/CD pipeline built in **Jenkins** and **Docker**. Every commit pushed to GitHub is automatically checked out, built into a Docker image, tested with pytest, and deployed as a running container.

## Tools Used

| Tool | Purpose |
|---|---|
| Jenkins (running in Docker) | Automation server that runs the pipeline |
| Docker | Builds the app image and runs the deployed container |
| Git / GitHub | Source control and the pipeline's trigger source |
| Python / Flask | Sample application |
| pytest | Automated tests |

## Project Structure

```
.
├── app.py             # Flask application
├── test_app.py        # pytest test
├── requirements.txt   # Python dependencies
├── Dockerfile         # Image for the Flask app
├── Jenkinsfile        # Pipeline definition (Declarative)
├── .gitignore
├── README.md
└── screenshots/       # Pipeline and app screenshots
```

## Pipeline Stages

Defined in the `Jenkinsfile`:

1. **Checkout**: pulls the latest code from GitHub (`checkout scm`).
2. **Build**: builds the Docker image, tagged with the build number and `latest`.
3. **Test**: runs `pytest` inside the freshly built image. If a test fails, the pipeline stops and nothing is deployed.
4. **Deploy**: removes the old `flask-app` container and starts a new one from the new image on port 5000.

A `post` block prints a success or failure message at the end.

## Setup

### 1. Run Jenkins in Docker

The Jenkins image needs the Docker CLI so the pipeline can build images. `Dockerfile` for Jenkins:

```dockerfile
FROM jenkins/jenkins:lts-jdk17
USER root
RUN apt-get update && apt-get install -y docker.io git && rm -rf /var/lib/apt/lists/*
```

Build and run (PowerShell, one line):

```powershell
docker build -t my-jenkins .
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock -u root my-jenkins
```

Get the initial admin password and open http://localhost:8080:

```powershell
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Install the suggested plugins and create an admin user.

### 2. Create the pipeline job

1. **New Item** → name `flask-pipeline` → **Pipeline** → OK.
2. **Build Triggers** → tick **Poll SCM** → schedule `H/2 * * * *`.
3. **Pipeline** section:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/Rithik72/jenkins-cicd-pipeline.git`
   - Branch Specifier: `*/main`
   - Script Path: `Jenkinsfile`
4. **Save**, then **Build Now**.

### 3. View the app

After a successful build, open http://localhost:5000. It shows: `Hello from Jenkins CI/CD pipeline!`

## Trigger on Each Commit

The job uses **Poll SCM**: Jenkins checks the repository about every 2 minutes and starts a build when it finds a new commit.

A GitHub webhook would trigger instantly, but GitHub cannot reach Jenkins on `localhost`. A webhook would need a public URL (for example through a tunnel such as ngrok) or a Jenkins server hosted online. Polling gives the same result for this setup.

To test it, change the greeting in `app.py`, then:

```powershell
git add .
git commit -m "Update greeting"
git push
```

A new build starts on its own, and after it finishes the new text appears at http://localhost:5000.

## Screenshots

| Stage View (all green) | Console Output |
|---|---|
| ![Stage view](screenshots/stage-view.png) | ![Console output](screenshots/console-output.png) |

| App running on port 5000 | Build triggered by a commit |
|---|---|
| ![App running](screenshots/app-running.png) | ![Auto-triggered build](screenshots/auto-build.png) |

## Problems Faced and How I Fixed Them

| Problem | Cause | Fix |
|---|---|---|
| Docker Desktop: "virtualisation support wasn't detected" | Virtualization disabled in BIOS | Enabled Intel VT-x / AMD-V (SVM) in BIOS |
| "WSL needs updating" | Outdated WSL component | Ran `wsl --update` and restarted Docker Desktop |
| `docker build` could not read the Dockerfile | File saved with the wrong name or empty | Recreated it from PowerShell and verified with `type Dockerfile` |
| `git` not recognized | Git not installed / PATH not refreshed | Installed Git and opened a new terminal |
| `git add .` tried to add my whole user profile | Ran Git commands in `C:\Users\USER` instead of the project folder | Switched to the project folder and checked with `git rev-parse --show-toplevel` before committing |
| pytest would not find tests | File named `testapp.py` | Renamed it to `test_app.py` |
| Jenkins error "missing WS at '*'" when saving the job | Poll SCM schedule written as `*****` with no spaces | Used `H/2 * * * *` with spaces between fields |

## Interview Questions

**1. What is Jenkins, and how is it used in CI/CD?**
Jenkins is an open-source automation server. In CI/CD it watches a code repository and, on each change, automatically builds the project, runs tests, and deploys it. This catches bugs early and replaces manual build and release steps with a repeatable process.

**2. What is a Jenkinsfile?**
A Jenkinsfile is a text file, stored in the project repository, that defines the pipeline as code: its stages, steps, and settings. Because it lives in version control, the pipeline is versioned, reviewable, and reproducible along with the application code.

**3. How do you create and configure Jenkins pipelines?**
Install Jenkins and the Pipeline and Git plugins, then write a Jenkinsfile in the repo. In Jenkins, create a new Pipeline job, choose "Pipeline script from SCM", enter the repository URL, branch, and script path, and set a build trigger (Poll SCM or a webhook). Save and run **Build Now**, then check results in Stage View and Console Output.

**4. What are some common stages in a Jenkins pipeline?**
Checkout (get the source), Build (compile or build an image), Test (unit and integration tests), Code Analysis or Security Scan, Package/Publish (push an artifact or image to a registry), Deploy (to staging or production), and Notify (email or Slack).

**5. What is the difference between a declarative and scripted Jenkins pipeline?**
Both are written in Groovy-based syntax.
- **Declarative** uses a structured, predefined format (`pipeline { agent ... stages { ... } }`). It is simpler, easier to read, validates syntax up front, and has built-in sections such as `post` and `environment`. This project uses it.
- **Scripted** uses `node { ... }` blocks and full Groovy code. It is more flexible for complex logic but harder to read and maintain.

## Possible Improvements

- Push the built image to Docker Hub and deploy from the registry
- Use a GitHub webhook with a tunnel for instant triggers
- Add linting and security scanning stages
- Add email or Slack notifications on failure
