---
name: whmcs-github-actions
description: GitHub Actions for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS GitHub Actions Skill

## Overview
This skill provides patterns and implementations for configuring GitHub Actions workflows for WHMCS projects.

## Implementation Patterns

### GitHub Actions Workflow Manager
```php
<?php
/**
 * WHMCS GitHub Actions Integration
 * Manages GitHub Actions workflows
 */

namespace WHMCS\Module\DevOps\GitHub;

class GitHubActionsManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Create GitHub Actions workflow
     */
    public function createWorkflow(array $params): array {
        $workflowId = 'ghwf_' . bin2hex(random_bytes(8));

        $workflow = [
            'id' => $workflowId,
            'service_id' => $params['service_id'],
            'name' => $params['name'],
            'trigger_on' => json_encode($params['trigger_on'] ?? ['push', 'pull_request']),
            'jobs' => json_encode($params['jobs']),
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_github_actions_workflows', $workflow);

        // Generate workflow file
        $workflowFile = $this->generateWorkflowFile($workflow);

        return [
            'success' => true,
            'workflow_id' => $workflowId,
            'workflow_content' => $workflowFile
        ];
    }

    /**
     * Generate standard CI workflow
     */
    public function generateCIWorkflow(): string {
        return <<<YAML
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: mbstring, gd, mysql, zip, curl
      - name: Install dependencies
        run: composer install --no-interaction
      - name: PHP Code Sniffer
        run: vendor/bin/phpcs

  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: whmcs_test
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
        ports:
          - 3306:3306
    steps:
      - uses: actions/checkout@v3
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          extensions: mbstring, gd, mysql, zip, curl
      - name: Copy env file
        run: cp .env.example .env
      - name: Install dependencies
        run: composer install --no-interaction
      - name: Run tests
        run: vendor/bin/phpunit

  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker image
        run: docker build -t whmcs:\${{ github.sha }} .

  deploy:
    needs: [build]
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        run: echo "Deploying..."
YAML;
    }

    /**
     * Generate Docker build workflow
     */
    public function generateDockerWorkflow(): string {
        return <<<YAML
name: Docker Build & Push

on:
  push:
    branches:
      - main
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: \${{ github.actor }}
          password: \${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            ghcr.io/\${{ github.repository }}:\${{ github.sha }}
            ghcr.io/\${{ github.repository }}:latest
YAML;
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_github_actions_workflows` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `trigger_on` TEXT,
  `jobs' TEXT,
  `created_at' DATETIME NOT NULL
);
```

## Best Practices

1. **Workflow Organization**: Use separate jobs for different tasks
2. **Caching**: Cache dependencies for faster builds
3. **Matrix Builds**: Test on multiple PHP versions
4. **Environment Variables**: Use GitHub secrets
5. **Artifacts**: Use artifacts to share build outputs

## Related Skills

- whmcs-gitlab-ci
- whmcs-jenkins-pipeline
- whmcs-cicd-integration
- whmcs-artifact-storage