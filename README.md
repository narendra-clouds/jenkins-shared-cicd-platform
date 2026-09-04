# Centralized CI/CD Platform Setup using Shared Jenkins Infrastructure

## Overview

This project builds a **centralized Jenkins CI/CD platform** that multiple applications can plug into, using a **shared pipeline library** instead of every team writing (and duplicating) their own Build/Test/Scan/Deploy logic.

### Scenario

As a Platform Engineer, I joined a company where every development team was running their **own Jenkins server**, resulting in:
- Duplicate infrastructure
- Inconsistent pipelines
- Security misconfigurations

Management's instruction: *"Build a centralized CI/CD platform where multiple applications can use standardized pipelines."*

**Goal:** One shared Jenkins platform, one shared pipeline library, multiple applications reusing the exact same standardized logic — with proper role-based access control so not everyone has admin rights.

---

## Architecture

```
                    ┌─────────────────────────┐
                    │   EC2: jenkins-server   │
                    │   Ubuntu 22.04, Docker  │
                    │   Jenkins 2.568.3       │
                    └───────────┬─────────────┘
                                │
                pulls pipeline logic from
                                │
                    ┌───────────▼─────────────┐
                    │  jenkins-shared-library │  (GitHub repo)
                    │  vars/buildApp.groovy   │
                    │  vars/testApp.groovy    │
                    │  vars/scanApp.groovy    │
                    │  vars/deployApp.groovy  │
                    └───────────┬─────────────┘
                                │
              referenced via @Library(...) by
                    ┌───────────┴─────────────┐
                    │                         │
          ┌─────────▼─────────┐    ┌───────────▼───────┐
          │   sample-app-1    │    │   sample-app-2    │
          │   (GitHub repo)   │    │   (GitHub repo)   │
          │   Jenkinsfile     │    │   Jenkinsfile     │
          │   Dockerfile      │    │   Dockerfile      │
          │   → port 8081     │    │   → port 8082     │
          └───────────────────┘    └───────────────────┘

Access control:
  narendra  → admin role      (full Jenkins control)
  dev-user  → developer role  (read + build only, no config/delete/admin)
```

**Repo split (real-world convention):**
- `jenkins-shared-cicd-platform` (this repo) — documentation, screenshots, proof
- `jenkins-shared-library` — the reusable pipeline code
- `sample-app-1`, `sample-app-2` — independent apps consuming the shared library

The library lives in its own repo because it's meant to be **versioned and reused across many teams** — bundling it inside one app's repo would wrongly couple every other team's pipeline to that one app's lifecycle.

---

## Step-by-Step: How This Was Built

### Step 1 — Launch the Jenkins EC2 Instance

1. EC2 console → **Launch Instance**.
2. Name: `jenkins-server`
3. AMI: **Ubuntu Server 22.04 LTS**
4. Instance type: `t2.medium` (Jenkins needs more RAM than `t2.micro` for smooth operation)
5. Key pair: created new (`jenkins-key.pem`)
6. Security group inbound rules:
   - SSH (22) — My IP
   - Custom TCP (8080) — My IP — Jenkins web UI
   - *(later expanded to 8081–8090 for sample app containers — see Step 6)*
7. Launched.

📸 `screenshots/01-ec2-launched.png` — instance running, public IP visible.

---

### Step 2 — Install Jenkins

Connected via **EC2 Instance Connect** (browser-based terminal, no `.pem` key handling needed), then ran the official Jenkins installation steps for Ubuntu:

```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre
java -version

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins
```

Verified it was running:
```bash
sudo systemctl status jenkins
sudo systemctl enable jenkins
```

Opened `http://<public-ip>:8080` in the browser and completed the setup wizard — created an admin user (`narendra`) directly (the setup wizard was completed in one pass, skipping straight to the "Welcome to Jenkins!" dashboard).

📸 `screenshots/02-jenkins-dashboard-ready.png` — Jenkins 2.568.3 dashboard live and accessible.

---

### Step 3 — Install Required Plugins

Since the setup wizard was run with **zero plugins pre-installed**, all required plugins had to be added manually:

Went to **Manage Jenkins → Plugins → Available plugins** and installed:
1. **Pipeline** — core Jenkinsfile/pipeline support
2. **Pipeline: Shared Groovy Libraries through HTTP retrieval** — allows Jenkins to load a shared library from a GitHub repo over HTTPS
3. **Docker Pipeline** — lets pipeline stages build/run Docker containers
4. **Git** — pulls source code from GitHub repos
5. **Role-based Authorization Strategy** — needed later for Developer/Admin access control

Selected all 5 and clicked **Install**, then restarted Jenkins when prompted.

📸 `screenshots/03-plugins-installed.png` — Installed plugins list confirming all 5 present.

---

### Step 4 — Write the Shared Pipeline Library

Created a **separate GitHub repo**, `jenkins-shared-library`, containing 4 reusable pipeline steps under `vars/` — this is the Jenkins-standard convention for shared libraries (each `.groovy` file's name becomes a callable function in any pipeline that imports the library).

**`vars/buildApp.groovy`:**
```groovy
def call(Map config = [:]) {
    stage('Build') {
        echo "🔨 Building ${config.appName ?: 'application'}..."
        if (config.dockerfile) {
            sh "docker build -t ${config.appName}:${env.BUILD_NUMBER} -f ${config.dockerfile} ."
        } else {
            sh "docker build -t ${config.appName}:${env.BUILD_NUMBER} ."
        }
        echo "✅ Build complete: ${config.appName}:${env.BUILD_NUMBER}"
    }
}
```

**`vars/testApp.groovy`:**
```groovy
def call(Map config = [:]) {
    stage('Test') {
        echo "🧪 Running tests for ${config.appName ?: 'application'}..."
        if (config.testCommand) {
            sh "${config.testCommand}"
        } else {
            echo "⚠️ No test command provided — skipping (define testCommand in Jenkinsfile)"
        }
        echo "✅ Tests complete: ${config.appName}"
    }
}
```

**`vars/scanApp.groovy`:**
```groovy
def call(Map config = [:]) {
    stage('Scan') {
        echo "🔍 Scanning ${config.appName ?: 'application'} for vulnerabilities..."
        sh "echo 'Simulated scan: no critical vulnerabilities found for ${config.appName}:${env.BUILD_NUMBER}'"
        echo "✅ Scan complete: ${config.appName}"
    }
}
```

**`vars/deployApp.groovy`:**
```groovy
def call(Map config = [:]) {
    stage('Deploy') {
        echo "🚀 Deploying ${config.appName ?: 'application'}..."
        sh "docker run -d --name ${config.appName}-${env.BUILD_NUMBER} -p ${config.port ?: '8081'}:${config.internalPort ?: '80'} ${config.appName}:${env.BUILD_NUMBER}"
        echo "✅ Deployed: ${config.appName}:${env.BUILD_NUMBER} on port ${config.port ?: '8081'}"
    }
}
```

Pushed to GitHub:
```bash
git init
git add .
git commit -m "Add shared Jenkins pipeline library: build, test, scan, deploy stages"
git branch -M main
git remote add origin https://github.com/narendra-clouds/jenkins-shared-library.git
git push -u origin main
```

**Who writes this in a real company:** shared libraries like this are typically owned by a Platform/DevOps team, not individual app developers — this project simulates that role directly.

---

### Step 5 — Register the Shared Library in Jenkins

1. **Manage Jenkins → System → Global Trusted Pipeline Libraries → Add**.
2. Configured:
   - **Name:** `jenkins-shared-library` (this exact string is referenced later via `@Library('jenkins-shared-library')`)
   - **Default version:** `main`
   - **Retrieval method:** Modern SCM
   - **Source Code Management:** Git
   - **Project Repository:** `https://github.com/narendra-clouds/jenkins-shared-library.git`
   - **Credentials:** none (public repo, HTTPS)
3. Saved.

📸 `screenshots/04-shared-library-registered.png` — library configuration saved with repo URL.

---

### Step 6 — Install Docker on the Jenkins Server

Before any pipeline could actually build/run containers, Docker had to be installed on the EC2 instance — and critically, the `jenkins` system user needed permission to run Docker commands.

```bash
sudo apt update -y
sudo apt install -y docker.io

sudo systemctl start docker
sudo systemctl enable docker

# Give the jenkins user permission to run docker commands
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu

# Restart Jenkins so it picks up the new group membership
sudo systemctl restart jenkins
```

Verified:
```bash
sudo systemctl status jenkins
sudo -u jenkins docker ps
```

**Why this step matters:** Jenkins runs as a dedicated `jenkins` system user, not as the logged-in human user. By default, only `root` and members of the `docker` group can run Docker commands — skipping `usermod -aG docker jenkins` is one of the most common real-world causes of Jenkins+Docker pipelines failing with "permission denied."

---

### Step 7 — Onboard Sample App 1

Created a minimal static site (`nginx`-based) in its own repo, `sample-app-1`, with 3 files:

**`index.html`:**
```html
<!DOCTYPE html>
<html>
<head><title>Sample App 1</title></head>
<body>
    <h1>Hello from Sample App 1</h1>
    <p>Deployed via the shared Jenkins CI/CD platform.</p>
</body>
</html>
```

**`Dockerfile`:**
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

**`Jenkinsfile`** (the key file — note how short it is thanks to the shared library):
```groovy
@Library('jenkins-shared-library') _

pipeline {
    agent any
    stages {
        stage('Pipeline') {
            steps {
                script {
                    buildApp(appName: 'sample-app-1', dockerfile: 'Dockerfile')
                    testApp(appName: 'sample-app-1', testCommand: 'echo "no automated tests yet — placeholder check passed"')
                    scanApp(appName: 'sample-app-1')
                    deployApp(appName: 'sample-app-1', port: '8081', internalPort: '80')
                }
            }
        }
    }
}
```

Pushed to `https://github.com/narendra-clouds/sample-app-1.git`.

**Created the Jenkins job:**
1. **New Item** → `sample-app-1` → **Pipeline** → OK.
2. Pipeline → **Definition: Pipeline script from SCM** → Git → Repository URL: `sample-app-1.git` → Branch `*/main` → Script Path `Jenkinsfile`.
3. Saved → **Build Now**.

**Result: `Finished: SUCCESS`** — all 4 stages ran in order (Build → Test → Scan → Deploy), ending with:
```
+ docker run -d --name sample-app-1-1 -p 8081:80 sample-app-1:1
c9c45e4e6f4a8d85e27f79f4531848034c29807f502e09dcb063c19305deab3a
✅ Deployed: sample-app-1:1 on port 8081
```

📸 `screenshots/05-pipeline-success.png` — full console output, all stages green.

**Verified live:** visited `http://<public-ip>:8081` — page loaded successfully.

📸 `screenshots/06-app-live-in-browser.png` — "Hello from Sample App 1" rendering in the browser, proving the container is genuinely running and serving traffic (not just a logged success message).

---

### Step 8 — Onboard Sample App 2 (proving reusability)

Repeated the exact same pattern in a second, independent repo — `sample-app-2` — to prove the shared library actually works across multiple applications, not just one.

**`index.html`:** different content, clearly a separate app.
**`Dockerfile`:** identical pattern (nginx + static copy).
**`Jenkinsfile`:** structurally identical to App 1's — only `appName` and `port` changed:
```groovy
@Library('jenkins-shared-library') _

pipeline {
    agent any
    stages {
        stage('Pipeline') {
            steps {
                script {
                    buildApp(appName: 'sample-app-2', dockerfile: 'Dockerfile')
                    testApp(appName: 'sample-app-2', testCommand: 'echo "no automated tests yet — placeholder check passed"')
                    scanApp(appName: 'sample-app-2')
                    deployApp(appName: 'sample-app-2', port: '8082', internalPort: '80')
                }
            }
        }
    }
}
```

Created the Jenkins job the same way as App 1, pointed at `sample-app-2.git`, ran **Build Now**.

**Result: `Finished: SUCCESS`**
```
+ docker run -d --name sample-app-2-1 -p 8082:80 sample-app-2:1
dd596e7c3d967003990c187bfc6de8971cebadabf9096110a04decc6d0efb2fc
✅ Deployed: sample-app-2:1 on port 8082
```

📸 `screenshots/07-app2-pipeline-success.png`

**Verified live:** `http://<public-ip>:8082` loaded "Hello from Sample App 2" correctly.

📸 `screenshots/08-app2-live-in-browser.png`

**This is the core proof of the project's goal:** two completely independent applications, two separate repos, two different ports — but zero duplicated CI/CD logic. Both Jenkinsfiles call the exact same 4 functions from the exact same shared library.

---

### Step 9 — Configure Role-Based Access Control

To prove the platform isn't "everyone is admin," Role-Based Authorization was configured to separate **Admins** from **Developers**.

1. **Manage Jenkins → Security → Authorization** → changed from the default to **Role-Based Strategy** → Saved.
2. **Manage Jenkins → Manage and Assign Roles → Manage Roles**:
   - **admin** role → all permissions checked.
   - **developer** role → only: Overall/Read, Job/Build, Job/Read, Job/Workspace. Explicitly excluded: Job/Configure, Job/Delete, and all "Manage Jenkins" permissions.
3. Created a second Jenkins user, `dev-user`, via **Manage Jenkins → Users → Create User**.
4. **Manage and Assign Roles → Assign Roles**:
   - `narendra` → **admin**
   - `dev-user` → **developer**

📸 `screenshots/09-rbac-roles-configured.png` — Manage Roles page showing the two permission sets.

**Validation — logged in as `dev-user`:**
- Profile page showed only personal-account options (Profile, Builds, My Views, Account, Appearance, Preferences, Security, Experiments, Credentials) — **no "Manage Jenkins" link**.

📸 `screenshots/10-dev-user-restricted-view.png`

- On the main dashboard, `dev-user` **could see and build** both `sample-app-1` and `sample-app-2` (Read + Build permissions working correctly) but had **no "New Item" option** and no admin controls in the profile dropdown.

📸 `screenshots/11-dev-user-dashboard-restricted.png`

This confirms the role separation works as intended: developers can trigger builds and see results, but cannot reconfigure, delete jobs, or access Jenkins system administration.

---

## Errors Encountered & How They Were Resolved

| Error / Issue | Cause | Resolution |
|---|---|---|
| Weak default admin password (`narendra`/`narendra`) set during initial Jenkins setup | Jenkins doesn't enforce password strength during the setup wizard | Changed to a stronger password before proceeding — flagged as important since this Jenkins instance is internet-facing on port 8080, and weak-credential Jenkins servers are a common real-world attack target |
| Zero plugins installed after setup wizard | Chose the "install none" option during initial Jenkins setup instead of the recommended plugin set | Manually installed all 5 required plugins (Pipeline, Shared Groovy Libraries via HTTP, Docker Pipeline, Git, Role-based Authorization Strategy) from Available Plugins |
| Docker commands would have failed with permission denied during pipeline runs | Jenkins runs as a dedicated `jenkins` system user, which isn't in the `docker` group by default | Ran `sudo usermod -aG docker jenkins` and restarted Jenkins before running any pipeline |
| `http://<ip>:8081` (and later `:8082`) not loading in browser after successful deploy | Security group only allowed ports 22 and 8080 — the app's container port wasn't open | Added an inbound rule for a port range `8081-8090` to the EC2 security group, covering current and future sample apps in one step |

---

## Technologies & Tools

- Jenkins (self-hosted on EC2)
- Amazon EC2 (Ubuntu 22.04)
- Docker
- GitHub (3 supporting repos: shared library + 2 sample apps)
- Jenkins Pipeline / Shared Groovy Libraries
- Role-based Authorization Strategy plugin

---

## Repository Structure

```
jenkins-shared-cicd-platform/     (this repo — documentation)
├── README.md
└── screenshots/
    ├── 01-ec2-launched.png
    ├── 02-jenkins-dashboard-ready.png
    ├── 03-plugins-installed.png
    ├── 04-shared-library-registered.png
    ├── 05-pipeline-success.png
    ├── 06-app-live-in-browser.png
    ├── 07-app2-pipeline-success.png
    ├── 08-app2-live-in-browser.png
    ├── 09-rbac-roles-configured.png
    ├── 10-dev-user-restricted-view.png
    └── 11-dev-user-dashboard-restricted.png

jenkins-shared-library/           (separate repo)
└── vars/
    ├── buildApp.groovy
    ├── testApp.groovy
    ├── scanApp.groovy
    └── deployApp.groovy

sample-app-1/                     (separate repo)
├── index.html
├── Dockerfile
└── Jenkinsfile

sample-app-2/                     (separate repo)
├── index.html
├── Dockerfile
└── Jenkinsfile
```

---

## How to Reproduce This Project

1. Launch an EC2 instance (Ubuntu 22.04), open ports 22, 8080, and a range like 8081-8090.
2. Install Java 21 + Jenkins via the official APT repo, start and enable the service.
3. Complete the Jenkins setup wizard, creating an admin user with a strong password.
4. Install the plugins: Pipeline, Pipeline: Shared Groovy Libraries (HTTP retrieval), Docker Pipeline, Git, Role-based Authorization Strategy.
5. Install Docker on the instance and add the `jenkins` user to the `docker` group; restart Jenkins.
6. Create a `vars/`-based shared library repo with `buildApp`, `testApp`, `scanApp`, `deployApp` functions, and register it under Manage Jenkins → System → Global Trusted Pipeline Libraries.
7. Create at least 2 sample app repos, each with a short Jenkinsfile that imports the shared library (`@Library('jenkins-shared-library') _`) and calls the 4 functions with app-specific parameters.
8. Create a Pipeline job per app, pointing at "Pipeline script from SCM" → each app's GitHub repo → `Jenkinsfile`.
9. Enable Role-Based Strategy under Manage Jenkins → Security, define an admin and a developer role, create a second user, and verify the restricted view.

---

## Key Takeaway

A "centralized CI/CD platform" isn't just running Jenkins in one place — it's about designing a **shared, reusable pipeline library** so that every application team gets Build/Test/Scan/Deploy consistency for free, without duplicating logic or reinventing their own (often insecure) pipeline from scratch. Pairing that with role-based access control ensures the platform stays centrally governed, not a free-for-all where every developer has admin rights.