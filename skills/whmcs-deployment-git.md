# WHMCS Git Deployment

## Skill Description
Set up Git-based deployment workflows for WHMCS modules to manage version control, branching strategies, and automated deployments.

## Prerequisites
- Git installed
- SSH access to server
- Basic Git knowledge

## Step-by-Step Implementation

### 1. Git Repository Structure
```
your-module/
├── .git/
├── src/
│   ├── controllers/
│   ├── models/
│   └── services/
├── tests/
├── hooks/
├── templates/
├── migrations/
├── config.php
├── composer.json
├── phpunit.xml
└── README.md
```

### 2. Gitignore
```
# Dependencies
/vendor/

# IDE
.idea/
.vscode/
*.swp
*.swo

# Environment
.env
.env.local
.env.*.local

# Cache
/cache/
/storage/
/logs/

# Test
/coverage/

# OS
.DS_Store
Thumbs.db

# Build
*.zip
*.tar.gz
```

### 3. Deployment Script
```bash
#!/bin/bash
# deploy.sh

set -e

# Configuration
REMOTE_HOST="your-server.com"
REMOTE_USER="whmcs"
REMOTE_PATH="/var/www/whmcs/modules/yourmodule"
BRANCH="main"

# Colors
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m'

echo -e "${GREEN}Starting deployment...${NC}"

# Pull latest changes
git pull origin $BRANCH

# Install dependencies
composer install --no-dev --optimize-autoloader

# Run migrations
php artisan migrate --force

# Clear cache
php artisan cache:clear
php artisan config:clear
php artisan view:clear

# Run tests
if [ -f "phpunit.xml" ]; then
    ./vendor/bin/phpunit --no-coverage
fi

# Commit version bump
git add -A
git commit -m "Deploy $(git rev-parse --short HEAD)"

# Push to remote
git push origin $BRANCH

echo -e "${GREEN}Deployment complete!${NC}"
```

### 4. Post-Receive Hook
```bash
#!/bin/bash
# post-receive

GIT_DIR="/var/repos/your-module.git"
WHMCS_PATH="/var/www/whmcs/modules/yourmodule"

while read oldrev newrev refname; do
    branch=$(echo $refname | cut -d/ -f3)

    if [ "$branch" = "main" ]; then
        echo "Deploying main branch..."

        cd $WHMCS_PATH
        git pull origin main

        # Run post-deployment tasks
        composer install --no-dev --optimize-autoloader
        php artisan migrate --force
        php artisan cache:clear
    fi
done
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Deployment failures | Use rollback scripts |
| Missing dependencies | Verify composer install |
| Permission issues | Set correct ownership |

## Security Considerations

1. **Use SSH keys** - Don't use password authentication
2. **Protect .git** - Block web access to .git
3. **Verify signatures** - Verify commit signatures
4. **Limit deploy keys** - Use deploy-only keys

## Testing Checklist

- [ ] Test local deployment
- [ ] Test remote deployment
- [ ] Verify permissions
- [ ] Test rollback

## Reference Links

- [Git Deployment](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)
