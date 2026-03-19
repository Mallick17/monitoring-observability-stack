# ⬡ Snyk — Complete Security Guide
> **Beginner → Expert** · Everything you need to understand, configure, and run Snyk across every environment.

---

[![Snyk](https://img.shields.io/badge/Snyk-Developer--first%20Security-4C9BE8?style=flat-square&logo=snyk)](https://snyk.io)
[![License](https://img.shields.io/badge/Guide-Open%20Reference-39D98A?style=flat-square)](https://docs.snyk.io)
[![CLI](https://img.shields.io/badge/CLI-snyk%20test-6CB8FF?style=flat-square&logo=gnubash)](https://docs.snyk.io/snyk-cli)

---

## 📋 Table of Contents

| # | Section | Level |
|---|---------|-------|
| 01 | [What is Snyk?](#01-what-is-snyk) | 🟢 Beginner |
| 02 | [Product Suite](#02-product-suite) | 🟢 Beginner |
| 03 | [How Snyk Works — Internals](#03-how-snyk-works--internals) | 🟡 Intermediate |
| 04 | [Severity Levels & CVSS](#04-severity-levels--cvss) | 🟡 Intermediate |
| 05 | [CLI Installation](#05-cli-installation) | 🟢 Beginner |
| 06 | [Authentication](#06-authentication) | 🟢 Beginner |
| 07 | [First Scan & Core Commands](#07-first-scan--core-commands) | 🟢 Beginner |
| 08 | [Docker / Container Scanning](#08-docker--container-scanning) | 🟡 Intermediate |
| 09 | [AWS CodeBuild Integration](#09-aws-codebuild-integration) | 🟡 Intermediate |
| 10 | [GitLab CI/CD Integration](#10-gitlab-cicd-integration) | 🟡 Intermediate |
| 11 | [GitHub Actions Integration](#11-github-actions-integration) | 🟡 Intermediate |
| 12 | [Jenkins Integration](#12-jenkins-integration) | 🟡 Intermediate |
| 13 | [Kubernetes Integration](#13-kubernetes-integration) | 🔴 Advanced |
| 14 | [.snyk Policy File & Configuration](#14-snyk-policy-file--configuration) | 🔴 Advanced |
| 15 | [CLI Flags Complete Reference](#15-cli-flags-complete-reference) | 🔴 Advanced |
| 16 | [IaC Scanning Deep-Dive](#16-iac-scanning-deep-dive) | 🔴 Advanced |
| 17 | [Snyk REST API](#17-snyk-rest-api) | 🔴 Advanced |
| 18 | [Best Practices — Enterprise Setup](#18-best-practices--enterprise-setup) | 🔴 Advanced |
| 19 | [Environment Comparison & Use Cases](#19-environment-comparison--use-cases) | 🔴 Advanced |
| 20 | [Plan Feature Matrix](#20-plan-feature-matrix) | 🟡 Intermediate |

---

## 01 · What is Snyk?

**Snyk** *(pronounced "sneak")* is a **developer-first security platform** that automatically finds and fixes vulnerabilities in your:
- Open source dependencies
- Container images
- Infrastructure as Code (IaC)
- Application source code (SAST)

It integrates directly into development workflows — IDEs, SCMs, CI/CD pipelines — rather than sitting outside as a bolt-on tool.

> **Core Philosophy:** *Shift security left* — catch vulnerabilities at development time, not in production. Developers own the fix, not a separate security team.

### Why Snyk Exists

Modern applications are ~**80% open source code**. Every dependency (and its transitive dependencies) can carry known CVEs. Without automated scanning, these slip into production undetected. Snyk automates that detection and surfaces actionable remediation paths *(upgrade to version X, apply patch Y)*.

---

## 02 · Product Suite

| Product | What It Scans | Key Output |
|---------|--------------|------------|
| **Snyk Open Source** (SCA) | Package manifests — npm, pip, Maven, Go modules, Gradle, NuGet, Composer, Gemfile | Vulnerable dependency with upgrade/patch path |
| **Snyk Code** (SAST) | Your own application source code | SQL injection, XSS, hardcoded secrets, path traversal |
| **Snyk Container** | Docker/OCI images, base OS layers | CVEs in Alpine, Debian, RHEL, Ubuntu base layers + app deps |
| **Snyk IaC** | Terraform, Helm, Kubernetes YAML, CloudFormation, ARM templates, Ansible | Misconfigurations before infrastructure is provisioned |

---

## 03 · How Snyk Works — Internals

### The Scan Pipeline

When you run `snyk test`, here's exactly what happens:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Detect    │────▶│    Build    │────▶│  Send to    │────▶│   Match     │────▶│   Return    │────▶│  Exit Code  │
│  Manifest   │     │  Dep Tree   │     │  Snyk API   │     │   VulnDB    │     │   Results   │     │   0 / 1     │
│package.json │     │direct +     │     │ (encrypted  │     │CVE + Snyk   │     │ with fix    │     │pass / fail  │
│  pom.xml    │     │transitive   │     │  dep list)  │     │ advisories  │     │   paths     │     │  pipeline   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

### Dependency Resolution Per Ecosystem

| Ecosystem | Manifest Files | Resolution Strategy | Transitive Deps |
|-----------|---------------|---------------------|-----------------|
| **Node.js** | `package.json`, `package-lock.json`, `yarn.lock` | Lock file preferred, fallback to npm/yarn resolve | ✅ Full |
| **Python** | `requirements.txt`, `Pipfile`, `pyproject.toml` | pip resolution + virtual env | ✅ Full |
| **Java** | `pom.xml`, `build.gradle` | Maven/Gradle build required locally | ✅ Full |
| **Go** | `go.mod`, `go.sum` | Module graph from `go list` | ✅ Full |
| **Ruby** | `Gemfile.lock` | Lock file parse | ✅ Full |
| **.NET** | `*.csproj`, `packages.config` | NuGet resolution | ✅ Full |
| **PHP** | `composer.lock` | Lock file parse | ✅ Full |

> ⚠️ **Java/Gradle**: Snyk needs a build step first (`./gradlew dependencies`) or use `--all-sub-projects`.
> ⚠️ **Python**: Activate your virtualenv before scanning.

---

## 04 · Severity Levels & CVSS

Snyk uses **CVSS v3.1** scoring augmented with its own exploit intelligence (actual exploit availability, ease of exploitation in real environments).

| Severity | CVSS Score | Snyk Action | Real-World Example |
|----------|-----------|-------------|-------------------|
| 🔴 **CRITICAL** | 9.0 – 10.0 | Block pipeline immediately | RCE via log4shell `CVE-2021-44228` |
| 🟠 **HIGH** | 7.0 – 8.9 | Block or alert depending on policy | Prototype pollution with code exec |
| 🟡 **MEDIUM** | 4.0 – 6.9 | Warn, create ticket | ReDoS, SSRF without auth bypass |
| 🟢 **LOW** | 0.1 – 3.9 | Track, batch-fix | Info disclosure, timing attacks |

> 💡 **Pro Tip:** Use `--severity-threshold=high` to only fail builds on HIGH+ findings. This avoids alert fatigue from LOW/MEDIUM noise while still catching what matters.

---

## 05 · CLI Installation

### Option A — npm (Recommended for Node.js projects)

```bash
# Install globally via npm (requires Node 18+)
npm install -g snyk

# Verify installation
snyk --version
# → 1.1291.0 (or latest)

# Keep it updated
npm update -g snyk
```

### Option B — Homebrew (macOS / Linux)

```bash
brew tap snyk/tap
brew install snyk

# Update later
brew upgrade snyk
```

### Option C — Standalone Binary (No Node required)

```bash
# Linux x64
curl -Lo snyk https://static.snyk.io/cli/latest/snyk-linux
chmod +x snyk
mv snyk /usr/local/bin/

# macOS arm64
curl -Lo snyk https://static.snyk.io/cli/latest/snyk-macos-arm64
chmod +x snyk
mv snyk /usr/local/bin/

# Windows — PowerShell
Invoke-WebRequest -Uri https://static.snyk.io/cli/latest/snyk-win.exe -OutFile snyk.exe
```

### Option D — Docker (Zero local install)

```bash
# Run Snyk CLI via Docker — no installation needed
docker run --rm -it \
  -e SNYK_TOKEN=$SNYK_TOKEN \
  -v "$(pwd):/project" \
  snyk/snyk:linux snyk test
```

---

## 06 · Authentication

### Methods

| Method | Use Case | Command |
|--------|----------|---------|
| **Interactive OAuth** | Local development | `snyk auth` |
| **Token env var** | CI/CD pipelines | `export SNYK_TOKEN=xxx` |
| **Service account** | Enterprise, multi-team | Create in Snyk Org Settings |

```bash
# Interactive (local machine) — opens browser
snyk auth

# CI/CD — set environment variable
export SNYK_TOKEN="your-api-token-here"

# OR authenticate non-interactively
snyk auth $SNYK_TOKEN

# Verify authenticated identity
snyk whoami
```

> 🔐 **Security Rule:** Never hardcode tokens in config files. Always use environment variables or secrets managers (AWS SSM, HashiCorp Vault, GitHub Secrets).

---

## 07 · First Scan & Core Commands

```bash
# ── OPEN SOURCE SCAN ────────────────────────────────────────────────
snyk test                           # Scan current directory
snyk test --all-projects            # Scan all sub-projects (monorepo)
snyk test --severity-threshold=high # Only fail on HIGH+
snyk test --json                    # Machine-readable JSON output
snyk test --sarif                   # SARIF format (for GitHub Advanced Security)
snyk test --file=package.json       # Specify manifest explicitly

# ── MONITOR (continuous tracking on dashboard) ───────────────────────
snyk monitor                        # Upload snapshot to Snyk dashboard
snyk monitor --project-name=my-app  # Name the project in dashboard

# ── CODE SCAN (SAST) ─────────────────────────────────────────────────
snyk code test
snyk code test --severity-threshold=medium

# ── CONTAINER SCAN ───────────────────────────────────────────────────
snyk container test nginx:latest
snyk container test myapp:latest --file=Dockerfile
snyk container monitor myapp:latest --project-name=myapp-prod

# ── IaC SCAN ─────────────────────────────────────────────────────────
snyk iac test                       # Scan all IaC files in directory
snyk iac test terraform/            # Specific directory
snyk iac test k8s/deployment.yaml   # Specific file

# ── AUTO-FIX ─────────────────────────────────────────────────────────
snyk fix                            # Auto-upgrade vulnerable dependencies
```

### Reading the Output

```
✔ Tested 847 dependencies for known issues

✗ 3 vulnerabilities found (1 critical, 2 high)

  ✗ [Critical] Prototype Pollution
    Package:    lodash
    Version:    4.17.15
    Fix:        Upgrade to lodash@4.17.21
    CVE:        CVE-2020-8203
    CVSS Score: 9.8
    Introduced: project > express > lodash
    Info:       https://snyk.io/vuln/SNYK-JS-LODASH-567746

Run `snyk fix` to apply the recommended fixes.
Run `snyk monitor` to track this project on snyk.io

✗ 1 critical severity vulnerability, fix available
```

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | No vulnerabilities found (or below threshold) |
| `1` | Vulnerabilities found at or above threshold |
| `2` | Snyk CLI error (e.g., auth failure, network) |
| `3` | No supported files found |

---

## 08 · Docker / Container Scanning

Snyk supports two container modes:
1. **Scanning Docker images** — finds CVEs in OS layers + app dependencies
2. **Running Snyk CLI inside Docker** — clean, reproducible CI environment

### 8.1 — Scan a Docker Image

```bash
# Scan a public image
snyk container test nginx:latest

# Scan your own built image
docker build -t myapp:latest .
snyk container test myapp:latest \
  --file=Dockerfile \              # Include Dockerfile for better context
  --severity-threshold=high \
  --exclude-base-image-vulns       # Only app-layer vulns, not OS base

# Monitor container in Snyk dashboard
snyk container monitor myapp:latest --project-name=myapp-prod

# Output as JSON for parsing
snyk container test myapp:latest --json > container-results.json
```

### 8.2 — Multi-Stage Dockerfile with Security Gate

```dockerfile
# ── Stage 1: Build ──────────────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

# ── Stage 2: Security Scan (fails build if critical vulns found) ─────
FROM snyk/snyk:node AS security-check
WORKDIR /app
COPY --from=builder /app /app
ARG SNYK_TOKEN
ENV SNYK_TOKEN=$SNYK_TOKEN
RUN snyk test --severity-threshold=critical || true

# ── Stage 3: Production Image ────────────────────────────────────────
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app /app
EXPOSE 3000
CMD ["node", "server.js"]
```

Build with:
```bash
docker build --build-arg SNYK_TOKEN=$SNYK_TOKEN -t myapp:latest .
```

### 8.3 — Docker Compose Integration

```yaml
version: '3.8'
services:
  snyk-scan:
    image: snyk/snyk:linux
    volumes:
      - ./:/project
    working_dir: /project
    environment:
      - SNYK_TOKEN=${SNYK_TOKEN}
    command: snyk test --severity-threshold=high
    profiles: [security]   # Activate with: docker compose --profile security up
```

### 8.4 — Container Scan Flags

| Flag | Purpose |
|------|---------|
| `--file=Dockerfile` | Adds Dockerfile context for better fix suggestions |
| `--exclude-base-image-vulns` | Show only app-layer vulns, skip OS base image noise |
| `--platform=linux/amd64` | Specify target platform for multi-arch images |
| `--app-vulns` | Include app dependency scan inside the container |
| `--json` | Machine-readable output |

---

## 09 · AWS CodeBuild Integration

### Step 1 — Store Token in AWS SSM Parameter Store

```bash
aws ssm put-parameter \
  --name "/cicd/snyk-token" \
  --value "your-snyk-api-token" \
  --type SecureString \
  --region ap-south-1
```

> ℹ️ Add **`ssm:GetParameter`** permission to the CodeBuild IAM role.

### Step 2 — Full `buildspec.yml`

```yaml
version: 0.2

env:
  parameter-store:
    SNYK_TOKEN: /cicd/snyk-token        # Pulled from SSM automatically

phases:
  install:
    runtime-versions:
      nodejs: 20
    commands:
      - npm install -g snyk
      - npm ci

  pre_build:
    commands:
      - echo "Running Snyk security scan..."
      # Fail build on HIGH+ vulnerabilities
      - snyk test --severity-threshold=high --json > snyk-report.json || BUILD_FAILED=true
      # Upload snapshot for continuous monitoring
      - snyk monitor --project-name=my-app-${CODEBUILD_RESOLVED_SOURCE_VERSION}
      - if [ "$BUILD_FAILED" = "true" ]; then echo "Snyk found HIGH+ vulns"; exit 1; fi

  build:
    commands:
      - npm run build
      # Container scan after Docker build
      - docker build -t myapp:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - snyk container test myapp:$CODEBUILD_RESOLVED_SOURCE_VERSION --severity-threshold=high

  post_build:
    commands:
      - echo "Security scan complete"

reports:
  snyk-vulnerability-report:
    files:
      - snyk-report.json
    file-format: CUCUMBERJSON

artifacts:
  files:
    - snyk-report.json
```

### CodeBuild IAM Role Permissions Required

```json
{
  "Effect": "Allow",
  "Action": [
    "ssm:GetParameter",
    "ssm:GetParameters"
  ],
  "Resource": "arn:aws:ssm:*:*:parameter/cicd/snyk-token"
}
```

---

## 10 · GitLab CI/CD Integration

### Setup
Go to **Settings → CI/CD → Variables** → Add `SNYK_TOKEN` *(masked + protected)*.

### Full `.gitlab-ci.yml`

```yaml
stages:
  - security
  - build
  - deploy

# ── SNYK OPEN SOURCE SCAN ─────────────────────────────────────────────
snyk:open-source:
  stage: security
  image: snyk/snyk:node
  variables:
    SNYK_TOKEN: $SNYK_TOKEN
  script:
    - snyk test --severity-threshold=high --json > gl-sast-report.json
    - snyk monitor --project-name=$CI_PROJECT_NAME
  artifacts:
    reports:
      sast: gl-sast-report.json    # Appears in GitLab Security Dashboard
    when: always
  allow_failure: false             # Block merge if HIGH+ vulns found
  only:
    - merge_requests
    - main
    - develop

# ── SNYK CODE SCAN (SAST) ─────────────────────────────────────────────
snyk:code:
  stage: security
  image: snyk/snyk:node
  script:
    - snyk code test --severity-threshold=medium
  allow_failure: true              # Warn only, don't block pipeline

# ── CONTAINER SCAN ────────────────────────────────────────────────────
snyk:container:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - apk add --no-cache curl
    - curl -Lo /usr/local/bin/snyk https://static.snyk.io/cli/latest/snyk-linux
    - chmod +x /usr/local/bin/snyk
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - snyk container test $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA --severity-threshold=high
  only:
    - main

# ── IaC SCAN ──────────────────────────────────────────────────────────
snyk:iac:
  stage: security
  image: snyk/snyk:linux
  script:
    - snyk iac test --severity-threshold=medium
  allow_failure: true
```

### GitLab Stage Strategy

| Stage | Job | `allow_failure` | Blocks Merge? |
|-------|-----|-----------------|---------------|
| `security` | `snyk:open-source` | `false` | ✅ Yes |
| `security` | `snyk:code` | `true` | ❌ Warn only |
| `security` | `snyk:iac` | `true` | ❌ Warn only |
| `build` | `snyk:container` | `false` | ✅ Yes |

---

## 11 · GitHub Actions Integration

### Setup
Go to **Settings → Secrets and variables → Actions** → Add `SNYK_TOKEN`.

### Full Workflow — `.github/workflows/snyk.yml`

```yaml
name: Snyk Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'          # Weekly Monday 2AM UTC

jobs:
  snyk:
    runs-on: ubuntu-latest
    permissions:
      security-events: write      # Required for GHAS SARIF upload
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      # ── Open Source (SCA) ──────────────────────────────────────────
      - name: Snyk Open Source
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high --sarif-file-output=snyk.sarif

      # Upload results to GitHub Advanced Security (GHAS)
      - name: Upload SARIF to GitHub
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: snyk.sarif

      # ── Snyk Code (SAST) ───────────────────────────────────────────
      - name: Snyk Code
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          command: code test
        continue-on-error: true  # Warn only

      # ── Container Scan ─────────────────────────────────────────────
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Snyk Container
        uses: snyk/actions/docker@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          image: myapp:${{ github.sha }}
          args: --severity-threshold=high

      # ── IaC Scan ───────────────────────────────────────────────────
      - name: Snyk IaC
        uses: snyk/actions/iac@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=medium
        continue-on-error: true
```

### Available Official Snyk Actions

| Action | Use For |
|--------|---------|
| `snyk/actions/node@master` | Node.js open source + code |
| `snyk/actions/python@master` | Python projects |
| `snyk/actions/docker@master` | Container images |
| `snyk/actions/iac@master` | Terraform, K8s, CloudFormation |
| `snyk/actions/gradle@master` | Java/Gradle projects |
| `snyk/actions/maven@master` | Java/Maven projects |
| `snyk/actions/golang@master` | Go modules |

---

## 12 · Jenkins Integration

### Setup
Create a **Jenkins Credential** → Secret Text → ID: `snyk-api-token`.

### Jenkinsfile — Declarative Pipeline

```groovy
pipeline {
  agent { docker { image 'node:20-alpine' } }

  environment {
    SNYK_TOKEN = credentials('snyk-api-token')    // Pulls from Jenkins Credentials
  }

  stages {
    stage('Install') {
      steps {
        sh 'npm ci'
        sh 'npm install -g snyk'
      }
    }

    stage('Snyk Security Scan') {
      steps {
        script {
          try {
            sh '''
              snyk test \
                --severity-threshold=high \
                --json > snyk-report.json
            '''
          } catch (err) {
            // Capture failure but archive report first
            currentBuild.result = 'UNSTABLE'
          }
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'snyk-report.json'
        }
      }
    }

    stage('Snyk Monitor') {
      when { branch 'main' }             // Only upload snapshot from main
      steps {
        sh 'snyk monitor --project-name=my-app'
      }
    }

    stage('Container Scan') {
      steps {
        sh 'docker build -t myapp:${BUILD_NUMBER} .'
        sh 'snyk container test myapp:${BUILD_NUMBER} --severity-threshold=high'
      }
    }
  }

  post {
    failure {
      slackSend channel: '#security',
        message: "🔴 Snyk found vulnerabilities in ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
  }
}
```

---

## 13 · Kubernetes Integration

### 13.1 — Scan Kubernetes Manifests (IaC)

```bash
# Scan a specific manifest
snyk iac test k8s/deployment.yaml

# Scan entire k8s/ directory
snyk iac test k8s/ --severity-threshold=medium

# Generate SARIF output for IDE / GitHub Advanced Security
snyk iac test k8s/ --sarif > iac-results.sarif
```

### 13.2 — Snyk Controller (Runtime Monitoring via Helm)

```bash
# Add Helm repo
helm repo add snyk-charts https://snyk.github.io/kubernetes-monitor/
helm repo update

# Create namespace and secret
kubectl create namespace snyk-monitor
kubectl create secret generic snyk-monitor-secret \
  --from-literal=dockerconfigjson.json={} \
  --from-literal=integrationId=YOUR_INTEGRATION_ID \
  -n snyk-monitor

# Deploy Snyk Controller
helm install snyk-monitor snyk-charts/snyk-monitor \
  --namespace snyk-monitor \
  --set clusterName=production-cluster
```

The Snyk Controller continuously monitors running workloads and reports new CVEs to the Snyk dashboard — even without re-running CI.

### 13.3 — Kubernetes Job Manifest

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: snyk-scan
  namespace: security
spec:
  template:
    spec:
      containers:
      - name: snyk
        image: snyk/snyk:linux
        command: ["snyk", "test", "--severity-threshold=high"]
        env:
        - name: SNYK_TOKEN
          valueFrom:
            secretKeyRef:
              name: snyk-credentials
              key: token
        volumeMounts:
        - mountPath: /project
          name: source-code
        workingDir: /project
      volumes:
      - name: source-code
        configMap:
          name: app-source
      restartPolicy: Never
```

Create the secret:
```bash
kubectl create secret generic snyk-credentials \
  --from-literal=token=$SNYK_TOKEN \
  -n security
```

---

## 14 · .snyk Policy File & Configuration

The `.snyk` file lives in your **project root** and controls ignore rules, patches, and language settings. Commit it to version control.

### Full `.snyk` File Anatomy

```yaml
# .snyk — Snyk policy file (commit this to your repo)
version: v1.25.1

# ── IGNORE RULES ────────────────────────────────────────────────────
ignore:
  # Ignore a specific vulnerability in a specific package path
  SNYK-JS-LODASH-567746:
    - '* > lodash':
        reason: 'Not exploitable in our usage pattern — lodash used only for server-side clone()'
        expires: '2025-12-31T00:00:00.000Z'    # ALWAYS set an expiry date
        created: '2024-01-15T00:00:00.000Z'

  # Ignore across all paths (use very sparingly)
  SNYK-JS-AXIOS-1038255:
    - '*':
        reason: 'Tracked in JIRA-1234, fix scheduled for Q2 sprint'
        expires: '2025-06-01T00:00:00.000Z'

# ── PATCHES (Snyk-provided backport patches) ─────────────────────────
patch:
  SNYK-JS-UGLIFYJS-10179:
    - 'uglify-js > source-map':
        patched: '2024-01-10T00:00:00.000Z'

# ── LANGUAGE SETTINGS ────────────────────────────────────────────────
language-settings:
  javascript:
    packageManager: npm       # npm | yarn | pnpm
```

### Global CLI Config (`.snykrc` equivalent)

```bash
# Set default organization (override per-command with --org flag)
snyk config set org=my-org-slug

# Set API endpoint (for self-hosted Snyk broker)
snyk config set endpoint=https://app.snyk.io/api

# Disable analytics/telemetry
snyk config set disable-analytics=1

# View current config
snyk config get org
snyk config get endpoint

# Clear a config key
snyk config unset org
```

### Ignore Rules — Best Practices

| Rule | ✅ Good | ❌ Bad |
|------|---------|-------|
| Scope | Ignore only the specific vulnerable path | Wildcard `*` ignores on everything |
| Expiry | Always set `expires:` date | No expiry — ignores live forever silently |
| Reason | Detailed reason linked to JIRA ticket | Vague "not applicable" |
| Review | Re-review on expiry, update or fix | Set and forget |

---

## 15 · CLI Flags Complete Reference

### `snyk test` Flags

| Flag | Values | Purpose |
|------|--------|---------|
| `--severity-threshold` | `low` / `medium` / `high` / `critical` | Only fail at or above this level |
| `--json` | — | Machine-readable JSON output |
| `--sarif` | — | SARIF output for GHAS / IDEs |
| `--all-projects` | — | Scan all projects in monorepo |
| `--org` | `<org-slug>` | Override Snyk organization |
| `--file` | `<path>` | Specify manifest file explicitly |
| `--fail-on` | `all` / `upgradable` / `patchable` | When to exit code 1 |
| `--show-vulnerable-paths` | `none` / `some` / `all` | Dep chain display verbosity |
| `--policy-path` | `<path>` | Custom `.snyk` policy file location |
| `--trust-policies` | — | Apply upstream package policy files |
| `--prune-repeated-subdependencies` | — | Reduce noise in large dep trees (`-p`) |
| `--detection-depth` | `<int>` | How deep to search for manifest files |
| `--exclude` | `<dir>` | Exclude directory from scan |

### `snyk monitor` Flags

| Flag | Values | Purpose |
|------|--------|---------|
| `--project-name` | `<string>` | Name project in Snyk dashboard |
| `--target-reference` | `<string>` | Tag snapshot by branch/env (e.g., `main`) |
| `--tags` | `key=val,key2=val2` | Add metadata tags for filtering |
| `--remote-repo-url` | `<url>` | Link to source repo URL |

### `snyk container test` Flags

| Flag | Purpose |
|------|---------|
| `--file=Dockerfile` | Include Dockerfile for enhanced context |
| `--exclude-base-image-vulns` | Skip OS base layer, show only app deps |
| `--platform=linux/amd64` | Target platform for multi-arch images |
| `--app-vulns` | Scan app deps inside the container image |

---

## 16 · IaC Scanning Deep-Dive

### Supported IaC Targets

| Tool | File Types | What Snyk Checks |
|------|-----------|-----------------|
| **Terraform** | `*.tf`, `*.tfvars`, `tfplan.json` | Open S3 buckets, unencrypted RDS, open security groups, missing encryption |
| **Kubernetes** | `*.yaml`, `*.yml` | Privileged containers, missing resource limits, root users, missing network policies |
| **CloudFormation** | `*.json`, `*.yaml` template | IAM wildcards, public S3, unencrypted EBS, missing CloudTrail |
| **Helm** | Rendered chart templates | Same as K8s — render first with `helm template` |
| **Azure ARM** | `*.json` ARM templates | Missing encryption, open NSG rules, missing auditing |
| **Ansible** | `*.yaml` playbooks | Insecure task configurations |

### IaC Scan Examples

```bash
# ── Terraform ────────────────────────────────────────────────────────
# Scan .tf files directly
snyk iac test infrastructure/

# Scan a Terraform plan (post-apply preview)
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > tfplan.json
snyk iac test tfplan.json

# ── Kubernetes ───────────────────────────────────────────────────────
snyk iac test k8s/deployment.yaml
snyk iac test k8s/ --severity-threshold=medium

# ── Helm ─────────────────────────────────────────────────────────────
helm template my-release ./helm-chart > rendered.yaml
snyk iac test rendered.yaml

# ── Custom OPA/Rego Rules ────────────────────────────────────────────
# Bundle your custom rules and pass them
snyk iac test --rules=./custom-rules.tar.gz

# ── SARIF output for IDE integration ─────────────────────────────────
snyk iac test k8s/ --sarif > iac-results.sarif
```

### Common IaC Findings

```
✗ [High] S3 Bucket Has Public ACL
  Path:    resource > aws_s3_bucket > my-bucket > acl
  Fix:     Set acl = "private"
  Rule:    SNYK-CC-TF-19

✗ [Medium] Container Running as Root
  Path:    spec > containers[0] > securityContext > runAsNonRoot
  Fix:     Set runAsNonRoot: true
  Rule:    SNYK-CC-K8S-6
```

---

## 17 · Snyk REST API

For automation beyond the CLI — bulk reporting, JIRA integration, custom dashboards.

### Base URLs

| API Version | Base URL | Notes |
|-------------|----------|-------|
| **v1** (stable) | `https://api.snyk.io/v1` | Full-featured, widely used |
| **REST v3** (newer) | `https://api.snyk.io/rest` | Newer resources, use for new integrations |

### Common API Calls

```bash
BASE="https://api.snyk.io/v1"
TOKEN="$SNYK_TOKEN"

# ── List all organizations ────────────────────────────────────────────
curl -H "Authorization: token $TOKEN" \
  $BASE/orgs

# ── List all projects in an org ───────────────────────────────────────
curl -H "Authorization: token $TOKEN" \
  $BASE/org/ORG_ID/projects

# ── Get issues for a specific project ─────────────────────────────────
curl -H "Authorization: token $TOKEN" \
  $BASE/org/ORG_ID/project/PROJECT_ID/issues

# ── Trigger a test via API (Node.js example) ─────────────────────────
curl -X POST \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"packageManager":"npm","encodingType":"plain","projectName":"my-app"}' \
  $BASE/test/npm

# ── Create an ignore via API (for scripted batch ignores) ────────────
curl -X POST \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"ignorePath":"*","reason":"False positive","disregardIfFixable":false}' \
  $BASE/org/ORG_ID/project/PROJECT_ID/ignore/ISSUE_ID

# ── Get aggregate vuln stats per org ─────────────────────────────────
curl -H "Authorization: token $TOKEN" \
  "$BASE/reporting/issues?from=2024-01-01&to=2024-12-31"
```

---

## 18 · Best Practices — Enterprise Setup

### Core Rules

| Rule | Why |
|------|-----|
| **Gate on HIGH+ only in CI** | Avoid alert fatigue. LOW/MEDIUM tracked separately in dashboard. |
| **Always set expiry on ignores** | Forces periodic review. Prevents silent permanent suppression. |
| **Run `snyk monitor` on main only** | Keep dashboard clean. Don't pollute with every feature branch. |
| **Tag projects consistently** | Enables filtered reporting and team ownership assignment. |
| **Rotate service account tokens quarterly** | Limit blast radius of token compromise. |
| **Never use `*` wildcard ignores without justification** | Overly broad ignores defeat the purpose of scanning. |
| **Use `--fail-on=upgradable`** | Only block builds for vulns that actually have a fix available. |

### Recommended Workflow Per Stage

| Stage | Command | Threshold | On Failure | Plan |
|-------|---------|-----------|------------|------|
| **IDE** | Snyk IDE Plugin | All severities shown | Warning only | `FREE` |
| **PR / Merge Request** | `snyk test` | HIGH+ | Block merge | `FREE` |
| **Main branch CI** | `snyk test` + `snyk monitor` | HIGH+ | Fail build | `FREE` |
| **Container registry** | `snyk container test` | HIGH+ | Block push | `TEAM` |
| **Pre-production deploy** | `snyk iac test` | MEDIUM+ | Block deploy | `TEAM` |
| **Production runtime** | `snyk monitor` | Monitoring only | Slack alert | `ENTERPRISE` |
| **Scheduled (weekly)** | `snyk test --all-projects` | MEDIUM+ | Auto JIRA ticket | `TEAM` |

### Tagging Strategy

```bash
# Tag projects for org-level filtering in Snyk dashboard
snyk monitor \
  --project-name=payment-service \
  --target-reference=main \
  --tags=team=payments,env=prod,repo=payment-service,criticality=high
```

### Secret Management Per Platform

| Platform | Where to Store `SNYK_TOKEN` |
|----------|-----------------------------|
| AWS CodeBuild | SSM Parameter Store (SecureString) or Secrets Manager |
| GitLab CI | CI/CD Variables (masked + protected) |
| GitHub Actions | Repository or Organization Secrets |
| Jenkins | Jenkins Credentials (Secret Text) |
| Kubernetes | K8s Secret + secretKeyRef |
| HashiCorp Vault | `vault kv put secret/snyk token=xxx` |

---

## 19 · Environment Comparison & Use Cases

### Which CI/CD Setup is Best?

| | GitHub Actions | GitLab CI | AWS CodeBuild | Jenkins | CircleCI |
|-|---------------|-----------|---------------|---------|----------|
| **Setup Effort** | 🟢 Low | 🟢 Low | 🟡 Medium | 🔴 High | 🟢 Low |
| **Official Action/Template** | ✅ Yes | ✅ Yes | ❌ Manual | ✅ Plugin | ✅ Orb |
| **SARIF / Security Dashboard** | ✅ GHAS | ✅ Native | ❌ Reports only | ❌ Manual | ❌ Manual |
| **Self-hosted support** | 🟡 Runners | ✅ Excellent | 🟡 VPC | ✅ Best | 🟡 Runners |
| **AWS-native secrets** | ❌ | ❌ | ✅ SSM/SM native | 🟡 Plugin | ❌ |
| **PR-level blocking** | ✅ Native | ✅ Native | 🟡 Via webhook | 🟡 Plugin | 🟡 Plugin |
| **Best For** | GitHub teams | GitLab self-managed | AWS-native teams | Enterprise on-prem | SaaS teams |

### Use-Case Recommendations

| Scenario | Recommended Setup |
|----------|-------------------|
| SaaS startup on GitHub | GitHub Actions + `snyk/actions/node` + GHAS SARIF |
| Enterprise with self-hosted GitLab | GitLab CI with Security Dashboard + Snyk Controller in K8s |
| AWS-first team (CodePipeline) | CodeBuild + SSM token + `snyk container test` on ECR push |
| Large enterprise, Jenkins on-prem | Jenkins Groovy pipeline + Snyk plugin + Vault for token |
| Kubernetes-heavy platform team | Snyk Controller (runtime) + `snyk iac test` in CI + container scan |
| Monorepo | `snyk test --all-projects` + `--detection-depth=4` |

---

## 20 · Plan Feature Matrix

| Feature | `FREE` | `TEAM` | `ENTERPRISE` |
|---------|--------|--------|--------------|
| **Open Source SCA** | ✅ 200 tests/mo | ✅ Unlimited | ✅ Unlimited |
| **Snyk Code (SAST)** | ✅ Limited | ✅ Full | ✅ Full |
| **Container Scanning** | ✅ 100 tests/mo | ✅ Unlimited | ✅ Unlimited |
| **IaC Scanning** | ✅ 300 tests/mo | ✅ Unlimited | ✅ Unlimited |
| **Priority Score** | ❌ | ✅ | ✅ |
| **Custom IaC Rules (OPA)** | ❌ | ❌ | ✅ |
| **SSO / SAML** | ❌ | ❌ | ✅ |
| **Service Accounts** | ❌ | ❌ | ✅ |
| **JIRA Integration** | ❌ | ✅ | ✅ |
| **Slack Integration** | ❌ | ✅ | ✅ |
| **Snyk Learn (training)** | ✅ | ✅ | ✅ |
| **IDE Plugin** | ✅ | ✅ | ✅ |
| **Reporting API** | ❌ | ✅ | ✅ |
| **Group-level policies** | ❌ | ❌ | ✅ |
| **Audit Logs** | ❌ | ❌ | ✅ |

---

## Quick Reference Cheatsheet

```bash
# ── AUTHENTICATE ──────────────────────────────────────────────────────
snyk auth                                         # Interactive browser login
export SNYK_TOKEN="your-token"                    # CI/CD token method

# ── OPEN SOURCE ───────────────────────────────────────────────────────
snyk test                                         # Basic scan
snyk test --severity-threshold=high               # Fail on HIGH+ only
snyk test --all-projects                          # Monorepo
snyk test --json > report.json                    # JSON output
snyk monitor --project-name=app --tags=env=prod   # Dashboard snapshot

# ── CODE ──────────────────────────────────────────────────────────────
snyk code test                                    # SAST scan

# ── CONTAINER ─────────────────────────────────────────────────────────
snyk container test myapp:latest                  # Scan image
snyk container test myapp:latest --file=Dockerfile
snyk container monitor myapp:latest               # Track on dashboard

# ── IaC ───────────────────────────────────────────────────────────────
snyk iac test                                     # All IaC files
snyk iac test terraform/ --severity-threshold=medium
snyk iac test --sarif > iac.sarif                 # SARIF output

# ── FIX ───────────────────────────────────────────────────────────────
snyk fix                                          # Auto-upgrade deps
snyk ignore --id=SNYK-JS-LODASH-567746 --expiry=2025-12-31 --reason="Not exploitable"
```

---

## Resources

| Resource | URL |
|----------|-----|
| Official Docs | https://docs.snyk.io |
| Vulnerability Database | https://snyk.io/vuln |
| CLI Reference | https://docs.snyk.io/snyk-cli/cli-commands-and-options-summary |
| API Reference | https://snyk.docs.apiary.io |
| Snyk Learn (training) | https://learn.snyk.io |
| GitHub (open source CLI) | https://github.com/snyk/snyk |
| Snyk Images (Docker) | https://hub.docker.com/r/snyk/snyk |

---

*Guide maintained for Snyk CLI v1.x — Last updated 2025*
