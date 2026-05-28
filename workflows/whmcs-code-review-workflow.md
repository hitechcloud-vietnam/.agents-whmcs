# WHMCS Code Review Workflow

## Purpose

Establish a standardized process for reviewing custom WHMCS code, modules, and modifications. Ensures code quality, security, and consistency across all WHMCS customizations before deployment.

## Prerequisites

- Access to code repository with proposed changes
- WHMCS development environment
- Code review tools (git, diff viewers)
- WHMCS coding standards documentation
- Security checklist reference

## Workflow Steps

### Step 1: Prepare Code for Review

The code author prepares the submission:

```bash
# Ensure all changes are on a dedicated branch
git checkout -b feature/your-feature-name
git add .
git commit -m "feat: Add custom billing calculation hook

- Implement custom pricing logic for tiered clients
- Add validation for negative amounts
- Include unit tests

Closes #123"

# Push branch for review
git push origin feature/your-feature-name
```

Create a pull request with:

```markdown
## Summary
Brief description of the changes and their purpose.

## Type of Change
- [ ] New feature
- [ ] Bug fix
- [ ] Performance improvement
- [ ] Security fix
- [ ] Refactoring

## Testing Done
- [ ] Unit tests added/updated
- [ ] Manual testing completed
- [ ] Tested on staging environment

## Screenshots (if applicable)
[Attach relevant screenshots]

## Breaking Changes
[Describe any breaking changes]
```

### Step 2: Automated Checks

Run automated analysis before human review:

```bash
# PHP CodeSniffer - Check coding standards
./vendor/bin/phpcs --standard=PSR12 modules/custom/mycustommodule/

# PHPStan - Static analysis
./vendor/bin/phpstan analyse modules/custom/mycustommodule/ --level=5

# PHP Mess Detector - Code quality
./vendor/bin/phpmd modules/custom/mycustommodule/ text cleancode,codesize,controversial,design,naming,unusedcode

# Security check with Composer audit
composer audit

# Custom WHMCS-specific checks
php -r "
require_once __DIR__ . '/init.php';
echo 'WHMCS Version: ' . \App::getVersion() . PHP_EOL;
echo 'PHP Version: ' . PHP_VERSION . PHP_EOL;
"
```

Run WHMCS-specific validation:

```php
// Validate hook syntax
// File: /resources/testing/ValidateHooks.php

function validateHookFile($filePath) {
    $content = file_get_contents($filePath);
    
    // Check for required elements
    if (!preg_match('/add_hook\s*\(/', $content)) {
        return ['valid' => false, 'error' => 'No add_hook() call found'];
    }
    
    // Check hook priority
    if (!preg_match('/add_hook\([^,]+,\s*\d+\)/', $content)) {
        return ['valid' => false, 'error' => 'Hook priority not specified'];
    }
    
    return ['valid' => true];
}
```

### Step 3: Security Review

Examine code for security vulnerabilities:

```php
// Review checklist for each file:

// 1. Input validation
// BAD:
$userId = $_GET['id'];
$query = "SELECT * FROM tblclients WHERE id = $userId";

// GOOD:
$userId = (int) $_GET['id'];
$query = "SELECT * FROM tblclients WHERE id = ?";
$stmt = mysqli_prepare($connection, $query);
mysqli_stmt_bind_param($stmt, "i", $userId);

// 2. Output escaping
// BAD:
echo $userInput;

// GOOD:
echo htmlspecialchars($userInput, ENT_QUOTES, 'UTF-8');

// 3. Permission checks
// Check for proper ACL validation
if (!function_exists('checkPermission')) {
    // Missing permission check
}

// 4. Sensitive data handling
// Verify no hardcoded credentials
// Check that passwords/secrets come from config, not code
```

Run security scan:

```bash
# Check for common vulnerabilities
grep -rn "mysqli_query.*\$_" modules/ 2>/dev/null | grep -v ".bak"
grep -rn "\$_GET\[" modules/ 2>/dev/null | grep -v "intval\|filter_input"

# Verify no debug output remains
grep -rn "var_dump\|print_r\|console.log" modules/ 2>/dev/null

# Check file permissions
find modules/ -type f -name "*.php" -exec stat -c '%a %n' {} \;
```

### Step 4: Architecture Review

Evaluate code structure and design:

```php
// Review points for WHMCS code:

// 1. Hook usage
// Good: Use appropriate hook points
// Bad: Overusing hooks or using wrong hook types

// Example of proper hook usage:
add_hook('InvoiceCreationComplete', 1, function($vars) {
    // Hook logic here
});

// 2. Module structure (for custom modules)
class My_Custom_Module extends \WHMCS\Module\Contracts\ModuleInterface
{
    // Required methods implementation
    // Proper error handling
    // Configuration options
    
    public function doSomething()
    {
        // Use WHMCS API where possible
        // Follow WHMCS conventions
    }
}

// 3. Database operations
// Use WHMCS database class
$results = \Illuminate\Database\Capsule\Manager::table('tblcustom')
    ->where('userid', $userId)
    ->get();

// 4. Template integration
// Use Smarty properly
{$LANG.your_custom_lang_key}
{include file="$template/custom/file.tpl"}
```

### Step 5: Performance Review

Assess performance implications:

```php
// Performance anti-patterns to check:

// N+1 queries
// BAD:
foreach ($clients as $client) {
    $orders = Capsule::table('tblorders')
        ->where('userid', $client->id)->get();
}

// GOOD (eager loading):
$clients = Capsule::table('tblclients')
    ->with(['orders'])
    ->get();

// Missing indexes check
// Verify frequently queried columns have indexes
// Check with: EXPLAIN SELECT * FROM tblcustom WHERE...

// Caching
// Implement caching for expensive operations
if (!$cached = Cache::get('my_custom_cache_' . $key)) {
    $cached = expensive_operation();
    Cache::set('my_custom_cache_' . $key, $cached, 1440); // 24 hours
}
```

### Step 6: Human Review Process

For human reviewers:

```markdown
## Code Review Comments

### Issues Found:

1. **[CRITICAL]** SQL Injection vulnerability
   - File: `modules/custom/billing.php`
   - Line: 145
   - Comment: User input directly in query
   
2. **[HIGH]** Missing permission check
   - File: `hooks/client.php`
   - Line: 23
   - Comment: Function called without admin permission validation

3. **[MEDIUM]** Performance issue
   - File: `modules/custom/reporting.php`
   - Line: 89
   - Comment: Query inside loop causes N+1 issue

### Suggestions:

1. Consider using WHMCS built-in validation classes
2. Add unit tests for edge cases
3. Document complex business logic

### Approval Status:
- [ ] Approved
- [ ] Changes Requested
- [ ] Blocked
```

### Step 7: Address Feedback

Author addresses review comments:

```bash
# Make corrections based on feedback
git checkout feature/your-feature-name
git pull origin main

# Make fixes
# ... edit files ...

git add .
git commit -m "fix: Address code review feedback

- Fixed SQL injection vulnerability in billing.php
- Added permission check in client.php hook
- Optimized reporting query with eager loading"

git push origin feature/your-feature-name
```

### Step 8: Final Approval and Merge

Once all issues resolved:

```bash
# Merge to main branch
git checkout main
git pull origin main
git merge feature/your-feature-name
git push origin main

# Tag the release
git tag -a v1.2.3 -m "Release version 1.2.3 with custom billing fix"
git push origin v1.2.3

# Create deployment package
./scripts/create-deployment-package.sh v1.2.3
```

## Verification Checklist

- [ ] All automated checks passing (PHPCS, PHPStan, security)
- [ ] No critical or high severity issues outstanding
- [ ] Code follows WHMCS coding standards
- [ ] Security review completed and signed off
- [ ] Performance impact assessed and acceptable
- [ ] Unit tests written and passing
- [ ] Manual testing documented
- [ ] All review comments addressed
- [ ] Code approved by at least one reviewer
- [ ] Changes documented in changelog
- [ ] Deployment plan prepared

## Related Skills and Documentation

- [WHMCS Module Development](whmcs-module-development-workflow.md)
- [WHMCS Security Audit](whmcs-security-audit.md)
- [WHMCS Module Testing](whmcs-module-testing.md)
- [WHMCS Module Upgrade](whmcs-module-upgrade.md)
- WHMCS Coding Standards: https://developers.whmcs.com/advanced/coding-standards/
- PSR-12 Coding Style Guide

## Notes

- Set a maximum review time limit (e.g., 24-48 hours) to avoid bottlenecks
- For urgent security fixes, use expedited review process with additional scrutiny
- Keep reviews focused on the specific changes, not unrelated code
- Use consistent review templates for efficiency
- Consider pair programming for complex features
