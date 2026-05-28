# WHMCS Deployment Strategy Workflow

## Purpose

Comprehensive guide to deploying WHMCS in various environments, including development, staging, and production, with proper rollback procedures and zero-downtime deployment strategies.

## Prerequisites

- WHMCS installation
- Server access (SSH)
- CI/CD infrastructure
- Deployment automation tools
- Version control system

## Workflow Steps

### Step 1: Environment Architecture

Define deployment environments:

```yaml
# deployment/environments.yml

environments:
  development:
    url: http://whmcs-dev.local
    branch: develop
    auto_deploy: true
    debug_mode: true
    
  staging:
    url: https://whmcs-staging.example.com
    branch: staging
    auto_deploy: false
    approval_required: true
    
  production:
    url: https://whmcs.example.com
    branch: main
    auto_deploy: false
    approval_required: true
    blue_green: true

database:
  development: whmcs_dev
  staging: whmcs_staging
  production: whmcs_prod
  
backup:
  before_deploy: true
  retention_days: 14
  remote_backup: true
```

### Step 2: Deployment Script Creation

Create deployment automation scripts:

```bash
#!/bin/bash
# scripts/deploy.sh - Main deployment script

set -euo pipefail

ENVIRONMENT=${1:-staging}
VERSION=${2:-$(git rev-parse --short HEAD)}
DEPLOY_DIR="/var/www/whmcs-${ENVIRONMENT}"
LOG_FILE="/var/log/deployments/whmcs-${ENVIRONMENT}-$(date +%Y%m%d).log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

error() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR: $1" | tee -a "$LOG_FILE" >&2
    exit 1
}

log "Starting deployment to ${ENVIRONMENT}"
log "Version: ${VERSION}"

# Pre-deployment checks
log "Running pre-deployment checks..."
./scripts/pre-deploy-check.sh "$ENVIRONMENT" || error "Pre-deployment check failed"

# Create backup
log "Creating backup..."
./scripts/backup.sh "$ENVIRONMENT" || error "Backup failed"

# Pull latest code
log "Pulling latest code..."
cd "$DEPLOY_DIR"
git fetch origin
git checkout "$VERSION" || error "Checkout failed"
git pull origin "$(git rev-parse --abbrev-ref HEAD)" || error "Pull failed"

# Run database migrations
log "Running database migrations..."
./scripts/run-migrations.sh "$ENVIRONMENT" || error "Migration failed"

# Clear cache
log "Clearing cache..."
php artisan cache:clear || error "Cache clear failed"

# Update permissions
log "Updating file permissions..."
./scripts/update-permissions.sh "$ENVIRONMENT" || error "Permission update failed"

# Run health checks
log "Running health checks..."
./scripts/health-check.sh "$ENVIRONMENT" || error "Health check failed"

# Notify completion
log "Deployment completed successfully"
./scripts/notify-success.sh "$ENVIRONMENT" "$VERSION"
```

```bash
#!/bin/bash
# scripts/pre-deploy-check.sh

ENVIRONMENT=$1

# Check disk space
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -gt 85 ]; then
    echo "ERROR: Disk usage at ${DISK_USAGE}%"
    exit 1
fi

# Check memory
MEMORY_AVAILABLE=$(free -m | awk 'NR==2 {print $7}')
if [ "$MEMORY_AVAILABLE" -lt 512 ]; then
    echo "ERROR: Low memory available: ${MEMORY_AVAILABLE}MB"
    exit 1
fi

# Check database connection
php -r "
require '/var/www/whmcs-${ENVIRONMENT}/includes/init.php';
echo 'Database connection OK';
" || exit 1

# Verify required directories
for dir in storage templates_c downloads; do
    if [ ! -d "/var/www/whmcs-${ENVIRONMENT}/${dir}" ]; then
        echo "ERROR: Required directory ${dir} does not exist"
        exit 1
    fi
done

echo "All pre-deployment checks passed"
exit 0
```

### Step 3: Database Migration Strategy

Handle database changes safely:

```php
// includes/MigrationManager.php

class MigrationManager
{
    private $db;
    private $migrationsTable = 'schema_migrations';
    
    public function __construct()
    {
        $this->db = Capsule::connection()->getPdo();
        $this->ensureMigrationsTable();
    }
    
    /**
     * Ensure migrations tracking table exists
     */
    private function ensureMigrationsTable(): void
    {
        $sql = "CREATE TABLE IF NOT EXISTS {$this->migrationsTable} (
            id INT AUTO_INCREMENT PRIMARY KEY,
            migration VARCHAR(255) NOT NULL,
            batch INT NOT NULL,
            executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            UNIQUE KEY unique_migration (migration)
        )";
        
        $this->db->exec($sql);
    }
    
    /**
     * Run pending migrations
     */
    public function runPendingMigrations(): array
    {
        $pending = $this->getPendingMigrations();
        $results = ['run' => 0, 'failed' => 0, 'errors' => []];
        
        foreach ($pending as $migration) {
            try {
                $this->runMigration($migration);
                $results['run']++;
            } catch (Exception $e) {
                $results['failed']++;
                $results['errors'][] = [
                    'migration' => $migration['file'],
                    'error' => $e->getMessage(),
                ];
                
                if ($this->shouldRollback($e)) {
                    $this->rollbackMigration($migration);
                }
            }
        }
        
        return $results;
    }
    
    /**
     * Run single migration
     */
    private function runMigration(array $migration): void
    {
        logActivity("Running migration: {$migration['file']}");
        
        // Start transaction
        $this->db->beginTransaction();
        
        try {
            // Include migration file
            require_once $migration['path'];
            
            // Get class name from file
            $className = $this->getMigrationClassName($migration['file']);
            
            // Instantiate and run
            $migrationInstance = new $className();
            $migrationInstance->up();
            
            // Record migration
            $stmt = $this->db->prepare(
                "INSERT INTO {$this->migrationsTable} (migration, batch) VALUES (?, ?)"
            );
            $stmt->execute([$migration['file'], $this->getCurrentBatch()]);
            
            $this->db->commit();
            
            logActivity("Migration completed: {$migration['file']}");
        } catch (Exception $e) {
            $this->db->rollBack();
            throw $e;
        }
    }
    
    /**
     * Create new migration
     */
    public static function createMigration(string $name): string
    {
        $timestamp = date('Y_m_d_His');
        $filename = "{$timestamp}_{$name}.php";
        $path = __DIR__ . '/migrations/' . $filename;
        
        $content = <<<PHP
<?php

use WHMCS\Database\SchemaMigration;

class {$timestamp}_{$name}
{
    public function up(SchemaMigration \$schema): void
    {
        // Add your migration code here
    }
    
    public function down(SchemaMigration \$schema): void
    {
        // Add rollback code here
    }
}
PHP;
        
        file_put_contents($path, $content);
        
        return $path;
    }
}

/**
 * Example migration file
 */
class Migration_2026_05_28_AddCustomTable implements SchemaMigration
{
    public function up(SchemaMigration $schema): void
    {
        $schema->createTable('mod_custom_module_data', function($table) {
            $table->id();
            $table->string('name');
            $table->text('data')->nullable();
            $table->integer('user_id')->nullable();
            $table->timestamp('created_at')->useCurrent();
            $table->timestamp('updated_at')->useCurrent();
            
            $table->index('user_id');
        });
    }
    
    public function down(SchemaMigration $schema): void
    {
        $schema->dropTable('mod_custom_module_data');
    }
}
```

### Step 4: Blue-Green Deployment

Implement zero-downtime deployment:

```yaml
# docker-compose.blue-green.yml

version: '3.8'

services:
  whmcs-blue:
    image: whmcs:latest
    environment:
      - ENVIRONMENT=blue
    networks:
      - whmcs-network

  whmcs-green:
    image: whmcs:latest
    environment:
      - ENVIRONMENT=green
    networks:
      - whmcs-network
    deploy:
      mode: replicated
      replicas: 0  # Start with 0 replicas

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - whmcs-blue
      - whmcs-green
    networks:
      - whmcs-network

networks:
  whmcs-network:
    driver: bridge
```

```nginx
# nginx.conf - Blue-Green switching

upstream whmcs_backend {
    server whmcs-blue:80;
}

server {
    listen 80;
    server_name whmcs.example.com;
    
    # Health check endpoint
    location /health {
        proxy_pass http://whmcs_backend;
        proxy_connect_timeout 2s;
        proxy_next_upstream error timeout;
        
        # Only return 200 if backend is healthy
        proxy_intercept_errors off;
    }
    
    # Main application
    location / {
        proxy_pass http://whmcs_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Timeout settings
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

```bash
#!/bin/bash
# scripts/blue-green-switch.sh

set -e

CURRENT="${1:-blue}"
NEW="${2:-green}"

if [ "$CURRENT" = "blue" ]; then
    NEW="green"
else
    NEW="blue"
fi

echo "Switching from $CURRENT to $NEW..."

# Deploy to new environment
docker-compose up -d whmcs-$NEW

# Wait for new environment to be ready
echo "Waiting for $NEW to be ready..."
for i in {1..30}; do
    if curl -sf "http://whmcs-$NEW/health" > /dev/null; then
        echo "$NEW is ready"
        break
    fi
    sleep 2
done

# Switch nginx upstream
sed -i "s/whmcs-$CURRENT/whmcs-$NEW/g" /etc/nginx/nginx.conf

# Reload nginx
nginx -s reload

# Keep old environment running briefly for rollback
sleep 30

# If no issues, stop old environment
docker-compose stop whmcs-$CURRENT

echo "Switched to $NEW successfully"
```

### Step 5: Rollback Procedures

Implement safe rollback procedures:

```bash
#!/bin/bash
# scripts/rollback.sh

set -e

ENVIRONMENT=$1
VERSION=${2:-previous}
DEPLOY_DIR="/var/www/whmcs-${ENVIRONMENT}"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] ROLLBACK: $1"
}

# Validate rollback
if [ ! -d "$DEPLOY_DIR" ]; then
    echo "ERROR: Deployment directory not found"
    exit 1
fi

log "Starting rollback to ${VERSION}"

# Check if version exists
cd "$DEPLOY_DIR"
if ! git rev-parse "$VERSION" > /dev/null 2>&1; then
    echo "ERROR: Version ${VERSION} not found"
    exit 1
fi

# Get previous successful deployment
if [ "$VERSION" = "previous" ]; then
    VERSION=$(git describe --tags --abbrev=0 HEAD^)
fi

log "Rolling back to ${VERSION}"

# Stop maintenance mode
rm -f "${DEPLOY_DIR}/includes/maintenance.php"

# Checkout previous version
git checkout "$VERSION"

# Restore database from backup (if needed)
read -p "Restore database from backup? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    ./scripts/restore-database.sh "$ENVIRONMENT"
fi

# Clear cache
php artisan cache:clear

# Update permissions
./scripts/update-permissions.sh "$ENVIRONMENT"

# Verify rollback
./scripts/health-check.sh "$ENVIRONMENT"

log "Rollback completed successfully"
```

### Step 6: CI/CD Pipeline

Configure continuous deployment:

```yaml
# .github/workflows/deploy.yml

name: WHMCS Deployment

on:
  push:
    branches: [main, staging, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          extensions: mysql, curl, gd, mbstring, xml, zip
          
      - name: Install dependencies
        run: composer install --no-interaction
        
      - name: Run tests
        run: |
          cp phpunit.xml.dist phpunit.xml
          ./vendor/bin/phpunit --coverage-text
          
      - name: Run linting
        run: composer lint

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/staging'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to staging
        run: |
          ./scripts/deploy.sh staging ${{ github.sha }}
          
      - name: Run smoke tests
        run: ./scripts/smoke-tests.sh staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    environment:
      name: production
      url: https://whmcs.example.com
      
    steps:
      - uses: actions/checkout@v3
      
      - name: Require approval
        run: echo "Awaiting manual approval"
        
      - name: Deploy to production
        run: |
          ./scripts/deploy.sh production ${{ github.sha }}
          
      - name: Verify deployment
        run: |
          ./scripts/health-check.sh production
          ./scripts/run-e2e-tests.sh production
```

## Best Practices

1. **Automate everything** - Remove manual steps
2. **Test in CI** - Don't deploy untested code
3. **Blue-green deployments** - Zero downtime
4. **Feature flags** - Control feature releases
5. **Database migrations** - Safe, reversible changes
6. **Health checks** - Verify after deployment
7. **Rollback plan** - Always have a way back
8. **Monitoring** - Watch for issues after deploy

## Common Pitfalls to Avoid

1. **Manual deployments** - Human error is inevitable
2. **No testing** - Deploying untested code
3. **No rollback plan** - Stuck with bad deployment
4. **Database locks** - Long-running migrations
5. **Permission issues** - Wrong file ownership
6. **Cache issues** - Not clearing caches
7. **Environment differences** - Works locally, fails in prod
8. **No monitoring** - Don't know something went wrong
