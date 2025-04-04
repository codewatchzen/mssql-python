# GitHub to Azure DevOps (ADO) Sync & CI/CD Setup

This document provides a step-by-step guide for setting up a Python project with GitHub as the primary repository and Azure DevOps (ADO) as a mirror for internal CI/CD pipelines.

## Guidelines 

## 1. Repository Structure
### 1.1 GitHub Repository (Primary Development)
- **Branches:**
  - `main` (Protected, Stable Release)
  - `dev` (Active Development)
  - `feature/*` (Feature Branches)
  - `hotfix/*` (Hotfixes)

### 1.2 Azure DevOps Repository (Internal Pipeline Execution)
- **Branches Mirrored:** `main`, `dev`, `release-branches`
- **Does NOT push back to GitHub**
- **Changes from Github will be sync-back to ADO**

### 1.3  Folder Structure
```
/MSSQL-Python 
  |-- src/
  |-- tests/
  |-- .github/
  |   |--ISSUE_Template
  |   |   |-- bug_report.md
  |   |   |-- feature-request.md
  |   |   |-- question.md
  |   |--PR_Template
  |   |   |-- bug_fix.md
  |   |   |-- feature.md
  |   |   |-- other.md
  |   |-- workflows/
  |   │   |-- pr-checks.yml
  |   │   |-- codeql.yml
  |   │   |-- sync_to_ado.yml
  |   |-- dependabot.yml
  |-- ado-pipelines/
  |   |-- build.yml
  |   |-- release.yml
  |-- requirements.txt
  |-- setup.py
  |-- README.md
  |-- CONTRIBUTOR.md
  |-- test-pipeline.yml

---

## 2. Syncing Mechanism Between GitHub & ADO
### 2.1 GitHub Actions for Sync (`.github/workflows/sync_to_ado.yml`)
```yaml
name: Sync to ADO Repo

on:
  push:
    branches:
      - main
      - dev

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Push to Azure DevOps
        run: |
          git remote add ado https://<AZURE_USERNAME>:<AZURE_PERSONAL_ACCESS_TOKEN>@sqlclientdrivers.visualstudio.com/mssql-python/_git/mssql-python
          git push ado --mirror
```

### 2.2 Azure DevOps Scheduled Sync Pipeline (`ado-github-sync.yml`)
```yaml
trigger: none

schedules:
  - cron: "0 * * * *"
    displayName: Hourly Sync
    branches:
      include:
        - main
        - dev

pool:
  vmImage: <image-name>

steps:
  - checkout: none
  - script: |
      git clone --mirror https://github.com/microsoft/mssql-python.git
      cd <REPO>.git
      git push --mirror https://sqlclientdrivers.visualstudio.com/mssql-python/_git/mssql-python
    displayName: "Sync GitHub to ADO"
```

---

## 3. GitHub CI/CD Pipeline (PR Checks & Testing)
### 3.1 GitHub Actions for PR Checks (`.github/workflows/pr-checks.yml`)
```yaml
name: PR Checks

on:
  pull_request:
    branches:
      - main
      - dev

jobs:
  test:
    runs-on: <image-name>
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.13'

      - name: Install Dependencies
        run: pip install -r requirements.txt

      - name: Run Linter
        run: pylint src/

      - name: Run Tests
        run: pytest tests/
```

---

## 4. Security & Dependency Management
### 4.1 Dependabot for Dependency Updates (`.github/dependabot.yml`)
```yaml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
```

### 4.2 Bandit Security Scan
Modify `.github/workflows/pr-checks.yml`:
```yaml
      - name: Run Security Scan (Bandit)
        run: bandit -r mssql-python/
```

---

## 5. Azure DevOps CI/CD Pipelines
### 5.1 ADO Build Pipeline (`build.yml`)
```yaml
trigger:
  branches:
    include:
      - main
      - dev

pool:
  vmImage: ubuntu-latest

steps:
  - checkout: self
  - task: UsePythonVersion@0
    inputs:
      versionSpec: "3.13"
  - script: pip install -r requirements.txt
    displayName: "Install dependencies"
  - script: pytest tests/
    displayName: "Run tests"
  - script: pylint src/
    displayName: "Run linting"
  - task: PublishBuildArtifacts@1
    inputs:
      pathtoPublish: "dist"
      artifactName: "mssql-python"
```

### 5.2 ADO Release Pipeline (`release.yml`)
```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage:  

stages:
  - stage: Deploy
    jobs:
      - job: Deploy
        steps:
          - script: echo "TODO Based on ESRP Release Task"
            displayName: "Deploy Step"
```

---
## 5. Contribution Guidelines
### 5.1 Community Contributions
 - Fork -->Feature Branch --> PR --> PR Hooks and Checks --> Review and Approvals --> Merge
 - PR Approval mandatory from internal maintainers
 - PR validation checks triggered via Github Actions
### 5.2 ADO Coding Contributions
 - Internal Maintainers also code on Github
 - Fork -->Feature Branch --> PR --> PR Hooks and Checks --> Review and Approvals --> Merge
 - PR Approval mandatory from internal maintainers
 - PR validation checks triggered via Github Actions
 - Validate PR Code via ADO build for CI/CD (before merging PRs)
 - can create/code on protected branches for internal fixes
 ### 5.3 Access Controls
 - Github --> Branch protection rules for main and dev. 
 - Github --> Enable Dependabot
 - ADO --> Access only to Microsoft internal (for build/release and security fixes)
 

 


