# WHMCS Security Testing Workflow

## Overview
This workflow provides a comprehensive guide for security testing WHMCS modules and installations, identifying vulnerabilities and ensuring compliance with security best practices.

## Prerequisites
- WHMCS installation (v8.0+)
- Security testing tools (OWASP ZAP, Burp Suite, SQLMap)
- PHP security analysis tools
- Access to source code
- Test environment (never test on production)

## Step-by-Step Guide

### Step 1: Prepare Security Testing Environment

#### Isolated Test Environment
```bash
# Create isolated testing environment
git clone https://github.com/whmcs/whmcs /var/www/whmcs-test
cd /var/www/whmcs-test

# Configure test database
cp configuration.php.new configuration.php
# Edit configuration.php with test credentials

# Disable production features
echo "define('DEMO_MODE', true);" >> configuration.php
```

#### Security Testing Tools Setup
```bash
# Install OWASP ZAP
docker run -u zap -p 8080:8080 -p 9090:9090 \
  owasp/zap2docker-stable zap.sh -daemon \
  -port 8080 -host 0.0.0.0

# Install Burp Suite Community
# Download from https://portswigger.net/burp/releases

# Install SQLMap
pip install sqlmap
```

### Step 2: Static Code Analysis

#### PHP Security Checklist
```php
<?php
// Checklist for security review

// 1. Input Validation
// BAD: $input = $_GET['id'];
// GOOD:
$input = filter_input(INPUT_GET, 'id', FILTER_VALIDATE_INT);
if ($input === false) {
    http_response_code(400);
    exit('Invalid input');
}

// 2. SQL Injection Prevention
// BAD: "SELECT * FROM users WHERE id = $id"
// GOOD:
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$id]);

// 3. XSS Prevention
// BAD: echo $_GET['name'];
// GOOD:
echo htmlspecialchars($_GET['name'], ENT_QUOTES, 'UTF-8');

// 4. CSRF Protection
if (!hash_equals($_SESSION['csrf_token'], $_POST['csrf_token'])) {
    http_response_code(403);
    exit('CSRF validation failed');
}

// 5. Password Hashing
$hash = password_hash($password, PASSWORD_DEFAULT);

// 6. File Upload Security
$allowedTypes = ['image/jpeg', 'image/png'];
if (!in_array($_FILES['upload']['type'], $allowedTypes)) {
    exit('Invalid file type');
}

// 7. Session Security
session_regenerate_id(true);
ini_set('session.cookie_httponly', 1);
ini_set('session.cookie_secure', 1);
```

#### Security Scanner Script
```bash
#!/bin/bash
# security-scan.sh

echo "=== WHMCS Security Scan ==="

# PHP Syntax and Basic Security Check
echo "1. Checking for eval() usage..."
grep -rn "eval(" modules/ --include="*.php" | grep -v vendor

echo "2. Checking for SQL queries without prepared statements..."
grep -rn "mysql_query\|mysqli_query\|\$db->query" modules/ --include="*.php"

echo "3. Checking for file inclusion vulnerabilities..."
grep -rn "include.*\$_GET\|require.*\$_GET" modules/ --include="*.php"

echo "4. Checking for hardcoded credentials..."
grep -rn "password\s*=\s*['\"]" modules/ --include="*.php"
grep -rn "api_key\s*=\s*['\"]" modules/ --include="*.php"

echo "5. Checking for sensitive data in logs..."
grep -rn "password\|credit_card\|ssn" logs/ --include="*.log"

echo "6. Checking file permissions..."
find modules/ -type f -name "*.php" ! -perm 644

echo "7. Checking for dangerous functions..."
grep -rn "exec\|shell_exec\|system\|passthru" modules/ --include="*.php"
```

### Step 3: OWASP ZAP Scanning

#### Automated Scan Configuration
```bash
# Baseline scan
docker run -t owasp/zap2docker-stable zap-baseline.py \
  -t http://localhost/whmcs \
  -J zap-report.json

# Full scan with API
docker run -t owasp/zap2docker-stable zap-full-scan.py \
  -t http://localhost/whmcs \
  -r full-report.html

# API scan
docker run -t owasp/zap2docker-stable zap-api-scan.py \
  -f openapi \
  -t http://localhost/whmcs/api-docs.json \
  -r api-report.html
```

#### ZAP Script for WHMCS
```javascript
// zap-scripts/whmcs-active-scan.js
// Active scan script for WHMCS

function scan(as, target, state) {
  // Test for SQL Injection
  testSqlInjection(as, target, state);
  
  // Test for XSS
  testXSS(as, target, state);
  
  // Test for CSRF
  testCSRF(as, target, state);
  
  // Test for Authentication issues
  testAuthIssues(as, target, state);
}

function testSqlInjection(as, target, state) {
  var params = as.getParams(target, ['GET', 'POST']);
  
  for (var param in params) {
    // Test with SQL injection payloads
    var payloads = [
      "' OR '1'='1",
      "'; DROP TABLE users; --",
      "1 UNION SELECT * FROM users--",
      "1' AND '1'='1"
    ];
    
    for (var i = 0; i < payloads.length; i++) {
      as.addParamAlert({
        name: "SQL Injection",
        risk: 3,
        confidence: 2,
        param: param,
        attack: payloads[i]
      });
    }
  }
}

function testXSS(as, target, state) {
  var params = as.getParams(target, ['GET', 'POST']);
  
  for (var param in params) {
    var payloads = [
      "<script>alert('XSS')</script>",
      "<img src=x onerror=alert('XSS')>",
      "<svg/onload=alert('XSS')>",
      "javascript:alert('XSS')"
    ];
    
    for (var i = 0; i < payloads.length; i++) {
      as.addParamAlert({
        name: "Cross-Site Scripting (Reflected)",
        risk: 2,
        confidence: 3,
        param: param,
        attack: payloads[i]
      });
    }
  }
}
```

### Step 4: SQL Injection Testing

#### SQLMap Configuration
```bash
# Basic SQL injection scan
sqlmap -u "http://localhost/whmcs/clientarea.php?action=details&id=1" \
  --batch --level=5 --risk=3

# POST request testing
sqlmap -u "http://localhost/whmcs/submitticket.php" \
  --data="deptid=1&subject=test&message=test" \
  --batch

# With authentication
sqlmap -u "http://localhost/whmcs/dologin.php" \
  --data="username=admin&password=test" \
  --cookie="PHPSESSID=abc123" \
  --batch

# Test specific parameter
sqlmap -u "http://localhost/whmcs/clientarea.php" \
  --cookie="PHPSESSID=abc123" \
  -p "id" \
  --dbs

# Get database tables
sqlmap -u "http://localhost/whmcs/clientarea.php" \
  -p "id" \
  -D whmcs \
  --tables
```

### Step 5: Authentication Testing

#### Test Cases
```bash
# Test weak passwords
hydra -l admin -P wordlist.txt localhost http-post-form \
  "/whmcs/dologin.php:username=^USER^&password=^PASS^:Invalid"

# Test session management
# 1. Login and note session cookie
# 2. Logout
# 3. Try to use old session cookie

# Test brute force protection
for i in {1..20}; do
  curl -X POST http://localhost/whmcs/dologin.php \
    -d "username=admin&password=wrong$i"
done

# Test password reset vulnerabilities
curl -X POST http://localhost/whmcs/passwordreminder.php \
  -d "email=admin@example.com"
```

### Step 6: CSRF Testing

#### CSRF Token Verification
```php
// Verify CSRF protection in forms
<?php
// Check that all POST forms have CSRF tokens
$forms = [
    '/clientarea.php' => ['action' => 'update'],
    '/cart.php' => ['a' => 'checkout'],
    '/submitticket.php' => [],
];

foreach ($forms as $url => $params) {
    $html = file_get_contents($url);
    
    if (!preg_match('/<input[^>]*name=["\']csrf_token["\'][^>]*>/', $html)) {
        echo "Missing CSRF token in: $url\n";
    }
}
```

### Step 7: Run Security Tests

```bash
# Run security scan
./security-scan.sh

# Run OWASP ZAP scan
docker-compose up -d zap
curl http://localhost:8080

# Run SQLMap
sqlmap -m targets.txt --batch --level=5

# Generate security report
./generate-security-report.sh
```

## Security Checklist

### Authentication
- [ ] Strong password requirements enforced
- [ ] Brute force protection implemented
- [ ] Session timeout configured
- [ ] Multi-factor authentication available
- [ ] Password reset secure

### Authorization
- [ ] Role-based access control implemented
- [ ] Privilege escalation prevented
- [ ] API authentication secure

### Input Validation
- [ ] All user input validated
- [ ] SQL injection prevented
- [ ] XSS prevented
- [ ] CSRF tokens present
- [ ] File upload validation

### Data Protection
- [ ] Sensitive data encrypted
- [ ] HTTPS enforced
- [ ] Cookies secure/httponly
- [ ] No sensitive data in logs

### Configuration
- [ ] Debug mode disabled in production
- [ ] Error handling doesn't leak info
- [ ] File permissions correct
- [ ] Directory listing disabled

## Common Vulnerabilities to Check

| Vulnerability | Test Method | Remediation |
|---------------|-------------|-------------|
| SQL Injection | SQLMap, manual testing | Prepared statements |
| XSS | ZAP, manual testing | Output encoding |
| CSRF | Manual form review | CSRF tokens |
| Auth Bypass | Session testing | Proper session handling |
| IDOR | Parameter manipulation | Authorization checks |
| SSRF | URL injection testing | URL validation |
| File Upload | Malicious file upload | File type validation |

## Security Test Report Template
```markdown
# Security Test Report - [Date]

## Executive Summary
[High-level findings and recommendations]

## Scope
- WHMCS Version: [Version]
- Module Version: [Version]
- Test Date: [Date]
- Tester: [Name]

## Vulnerabilities Found

### Critical
| ID | Vulnerability | Location | Remediation |
|----|--------------|----------|-------------|
| C-001 | [Name] | [Path] | [Fix] |

### High
| ID | Vulnerability | Location | Remediation |
|----|--------------|----------|-------------|
| H-001 | [Name] | [Path] | [Fix] |

### Medium
| ID | Vulnerability | Location | Remediation |
|----|--------------|----------|-------------|
| M-001 | [Name] | [Path] | [Fix] |

### Low
| ID | Vulnerability | Location | Remediation |
|----|--------------|----------|-------------|
| L-001 | [Name] | [Path] | [Fix] |

## Tools Used
- [Tool 1]
- [Tool 2]

## Conclusion
[Overall security posture assessment]
```
