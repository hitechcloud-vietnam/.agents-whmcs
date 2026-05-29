# WHMCS Git-Based Development Workflow

## Overview

Establish a structured Git workflow for WHMCS development, enabling version control, collaboration, code review, and deployment automation.

## Prerequisites

- Git 2.30+
- GitHub/GitLab/Bitbucket repository
- Branch protection rules configured
- Developer access to repository

## Step-by-Step Instructions

### Step 1: Initialize Repository

```bash
# Navigate to WHMCS directory
cd /var/www/whmcs

# Initialize git repo (if new)
git init

# Configure git user
git config user.name "Developer Name"
git config user.email "dev@example.com"

# Create initial commit structure
git add .
git commit -m "Initial WHMCS installation"
```

### Step 2: Create .gitignore

Create `.gitignore`:

```gitignore
# Configuration and sensitive files
configuration.php
config一惊.yaml
.env
.env.*
*.local

# Storage and cache
/storage/*
/storage/logs/*
/storage/uploads/*
/storage/cache/*
!/storage/.gitkeep

# Templates compiled
/templates_c/*
!/templates_c/.gitkeep

# IDE and OS
.idea/
.vscode/
.DS_Store
Thumbs.db
*.swp
*~

# Dependencies
/vendor/
/node_modules/
package-lock.json

# Logs
*.log
*.sql

# Backups
*.bak
*.backup
```

### Step 3: Create Branch Strategy

```bash
# Main branches
git branch -m main
git branch develop
git branch staging

# Feature branches
git checkout -b feature/client-portal-redesign
git checkout -b feature/twilio-sms-integration
git checkout -b feature/multicurrency-support

# Bugfix branches
git checkout -b fix/payment-timeout-issue
git checkout -b fix/email-queue-stuck

# Release branches
git checkout -b release/v2.8.0
```

### Step 4: Set Up Remote Branches

```bash
# Add remote
git remote add origin git@github.com:org/whmcs-project.git

# Push branches
git push -u origin main
git push -u origin develop
git push -u origin staging

# Configure default branch
git symbolic-ref refs/remotes/origin/HEAD refs/remotes/origin/develop
```

### Step 5: Configure Pre-Commit Hooks

Create `.git/hooks/pre-commit`:

```bash
#!/bin/bash
# Pre-commit hook for WHMCS

# Check for sensitive data
if git diff --cached | grep -E "(password|secret|api_key)\s*=" | grep -v "^[-+#].*//.*password"; then
    echo "Error: Possible hardcoded credentials detected"
    exit 1
fi

# Run PHP syntax check
for file in $(git diff --cached --name-only --diff-filter=ACM | grep \.php$); do
    php -l "$file" || exit 1
done

# Check for debug code
if git diff --cached | grep -i "var_dump\|print_r\|die(" | grep -v "^[-+#].*//.*debug"; then
    echo "Warning: Debug code detected - remove before commit"
fi

exit 0
```

Make executable:

```bash
chmod +x .git/hooks/pre-commit
```

### Step 6: Create Commit Message Guidelines

Create `.git/COMMIT_CONVENTIONS.md`:

```
# WHMCS Commit Message Format

## Format
<type>(<scope>): <subject>

<body>

<footer>

## Types
- feat: New feature
- fix: Bug fix
- docs: Documentation
- style: Formatting
- refactor: Code restructuring
- test: Adding tests
- chore: Maintenance

## Examples
feat(module): Add Twilio SMS integration

- Implement SMS notification hook
- Add configuration page
- Include rate limiting

Closes #123
```

### Step 7: Implement Feature Development

```bash
# Start feature
git checkout -b feature/quick-access-dashboard develop

# Make changes
cd modules/addons/quick_dashboard
# ... implement feature ...

# Commit changes
git add .
git commit -m "feat(module): Add quick access dashboard

- Created dashboard hook
- Added widget configuration
- Implemented AJAX updates"

# Merge to develop
git checkout develop
git merge feature/quick-access-dashboard
git push origin develop

# Delete feature branch
git branch -d feature/quick-access-dashboard
```

### Step 8: Create Release Workflow

```bash
# Create release branch
git checkout -b release/v2.5.0 develop

# Update version
echo "2.5.0" > version.txt
git add version.txt
git commit -m "chore: Bump version to 2.5.0"

# Test and fix
# ... testing phase ...

# Merge to main
git checkout main
git merge release/v2.5.0 --no-ff
git tag -a v2.5.0 -m "Release v2.5.0"
git push origin main --tags

# Merge back to develop
git checkout develop
git merge release/v2.5.0 --no-ff
git push origin develop

# Delete release branch
git branch -d release/v2.5.0
```

### Step 9: Set Up GitHub Actions CI/CD

Create `.github/workflows/whmcs-ci.yml`:

```yaml
name: WHMCS CI/CD

on:
  push:
    branches: [develop, main]
  pull_request:
    branches: [develop, main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: whmcs_test
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
        ports:
          - 3306:3306

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: pdo, pdo_mysql, mbstring, xml, gd
          coverage: xdebug

      - name: Install Dependencies
        run: composer install --prefer-dist

      - name: Run Tests
        run: vendor/bin/phpunit

      - name: PHP Syntax Check
        run: |
          find modules -name "*.php" -exec php -l {} \;
          find templates -name "*.tpl" -exec php -l {} \;

      - name: Security Scan
        run: |
          git diff --cached --name-only | xargs -I{} php -l {} 2>/dev/null || true

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /var/www/whmcs
            git pull origin main
            composer install --no-dev --optimize-autoloader
            php artisan cache:clear
```

## Git Workflow Diagram

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    main     │◄────│   staging   │◄────│   develop   │
│  (releases) │     │  (testing)  │     │ (main dev) │
└─────────────┘     └─────────────┘     └─────────────┘
       ▲                                        │
       │                                        ▼
       │                                  ┌─────────────┐
       └──────────────────────────────────│   feature/* │
                                          │   fix/*     │
                                          └─────────────┘
```

## Expected Outcomes

- Organized Git repository with clear branch structure
- Automated pre-commit validation
- Consistent commit message format
- Protected main branch with code review requirements
- Automated testing on push/pull request
- Rollback capability via git revert/tag

## Testing Checklist

- [ ] All branches created according to strategy
- [ ] Pre-commit hooks install and execute
- [ ] PHP syntax validation runs on staged files
- [ ] CI/CD pipeline triggers on push
- [ ] Feature branches merge cleanly to develop
- [ ] Release workflow completes without conflicts
- [ ] Git tags created correctly
- [ ] Sensitive files excluded from repository
- [ ] Code review process functional
- [ ] Deployment automation successful
