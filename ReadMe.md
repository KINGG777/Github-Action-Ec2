# GitHub Actions - Automatic Deployment to Ubuntu Server

## Overview

This project demonstrates how to automatically deploy an application to an Ubuntu server using **GitHub Actions**. Whenever code is merged into the **main** branch, GitHub Actions automatically connects to the server using SSH, updates the application, and deploys the latest version.

---

# Architecture

```
Developer
     │
     ▼
Push Code to Feature Branch
     │
     ▼
Create Pull Request
     │
     ▼
Admin Reviews & Merges
     │
     ▼
Push to Main Branch
     │
     ▼
GitHub Actions Trigger
     │
     ▼
SSH into Ubuntu Server
     │
     ▼
Navigate to Application Directory
     │
     ▼
Update Source Code
     │
     ▼
Restart Application
     │
     ▼
Deployment Completed
```

---

# Prerequisites

Before configuring GitHub Actions, ensure the following requirements are met:

- Ubuntu Server is running.
- SSH access to the server is available.
- Git is installed on the server.
- The application repository is already cloned on the server.
- GitHub repository is created.
- Repository Administrator access is available.

Example application directory:

```bash
/var/www/html
```

---

# Step 1 - Prepare the Ubuntu Server

Update the package list.

```bash
sudo apt update
```

Install Git.

```bash
sudo apt install git -y
```

Verify Git installation.

```bash
git --version
```

Navigate to the application directory.

```bash
cd /var/www/html
```

Verify that the repository already exists.

```bash
git status
```

Expected Output

```
On branch main
Your branch is up to date with 'origin/main'.
```

---

# Step 2 - Generate an SSH Key Pair

Generate a dedicated deployment SSH key.

```bash
ssh-keygen -t ed25519 -C "github-actions"
```

Save the key as:

```
~/.ssh/github-actions
```

Verify the generated files.

```bash
ls ~/.ssh
```

Example

```
github-actions
github-actions.pub
```

---

# Step 3 - Authorize the Public Key

Display the public key.

```bash
cat ~/.ssh/github-actions.pub
```

Add it to the authorized keys.

```bash
cat ~/.ssh/github-actions.pub >> ~/.ssh/authorized_keys
```

Set the correct permissions.

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Verify.

```bash
cat ~/.ssh/authorized_keys
```

---

# Step 4 - Configure GitHub Secrets

Navigate to:

```
Repository

↓

Settings

↓

Secrets and variables

↓

Actions

↓

New repository secret
```

Create the following secrets.

| Secret Name | Description |
|-------------|-------------|
| HOST | Ubuntu Server Public IP |
| USERNAME | SSH Username |
| SSH_KEY | Private SSH Key (`github-actions`) |
| TARGET_DIR | `/var/www/html` |

---

# Step 5 - Create GitHub Actions Workflow

Create the following directory.

```
.github/
└── workflows/
```

Create a new file.

```
deploy.yml
```

Paste the following workflow.

```yaml
name: Deploy to Ubuntu Server

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Configure SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H ${{ secrets.HOST }} >> ~/.ssh/known_hosts

      - name: Deploy Application
        run: |
          ssh ${{ secrets.USERNAME }}@${{ secrets.HOST }} << EOF

            # Mark the repository as a trusted Git directory
            git config --global --add safe.directory ${{ secrets.TARGET_DIR }}

            # Navigate to the application directory
            cd ${{ secrets.TARGET_DIR }}

            # Fetch the latest code from GitHub
            git fetch origin

            # Synchronize the local repository with the main branch
            git reset --hard origin/main

            # Remove untracked files and directories
            git clean -fd

            # Install dependencies (if required)
            # npm install

            # Restart the application (if required)
            # pm2 restart myapp

          EOF
```

Commit and push the workflow.

```bash
git add .
git commit -m "Added GitHub Actions deployment workflow"
git push origin main
```

---

# Step 6 - Trigger Deployment

Merge a Pull Request into the **main** branch.

GitHub Actions automatically starts the deployment workflow.

Navigate to:

```
GitHub

↓

Actions
```

The workflow should execute automatically.

---

# Step 7 - Verify Deployment

SSH into the Ubuntu server.

```bash
ssh user@server-ip
```

Navigate to the application directory.

```bash
cd /var/www/html
```

Verify that the latest commit has been deployed.

```bash
git log --oneline -5
```

Check the application status.

For Apache:

```bash
sudo systemctl status apache2
```

For Nginx:

```bash
sudo systemctl status nginx
```

For PM2:

```bash
pm2 status
```

---

# Troubleshooting

## Permission denied (publickey)

Verify that:

- The public key exists in `authorized_keys`.
- The private key is correctly stored in GitHub Secrets.
- SSH permissions are correct.

---

## Fatal: Detected Dubious Ownership

If Git displays the following error:

```
fatal: detected dubious ownership in repository
```

Mark the deployment directory as a trusted Git repository.

For `/var/www/html`:

```bash
git config --global --add safe.directory /var/www/html
```

For `/var/www/ok`:

```bash
git config --global --add safe.directory /var/www/ok
```

It is recommended to include this command inside the GitHub Actions deployment workflow before executing any Git commands.

---

## GitHub Action Failed

Check:

- Repository Secrets
- Workflow syntax
- SSH connectivity
- Server availability

---

## Application Not Updated

Verify:

```bash
git status
git branch
git log --oneline
```

Ensure the repository is on the `main` branch.

---

# Deployment Flow

```
Developer
      │
      ▼
Merge Pull Request
      │
      ▼
Push to Main
      │
      ▼
GitHub Actions Trigger
      │
      ▼
SSH Login
      │
      ▼
Navigate to /var/www/html
      │
      ▼
git config --global --add safe.directory
      │
      ▼
git fetch
      │
      ▼
git reset --hard origin/main
      │
      ▼
git clean -fd
      │
      ▼
Restart Application
      │
      ▼
Deployment Completed
```

---

# Benefits

- Fully Automated Deployment
- Secure SSH Authentication
- GitHub Secrets Integration
- Zero Manual Deployment
- Production-Ready Workflow
- Handles Git "Dubious Ownership" Issues
- Easy Maintenance
- Consistent Deployments
