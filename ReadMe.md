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

---

# Step 2 - Configure Nginx Reverse Proxy

## Install Nginx

Install Nginx on the Ubuntu server.

```bash
sudo apt update
sudo apt install nginx -y
```

Verify the installation.

```bash
nginx -v
```

Start and enable the Nginx service.

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

---

## Create a Reverse Proxy Configuration

Create a new Nginx configuration file.

```bash
sudo nano /etc/nginx/sites-available/ok
```

Paste the following configuration.

```nginx
server {

    listen 80;

    server_name _;

    location / {

        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;

        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;

        proxy_set_header X-Real-IP $remote_addr;

        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_cache_bypass $http_upgrade;

    }

}
```

Save the file.

---

## Enable the Configuration

Enable the website by creating a symbolic link.

```bash
sudo ln -s /etc/nginx/sites-available/ok /etc/nginx/sites-enabled/
```

> **Explanation**
>
> - `sites-available` stores all Nginx configuration files.
> - `sites-enabled` contains only the active configurations.
> - `ln -s` creates a symbolic link (shortcut) from the configuration file to the enabled directory.

---

## Remove the Default Site

Disable the default Nginx configuration.

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

---

## Test the Configuration

Verify the Nginx configuration.

```bash
sudo nginx -t
```

Expected Output

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok

nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

## Restart Nginx

Restart the service.

```bash
sudo systemctl restart nginx
```

Verify the status.

```bash
sudo systemctl status nginx
```

Expected Output

```
Active: active (running)
```

---

## Verify the Reverse Proxy

Ensure the Node.js application is running on **port 3000**.

```bash
pm2 status
```

Open the application in your browser.

```
http://EC2_PUBLIC_IP
```

Instead of accessing:

```
http://EC2_PUBLIC_IP:3000
```

Nginx receives requests on **port 80** and forwards them internally to the Node.js application running on **port 3000**.

---

## Reverse Proxy Architecture

```
Browser
     │
     ▼
Port 80
     │
     ▼
Nginx
     │
     ▼
localhost:3000
     │
     ▼
Node.js Application
     │
     ▼
PM2
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
