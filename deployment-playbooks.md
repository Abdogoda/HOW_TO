# 🚀 Automated CI/CD Pipeline (GitHub Actions to cPanel)

This repository includes an automated testing and deployment workflow using GitHub Actions. Every push or pull request runs the full Laravel test suite. Upon a successful merge or direct push to the `main` branch, the pipeline securely logs into the cPanel server via SSH to perform an auto-pull and dependency installation.

---

## 🛠️ One-Time Server Setup

### 1. Generate & Authorize SSH Keys
GitHub Actions logs into cPanel via SSH authentication.
1. Open your local terminal or Git Bash and generate a key pair:
   ```bash
   ssh-keygen -t rsa -b 4096 -f ./cpanel_github_key -N ""

2. Log into **cPanel** → Navigate to **Security** → **SSH Access** → **Manage SSH Keys**.
3. Click **Import Key** and paste the text from `cpanel_github_key.pub` (Public Key) into the box.
4. **Critical:** Return to the SSH keys list, click **Manage** next to the newly imported key, and click **Authorize**.

### 2. Configure Git URL with Personal Access Token (PAT)

To bypass repository administrative constraints (such as missing permission to add repository-level Deploy Keys), embed a GitHub Personal Access Token into cPanel's local repository configuration.

1. Generate a **Personal Access Token (Classic)** on GitHub with full `repo` scopes.
2. Open the cPanel **Terminal** application, navigate to the target directory, and update the origin remote URL:
```bash
cd /home/muslimacademy/dashboard.muslim-academy.net
git remote set-url origin [https://YOUR_GITHUB_USERNAME:YOUR_PERSONAL_ACCESS_TOKEN@github.com/Brmja-Tech/Muslim-Academy-backend.git](https://YOUR_GITHUB_USERNAME:YOUR_PERSONAL_ACCESS_TOKEN@github.com/Brmja-Tech/Muslim-Academy-backend.git)

```


3. Run a manual `git pull origin main` once to confirm the remote authentication works seamlessly.

---

## 🔒 Configuration of GitHub Secrets

To make the pipeline work securely, add these parameters to your repository environment. Navigate to your **GitHub Repository** → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**:

| Secret Name | Description | Example / Value |
| --- | --- | --- |
| `SSH_HOST` | Your production server IP or hostname | `dashboard.muslim-academy.net` |
| `SSH_USER` | Your administrative cPanel username | `muslimacademy` |
| `SSH_PORT` | The port your server uses for SSH | `22` (or custom provider port like `2222`) |
| `SSH_PRIVATE_KEY` | The raw contents of your secret key file | *Copy everything inside `cpanel_github_key*` |

---

## ⚙️ CI/CD Workflow Configuration

The active workflow config file is located at `.github/workflows/laravel.yml`. Below is the blueprint containing optional modular commands commented out for future scalability (e.g., automated zero-downtime maintenance configurations or production schema updates).

```yaml
name: Run Tests and Deploy

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  laravel-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, bcmath, soap, intl, gd, exif, iconv
          coverage: none

      - name: Copy .env
        run: php -r "file_exists('.env') || copy('.env.example', '.env');"

      - name: Install Composer Dependencies
        run: composer install --no-ansi --no-interaction --no-scripts --no-progress --prefer-dist

      - name: Generate key
        run: php artisan key:generate

      - name: Directory Permissions
        run: chmod -R 777 storage bootstrap/cache

      - name: Create Database
        run: |
          mkdir -p database
          touch database/database.sqlite

      - name: Execute tests
        env:
          DB_CONNECTION: sqlite
          DB_DATABASE: database/database.sqlite
        run: php artisan test

  deploy:
    runs-on: ubuntu-latest
    needs: laravel-tests
    if: github.event_name == 'push' # Prevents deployment running on open PRs; triggers only on merge to main

    steps:
      - name: Auto Pulling to cPanel via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: ${{ secrets.SSH_PORT }}
          script: |
            # 1. Access the working directory
            cd /home/muslimacademy/dashboard.muslim-academy.net
            
            # [OPTIONAL] Activate maintenance mode during heavy database migrations
            # php artisan down --retry=60
            
            # 2. Update the codebase
            git fetch origin main
            git reset --hard origin/main
            
            # 3. Clean environment state to ensure smooth class lookup and package discovery
            rm -f bootstrap/cache/*.php
            
            # [OPTIONAL] Reset deep server configuration state manually if packages get locked
            # php artisan config:clear
            
            # 4. Fetch updated distribution packages
            composer install --no-interaction --prefer-dist --optimize-autoloader --no-dev
            
            # [OPTIONAL] Automatically run schema migrations in production environments
            # php artisan migrate --force
            
            # [OPTIONAL] Rebuild production environment file caching mechanisms 
            # php artisan optimize:clear
            # php artisan optimize
            
            # [OPTIONAL] Deactivate maintenance mode and release application live
            # php artisan up
