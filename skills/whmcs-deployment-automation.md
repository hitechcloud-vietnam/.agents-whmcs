# WHMCS Deployment Automation

## Skill Description
Implement automated deployment workflows for WHMCS modules using deployment tools and scripts.

## Prerequisites
- Deployment server
- CI/CD tools
- SSH access

## Step-by-Step Implementation

### 1. Deployment Automation Script
```php
<?php
// scripts/deploy.php

namespace WHMCS\Module\YourModule\Deployment;

class Deployer
{
    private string $environment;
    private string $deployPath;
    private array $steps = [];

    public function __construct(string $environment = 'staging')
    {
        $this->environment = $environment;
        $this->deployPath = $this->getDeployPath();
    }

    public function addStep(callable $step, string $description): self
    {
        $this->steps[] = [
            'step' => $step,
            'description' => $description
        ];

        return $this;
    }

    public function deploy(): array
    {
        $results = [];
        $startTime = microtime(true);

        foreach ($this->steps as $index => $step) {
            $stepStart = microtime(true);

            try {
                call_user_func($step['step']);
                $results[] = [
                    'step' => $step['description'],
                    'status' => 'success',
                    'duration' => microtime(true) - $stepStart
                ];
            } catch (\Exception $e) {
                $results[] = [
                    'step' => $step['description'],
                    'status' => 'failed',
                    'error' => $e->getMessage(),
                    'duration' => microtime(true) - $stepStart
                ];

                $this->rollback();
                break;
            }
        }

        return [
            'environment' => $this->environment,
            'success' => end($results)['status'] === 'success',
            'steps' => $results,
            'total_duration' => microtime(true) - $startTime
        ];
    }

    private function rollback(): void
    {
        // Rollback logic
        echo "Rolling back deployment...\n";
    }

    private function getDeployPath(): string
    {
        return match ($this->environment) {
            'production' => '/var/www/whmcs/modules/yourmodule',
            'staging' => '/var/www/staging/whmcs/modules/yourmodule',
            'local' => dirname(__DIR__),
        };
    }

    public static function createDeployment(string $environment): self
    {
        $deployer = new self($environment);

        $deployer->addStep(function () {
            echo "Pulling latest code...\n";
        }, 'Pull latest code');

        $deployer->addStep(function () {
            echo "Installing dependencies...\n";
        }, 'Install dependencies');

        $deployer->addStep(function () {
            echo "Running migrations...\n";
        }, 'Run migrations');

        $deployer->addStep(function () {
            echo "Clearing cache...\n";
        }, 'Clear cache');

        return $deployer;
    }
}
```

### 2. Deployment Configuration
```yaml
# deploy.yaml

environments:
  production:
    host: prod.whmcs.com
    path: /var/www/whmcs/modules/yourmodule
    branch: main
    before_deploy:
      - backup
      - notify
    after_deploy:
      - migrate
      - cache_clear
      - notify

  staging:
    host: staging.whmcs.com
    path: /var/www/staging/whmcs/modules/yourmodule
    branch: develop
    before_deploy:
      - notify
    after_deploy:
      - migrate
      - cache_clear
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Deployment failures | Implement rollback |
| Downtime | Use blue-green deployment |
| Configuration drift | Use config management |

## Security Considerations

1. **Secure credentials** - Use secrets management
2. **Audit deployments** - Log all deployments
3. **Limit access** - Restrict deployment permissions

## Testing Checklist

- [ ] Test deployment script
- [ ] Test rollback
- [ ] Verify notifications

## Reference Links

- [Deployment Automation](https://www.atlassian.com/continuous-delivery/continuous-delivery-workflows)
