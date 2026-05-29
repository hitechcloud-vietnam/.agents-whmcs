# WHMCS Code Review Process

## Overview
This workflow defines the code review process for WHMCS modules and customizations, ensuring code quality, security, and consistency across projects.

## Review Stages

### Stage 1: Self-Review (Before Submitting)

Before requesting a formal review, perform a self-review:

```bash
# Check your changes
git diff --stat
git diff modules/addons/your_module/

# Run linter
./vendor/bin/phpcs --standard=PSR12 modules/addons/your_module/

# Run tests
./vendor/bin/phpunit

# Check for common issues
./vendor/bin/phpstan analyse modules/addons/your_module/
```

### Stage 2: Automated Checks

All pull requests must pass automated checks:

```yaml
# .github/workflows/review.yml
name: Code Quality

on:
  pull_request:
    paths:
      - 'modules/**'

jobs:
  code-quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'

      - name: Install dependencies
        run: composer install

      - name: PHP CodeSniffer
        run: |
          ./vendor/bin/phpcs --standard=PSR12 \
            --colors \
            --extensions=php \
            modules/

      - name: PHPStan
        run: |
          ./vendor/bin/phpstan analyse \
            modules/ \
            --level=max \
            --no-progress

      - name: Psalm
        run: |
          ./vendor/bin/psalm \
            --show-info=false

      - name: Run Tests
        run: |
          ./vendor/bin/phpunit \
            --coverage-clover=coverage.xml

      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
```

### Stage 3: Manual Code Review

## Review Checklist

### Code Style & Standards
- [ ] Follows PSR-12 coding standard
- [ ] Consistent indentation (4 spaces)
- [ ] Descriptive variable and function names
- [ ] No commented-out code
- [ ] Proper use of whitespace

### Security Review
- [ ] All user input sanitized and validated
- [ ] SQL queries use prepared statements
- [ ] No sensitive data in logs
- [ ] Proper permission checks
- [ ] CSRF protection implemented
- [ ] Output properly escaped

### WHMCS Best Practices
- [ ] Uses WHMCS database capsule when possible
- [ ] Follows WHMCS hook system properly
- [ ] No direct database queries when Capsule available
- [ ] Proper use of Lang class for strings
- [ ] Template files properly structured
- [ ] Asset enqueueing follows WHMCS patterns

### Error Handling
- [ ] Exceptions properly caught
- [ ] User-friendly error messages
- [ ] Errors logged appropriately
- [ ] No exposed stack traces
- [ ] Graceful degradation

### Performance
- [ ] Database queries optimized
- [ ] No N+1 query patterns
- [ ] Caching implemented where appropriate
- [ ] Large operations use chunking
- [ ] No unnecessary loops

### Testing
- [ ] Unit tests for new functionality
- [ ] Edge cases covered
- [ ] Mock external dependencies
- [ ] Integration tests for workflows

## Code Review Comments Template

### Blocking Issues (Must Fix)
```
[BLOCKING] Security: {description}
- Found: {location}
- Impact: {what could happen}
- Suggestion: {how to fix}
```

```
[BLOCKING] Error Handling: {description}
- Found: {location}
- Issue: {what's wrong}
- Suggestion: {how to fix}
```

### Non-Blocking Suggestions
```
[SUGGESTION] Code Style: {description}
- Consider: {suggestion}
- Optional improvement
```

```
[NIT] Minor: {description}
- Nitpicky suggestion
```

### Questions
```
[QUESTION] Understanding: {question}
- I'm not sure about...
- Can you clarify?
```

## Review Process

### Step 1: Clone and Setup

```bash
# Clone the repository
git clone https://github.com/org/repo.git
cd repo

# Install dependencies
composer install

# Checkout the PR branch
git checkout feature/your-feature

# Set up pre-commit hooks
composer run-script post-install-cmd
```

### Step 2: Review Changes

```bash
# View the diff
git diff main..feature/your-feature

# View specific files
git diff main..feature/your-feature -- modules/addons/your_module/

# Check commit history
git log main..feature/your-feature --oneline

# Run static analysis
./vendor/bin/phpstan analyse modules/addons/your_module/

# Run tests
./vendor/bin/phpunit
```

### Step 3: Test the Changes

```php
<?php
// Manual test script
// test_changes.php

require_once __DIR__ . '/includes/init.php';

// Test activation
$result = your_module_activate();
echo "Activation: " . $result['status'] . "\n";

// Test configuration save
$config = your_module_config();
echo "Config fields: " . count($config['fields']) . "\n";

// Test functionality
$service = new YourModule\Service\ModuleService();
$result = $service->process(['test' => 'data']);
echo "Processing result: " . json_encode($result) . "\n";
```

### Step 4: Leave Feedback

```markdown
## Code Review: Your Module Feature

### Overall Assessment
{Give an overall summary of the PR quality}

### Changes Overview
- {List major changes}
- {Be specific about what was done}

### What's Good
- {Positive feedback}
- {Specific praise for good implementation}

### Issues to Address

#### [BLOCKING] SQL Injection Vulnerability
**File:** `src/Service/ModuleService.php:45`
```php
// Found:
$query = "SELECT * FROM table WHERE id = $id";

// Should be:
$query = "SELECT * FROM table WHERE id = ?";
$result = Capsule::select($query, [$id]);
```
**Impact:** User-controlled input could manipulate the query
**Fix:** Use parameterized queries

#### [BLOCKING] Missing Permission Check
**File:** `src/Controller/AdminController.php:78`
**Issue:** Admin function called without verify csrf token
**Fix:** Add `AdminAuth::getInstance()->validateAuthToken()`

### Suggestions

#### [SUGGESTION] Error Handling
**File:** `src/Service/ModuleService.php:102`
Consider wrapping the API call in a try-catch block to provide better error messages.

#### [NIT] Code Style
**File:** `src/Helper/Logger.php:15`
Variable name `$dt` could be more descriptive. Consider `$dateTime`.

### Testing Notes
- Ran unit tests: All passing
- Manual testing: Module activates correctly
- Security scan: Found 2 blocking issues

### Approval Status
**Request changes** - Please address the blocking issues before merging.
```

## Reviewer Guidelines

### Be Constructive
- Focus on the code, not the person
- Explain the "why" behind suggestions
- Offer alternatives when suggesting changes

### Be Timely
- Aim to review within 24 hours
- If you can't review, let the author know
- Set expectations for complex reviews

### Be Thorough
- Review all files, not just the main ones
- Test the functionality when possible
- Consider edge cases and error scenarios

## Author Response Guidelines

### Addressing Feedback
```bash
# Create fix branch
git checkout -b fix/address-review-feedback

# Make changes
git commit -am "Address review feedback"

# Push and notify
git push origin fix/address-review-feedback
```

### Response Template
```markdown
## Response to Code Review

### Addressed Issues

#### [FIXED] SQL Injection Vulnerability
- Added prepared statement
- Verified with PHPUnit tests
- Commit: abc123

#### [FIXED] Missing Permission Check
- Added CSRF token validation
- Added unit test for the check
- Commit: def456

### Clarifications

#### Regarding {question}
{Provide explanation}
```

## Verification Checklist

- [ ] Self-review completed
- [ ] All automated checks passing
- [ ] Security issues addressed
- [ ] Code style issues addressed
- [ ] Error handling reviewed
- [ ] Performance considerations addressed
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] Review feedback addressed
- [ ] Re-review requested
