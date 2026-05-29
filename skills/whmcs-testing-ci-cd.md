# WHMCS Testing - CI/CD Pipeline

## Skill Description
Set up CI/CD pipelines for WHMCS modules to automate testing, deployment, and quality assurance.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- GitHub Actions or GitLab CI
- CI/CD knowledge

## Step-by-Step Implementation

### 1. GitHub Actions Workflow
```yaml
# .github/workflows/ci.yml

name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: whmcs_test
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          extensions: pdo_mysql, redis
          coverage: xdebug

      - name: Install Dependencies
        run: composer install --no-interaction

      - name: Run Tests
        run: |
          cp phpunit.xml.dist phpunit.xml
          ./vendor/bin/phpunit --coverage-text

      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml

  lint:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          tools: phpcs, phpstan

      - name: Install Dependencies
        run: composer install --no-interaction

      - name: Run PHPCS
        run: ./vendor/bin/phpcs --standard=PSR12 src

      - name: Run PHPStan
        run: ./vendor/bin/phpstan analyse src --level=5

  deploy:
    needs: [test, lint]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v3

      - name: Deploy
        run: |
          echo "Deploying to production..."
          # Add deployment commands here
```

### 2. GitLab CI Configuration
```yaml
# .gitlab-ci.yml

stages:
  - test
  - lint
  - deploy

variables:
  MYSQL_DATABASE: whmcs_test
  MYSQL_ROOT_PASSWORD: root

test:
  stage: test
  image: php:8.1
  services:
    - mysql:8.0
  before_script:
    - apt-get update && apt-get install -y git unzip libpdo-mysql
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - composer install --no-interaction
  script:
    - cp phpunit.xml.dist phpunit.xml
    - ./vendor/bin/phpunit --coverage-text
  coverage: '/^Lines:\s+(\d+\.\d+)%/'

phpcs:
  stage: lint
  image: php:8.1
  before_script:
    - apt-get update && apt-get install -y git unzip
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - composer install --no-interaction
  script:
    - ./vendor/bin/phpcs --standard=PSR12 src

phpstan:
  stage: lint
  image: php:8.1
  before_script:
    - apt-get update && apt-get install -y git unzip
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - composer install --no-interaction
  script:
    - ./vendor/bin/phpstan analyse src --level=5

deploy:
  stage: deploy
  only:
    - main
  script:
    - echo "Deploying..."
```

### 3. Composer Scripts
```json
{
    "scripts": {
        "test": "phpunit",
        "test-coverage": "phpunit --coverage-html coverage",
        "lint": "phpcs --standard=PSR12 src",
        "lint-fix": "phpcbf --standard=PSR12 src",
        "analyse": "phpstan analyse src --level=5",
        "quality": [
            "@lint",
            "@analyse",
            "@test"
        ]
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Slow pipelines | Cache dependencies |
| Flaky tests | Fix test stability |
| Missing coverage | Add more tests |

## Testing Checklist

- [ ] Configure CI workflow
- [ ] Set up test database
- [ ] Configure linting
- [ ] Set up deployment
- [ ] Monitor pipeline health

## Reference Links

- [GitHub Actions](https://docs.github.com/en/actions)
- [GitLab CI/CD](https://docs.gitlab.com/ee/ci/)
- [PHPUnit CI Integration](https://phpunit.readthedocs.io/en/9.5/textui.html)
