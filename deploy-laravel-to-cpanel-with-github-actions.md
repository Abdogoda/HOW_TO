# 🚀 Automated CI/CD Pipeline: GitHub Actions → cPanel

This guide sets up an automatic deployment pipeline for a Laravel app hosted on cPanel.

**What it does:**
- Every push or pull request runs your test suite automatically.
- Every push to `main` (after tests pass) logs into your cPanel server via SSH and deploys the latest code.

**Who this is for:** Absolute beginners. No prior CI/CD experience needed — just follow each step in order.

---

## 📋 Prerequisites

- A GitHub repository containing your Laravel app.
- A cPanel hosting account with SSH access enabled.
- Terminal access (Mac/Linux) or Git Bash (Windows).

---

## 🛠️ Part 1: One-Time Server Setup

### Step 1: Generate an SSH Key Pair

An SSH key pair lets GitHub prove its identity to your server without a password. You generate two files: a **private key** (secret, stays with GitHub) and a **public key** (shared with cPanel).

Open a terminal and run:

```bash
ssh-keygen -t rsa -b 4096 -f ./server_deploy_key -N ""
```

**What each part means:**
| Part | Meaning |
| --- | --- |
| `ssh-keygen` | The command that creates SSH keys |
| `-t rsa` | Use the RSA encryption algorithm |
| `-b 4096` | Make the key 4096 bits long (strong security) |
| `-f ./server_deploy_key` | Save the key files here, named `server_deploy_key` (private) and `server_deploy_key.pub` (public) |
| `-N ""` | Set an empty password on the key, so GitHub Actions can use it without manual input |

This creates two files in your current folder:
- `server_deploy_key` → the **private** key (keep secret, goes into GitHub Secrets)
- `server_deploy_key.pub` → the **public** key (goes into cPanel)

### Step 2: Add the Public Key to cPanel

1. Log into **cPanel** → go to **Security** → click **SSH Access**.
2. Click **Manage SSH Keys** → **Import Key**.
3. Open `server_deploy_key.pub` in a text editor, copy its entire contents, and paste it into the import box. Save.
4. Go back to the key list, find your new key, click **Manage**, then click **Authorize**.
   - *Authorizing* is required — an imported key does nothing until it's authorized.

### Step 3: Let the Server Pull Code Without a Password Prompt

Your server's local Git needs permission to pull from GitHub. The easiest way (especially if you can't add a Deploy Key at the repo level) is to embed a Personal Access Token (PAT) in the remote URL.

1. On GitHub: **Settings** → **Developer settings** → **Personal access tokens** → generate a **Classic token** with the `repo` scope.
2. In cPanel, open **Terminal** (or SSH in), then run:

```bash
cd /path/to/your/application-root
```
*Moves you into your project's folder on the server. Replace with your actual path.*

```bash
git remote set-url origin https://YOUR_GITHUB_USERNAME:YOUR_PERSONAL_ACCESS_TOKEN@github.com/OWNER/REPOSITORY.git
```
*Updates Git's saved GitHub address to include your username and token, so future pulls don't ask for a password. Replace `YOUR_GITHUB_USERNAME`, `YOUR_PERSONAL_ACCESS_TOKEN`, `OWNER`, and `REPOSITORY` with your real values.*

```bash
git pull origin main
```
*Test pull to confirm it works silently, with no login prompt. If it asks for credentials, double-check Step 3.1 and the token scope.*

---

## 🔒 Part 2: Store Server Credentials as GitHub Secrets

Secrets keep sensitive values (like server passwords/keys) out of your code. Go to your **GitHub repo** → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**, and add these four:

| Secret Name | What It Is | Example |
| --- | --- | --- |
| `SSH_HOST` | Your server's address | `example.com` or `192.168.1.1` |
| `SSH_USER` | Your cPanel username | `your_cpanel_username` |
| `SSH_PORT` | SSH port your host uses | `22` (default) or a custom port like `2222` |
| `SSH_PRIVATE_KEY` | Full contents of `server_deploy_key` (the private one, not `.pub`) | Paste the entire file including `-----BEGIN...` and `-----END...` lines |

---

## ⚙️ Part 3: The Workflow File

Create this file at `.github/workflows/deploy.yml` in your repo. Optional/advanced steps are commented out — remove the `#` to enable them later.

```yaml
name: Run Tests and Deploy

# When this workflow runs:
on:
  push:
    branches:
      - main          # Runs on every push to main
  pull_request:
    branches:
      - main          # Also runs on PRs targeting main (tests only, no deploy)

jobs:
  # ---- JOB 1: Run the test suite ----
  laravel-tests:
    runs-on: ubuntu-latest   # Use a fresh Ubuntu virtual machine

    steps:
      - name: Checkout code
        uses: actions/checkout@v4   # Downloads your repo's code onto the runner

      - name: Setup PHP
        uses: shivammathur/setup-php@v2   # Installs PHP on the runner
        with:
          php-version: '8.2'               # PHP version to use
          extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, bcmath, soap, intl, gd, exif, iconv
          # ^ PHP extensions Laravel typically needs
          coverage: none                    # Skips code coverage tools (faster runs)

      - name: Copy .env
        run: php -r "file_exists('.env') || copy('.env.example', '.env');"
        # Creates a .env config file from the example template, if one doesn't exist yet

      - name: Install Composer Dependencies
        run: composer install --no-ansi --no-interaction --no-scripts --no-progress --prefer-dist
        # Installs PHP packages listed in composer.json
        # --no-interaction: never pause for prompts
        # --prefer-dist: download pre-built packages (faster than building from source)

      - name: Generate key
        run: php artisan key:generate
        # Creates Laravel's encryption key, required for the app to run

      - name: Directory Permissions
        run: chmod -R 777 storage bootstrap/cache
        # Gives full read/write/execute permission to Laravel's storage and cache folders
        # (777 is fine for a disposable test runner; never use this on a real server)

      - name: Create Database
        run: |
          mkdir -p database
          touch database/database.sqlite
        # Creates a folder and an empty SQLite database file for the test suite to use

      - name: Execute tests
        env:
          DB_CONNECTION: sqlite
          DB_DATABASE: database/database.sqlite
          # Tells Laravel to use the SQLite file we just created for this test run
        run: php artisan test
        # Runs Laravel's test suite

  # ---- JOB 2: Deploy to the server (only after tests pass) ----
  deploy:
    runs-on: ubuntu-latest
    needs: laravel-tests                       # Waits for the tests job to succeed first
    if: github.event_name == 'push'            # Only deploys on a direct push, never on a PR

    steps:
      - name: Auto Pulling to cPanel via SSH
        uses: appleboy/ssh-action@v1.0.3   # A ready-made action for running commands over SSH
        with:
          host: ${{ secrets.SSH_HOST }}          # Pulls your server address from GitHub Secrets
          username: ${{ secrets.SSH_USER }}      # Pulls your cPanel username from Secrets
          key: ${{ secrets.SSH_PRIVATE_KEY }}    # Pulls your private SSH key from Secrets
          port: ${{ secrets.SSH_PORT }}          # Pulls your SSH port from Secrets
          script: |
            # 1. Move into your app's folder on the server
            cd /home/${{ secrets.SSH_USER }}/your-project-folder-path

            # [OPTIONAL] Put the site into maintenance mode before heavy changes
            # php artisan down --retry=60

            # 2. Pull the latest code from GitHub
            git fetch origin main
            git reset --hard origin/main
            # fetch downloads the latest commits; reset --hard forces local files
            # to exactly match them (discarding any local changes)

            # 3. Clear cached files so Laravel doesn't use outdated class maps
            rm -f bootstrap/cache/*.php

            # [OPTIONAL] Clear Laravel's config cache if settings seem stuck
            # php artisan config:clear

            # 4. Reinstall production dependencies
            composer install --no-interaction --prefer-dist --optimize-autoloader --no-dev
            # --no-dev: skip development-only packages
            # --optimize-autoloader: speeds up class loading in production

            # [OPTIONAL] Run any pending database migrations automatically
            # php artisan migrate --force

            # [OPTIONAL] Rebuild cached config/routes/views for performance
            # php artisan optimize:clear
            # php artisan optimize

            # [OPTIONAL] Take the site out of maintenance mode
            # php artisan up
```

---

## ✅ Quick Recap

1. Generate an SSH key → authorize the public half in cPanel.
2. Set the Git remote URL on the server to include a Personal Access Token.
3. Add `SSH_HOST`, `SSH_USER`, `SSH_PORT`, `SSH_PRIVATE_KEY` as GitHub Secrets.
4. Commit `.github/workflows/deploy.yml`.
5. Push to `main` — tests run, then deployment happens automatically.
