# WHMCS Penetration Testing Workflow

## Overview
This workflow guides penetration testing activities for WHMCS installations.

## Step 1: Pre-Testing Preparation

```bash
# Reconnaissance tools setup
sudo apt install nmap sqlmap dirb nikto

# Information gathering
nmap -sV -sC target.example.com
nikto -h https://target.example.com/whmcs
dirb https://target.example.com/whmcs

# Create scope document
cat > scope.txt << 'EOF'
In Scope:
- https://whmcs.example.com
- Client area (authenticated and unauthenticated)
- Admin area
- API endpoints

Out of Scope:
- Email systems
- Third-party integrations
EOF
```

## Step 2: Testing Checklist

```markdown
## Automated Scanning

### Nikto Scan
```bash
nikto -h https://whmcs.example.com/whmcs -o nikto_results.txt
```

### SQLMap Testing
```bash
sqlmap -u "https://whmcs.example.com/whmcs/clientarea.php?id=1" --batch
sqlmap -u "https://whmcs.example.com/whmcs/api.php?action=GetClients" --cookie="SESSION=value"
```

### Directory Brute Force
```bash
dirb https://whmcs.example.com/whmcs /usr/share/dirb/wordlists/common.txt
```

## Manual Testing

### Authentication Testing
- [ ] Brute force protection
- [ ] Password policy enforcement
- [ ] Account lockout
- [ ] Session management
- [ ] Password reset vulnerabilities

### Authorization Testing
- [ ] IDOR in client area
- [ ] Privilege escalation
- [ ] Admin area access control

### Input Validation Testing
- [ ] XSS in all inputs
- [ ] SQL injection
- [ ] Command injection
- [ ] LDAP injection
- [ ] XML injection

### Business Logic Testing
- [ ] Price manipulation
- [ ] Invoice generation
- [ ] Payment processing
- [ ] Refund logic

### API Testing
- [ ] API authentication
- [ ] Rate limiting
- [ ] Input validation
- [ ] Response handling
```

## Step 3: Common Vulnerabilities Checklist

```markdown
## Common WHMCS Vulnerabilities to Test

### 1. Authentication Bypass
- Test default credentials
- Test weak session tokens
- Test SSO bypass

### 2. IDOR Vulnerabilities
- Test access to other clients' data
- Test access to admin functions
- Test parameter manipulation

### 3. XSS Vulnerabilities
- Client name fields
- Product descriptions
- Support ticket content
- Invoice notes

### 4. SQL Injection
- Search functionality
- API parameters
- URL parameters
- Cookie values

### 5. CSRF
- Form submissions
- Password changes
- Service cancellations

### 6. File Upload
- Avatar uploads
- Ticket attachments
- Module uploads
```

## Step 4: Report Template

```markdown
# Penetration Test Report

## Executive Summary
[Brief overview of findings]

## Scope
[Defined scope]

## Methodology
[Testing methodology used]

## Findings

### Critical Findings
| ID | Finding | Location | Impact | Remediation |
|----|---------|----------|--------|-------------|
| C-001 | [Title] | [URL] | [Impact] | [Fix] |

### High Findings
| ID | Finding | Location | Impact | Remediation |
|----|---------|----------|--------|-------------|
| H-001 | [Title] | [URL] | [Impact] | [Fix] |

## Proof of Concept
[Detailed POC for each finding]

## Recommendations
[Priority-ordered recommendations]

## Conclusion
[Summary]
```

## Verification Checklist

- [ ] Scope defined and approved
- [ ] Automated scans completed
- [ ] Manual testing completed
- [ ] Findings documented
- [ ] POC created for each finding
- [ ] Report generated
- [ ] Remediation plan created
