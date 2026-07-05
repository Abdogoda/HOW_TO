# 🧪 CI Pipeline: Automated Testing for a Laravel App (GitHub Actions)

This guide sets up a GitHub Actions workflow that automatically runs your Laravel test suite on every push and pull request. **No deployment is included** — this is testing only.

**Who this is for:** Absolute beginners. Just add the file below to your repo.

---

## 📋 Prerequisite

- A GitHub repository containing your Laravel app.

That's it — no server, SSH, or secrets needed for testing.

---

## ⚙️ The Workflow File

Create this file at `.github/workflows/tests.yml`.

```yaml
name: Run Laravel Tests

# When this workflow runs:
on:
  push:
    branches:
      - main          # Runs on every push to main
  pull_request:
    branches:
      - main          # Also runs on any PR targeting main

jobs:
  laravel-tests:
    runs-on: ubuntu-latest   # Use a fresh Ubuntu virtual machine

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        # Downloads your repo's code onto the runner

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        # Installs PHP on the runner
        with:
          php-version: '8.2'   # PHP version to use
          extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, bcmath, soap, intl, gd, exif, iconv
          # ^ PHP extensions Laravel typically needs
          coverage: none
          # Skips code coverage tools (faster runs)

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
        # (fine for a disposable test runner)

      - name: Create Database
        run: |
          mkdir -p database
          touch database/database.sqlite
        # Creates a folder and an empty SQLite file for the tests to use as a database

      - name: Execute tests
        env:
          DB_CONNECTION: sqlite
          DB_DATABASE: database/database.sqlite
          # Tells Laravel to use the SQLite file we just created for this test run
        run: php artisan test
        # Runs Laravel's test suite
```

---

## ✅ Quick Recap

1. Add `.github/workflows/tests.yml` with the content above.
2. Commit and push.
3. GitHub Actions runs tests automatically on every push/PR to `main`.
