# WHMCS Module Submission to Marketplace

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide covers the process of submitting modules to the WHMCS Marketplace, including preparation, review requirements, submission steps, and post-submission management.

---

## Submission Prerequisites

### Account Requirements

1. Create WHMCS Marketplace account
   - Register at: https://marketplace.whmcs.com
   - Verify email address
   - Complete profile information

2. Set up vendor profile
   - Add company/developer information
   - Upload logo (256x256px PNG)
   - Add support contact details
   - Set payout preferences

3. Complete verification
   - Identity verification (for paid modules)
   - Tax information (W-9/W-8BEN)
   - Payment method setup

### Module Requirements

#### Quality Standards

| Requirement | Standard |
|-------------|----------|
| Code Quality | No syntax errors, follows coding standards |
| Documentation | Clear README, installation instructions |
| Security | Passes security review |
| Testing | Tested on current WHMCS version |
| Compatibility | Supports WHMCS 8.x |
| Licensing | Valid license file |

#### Prohibited Content

- Code that collects user data without disclosure
- Hidden backdoors or malicious code
- Code that breaks WHMCS functionality
in
- Infringing intellectual property
- Adult content or gambling functionality
- Misleading descriptions

## Submission Process

### Step 1: Prepare Package

```bash
# Package structure
your_module/
├── modules/
│   └── addons/
│       └── your_addon/
├── README.md
├── CHANGELOG.md
├── LICENSE
├── module.xml
└── screenshots/
    ├── settings.png
    └── admin_panel.png
```

### Step 2: Create Listings

#### Addon Module Listing

| Field | Description | Example |
|-------|-------------|---------|
| Title | Module name | Advanced Analytics for WHMCS |
| Tagline | Brief description | Track client behavior and engagement |
| Description | Full feature list | Markdown formatted |
| Category | Primary category | Client Management |
| Tags | Search keywords | analytics, tracking, reports |
| Price | Module price | $49.99 |
| Support | Support policy | Email, 9-5 EST |

#### Gateway Module Listing

| Field | Description |
|-------|-------------|
| Title | Gateway name | Stripe Payments Gateway |
| Supported Currencies | List currencies | USD, EUR, GBP |
| Features | Payment types | Credit Card, ACH, crypto |
| Settlement | Settlement time | Instant, Daily, Weekly |

### Step 3: Upload Files

1. Navigate to Marketplace Dashboard
2. Click "Submit Module"
3. Select module type:
   - Addon Module
   - Payment Gateway
   - Registrar Module
   - Notification Provider
   - Widget

4. Fill metadata:
   - Module name
   - Version
   - WHMCS version requirements
   - PHP version requirements

5. Upload files:
   - ZIP package (max 50MB)
   - Screenshots (optional)
   - Demo video URL (optional)

### Step 4: Add Screenshots

#### Required Screenshots

| Module Type | Required Screenshots |
|-------------|----------------------|
| Addon | Admin settings, Module output |
| Gateway | Configuration, Checkout flow |
| Registrar | Domain management, Settings |
| Notification | Alert configuration, Notification output |

#### Screenshot Guidelines

- Minimum 800px width
- PNG or JPEG format
- Show realistic data (use test data)
- No sensitive information visible

### Step 5: Write Description

```markdown
# Module Title

## Overview
Brief introduction to what the module does.

## Features
- Feature 1 with detailed description
- Feature 2 with detailed description
- Feature 3 with detailed description

## Benefits
- Customer benefit 1
- Customer benefit 2
- Customer benefit 3

## Installation
Step-by-step installation instructions.

## Configuration
Detailed configuration options.

## FAQ
Common questions and answers.
```

## Review Process

### Review Stages

```
Submission → Initial Review → Security Review → Final Approval → Published
                   ↓              ↓
              Returned         Failed
               (fixes)         (rejected)
```

### Review Timeline

| Stage | Duration | Action |
|-------|----------|--------|
| Initial Review | 3-5 business days | Code quality, completeness |
| Security Review | 5-10 business days | Vulnerability scan |
| Final Approval | 1-2 business days | Final checks |

### Common Rejection Reasons

1. **Missing Documentation**
   - No README.md
   - Incomplete installation instructions
   - Missing configuration guide

2. **Security Issues**
   - SQL injection vulnerabilities
   - XSS vulnerabilities
   - Missing CSRF protection
   - Unsecured API calls

3. **Code Quality**
   - Syntax errors
   - Deprecated PHP functions
   - Missing error handling

4. **Compatibility**
   - Not tested on current WHMCS
   - Incompatible with required PHP version

### Resubmission Process

```bash
# Fix common issues
1. Update documentation
2. Fix security vulnerabilities
3. Add missing features
4. Test on latest WHMCS version
5. Create new package
6. Select "Revision" when rescheduling
```

## Marketplace Management

### Post-Submission Options

| Action | Description |
|--------|-------------|
| Edit Listing | Update description, screenshots |
| Update Version | Upload new module version |
| Set Pricing | Change price, add discounts |
| Enable/Disable | Temporarily hide from listing |

### Version Updates

```bash
# Version update process
1. Make changes to module
2. Update version number
3. Update CHANGELOG.md
4. Create new ZIP package
5. Go to Module Versions
6. Click "Add New Version"
7. Upload new ZIP
8. Submit for review
```

### Pricing Management

#### Pricing Options

| Model | Description |
|-------|-------------|
| Single Purchase | One-time payment |
| Subscription | Recurring charges |
| Free | No charge |

#### Discount Codes

```bash
# Create discount
1. Navigate to Module Listing
2. Click "Discount Codes"
3. Create new code:
   - Code: SAVE20
   - Discount: 20%
   - Expires: 2026-12-31
   - Max uses: 100
```

## Support Integration

### Marketplace Support

```bash
# Enable support features
1. Set support email
2. Configure support hours
3. Add FAQ section
4. Enable ticket system
5. Set response time expectation
```

### Customer Communication

| Message Type | Purpose |
|-------------|---------|
| Thank You | Post-purchase message |
| Installation Guide | Initial setup help |
| Update Notice | New version announcement |
| Support Response | Ticket replies |

---

## Related Skills and Workflows

- `module-packaging-guide` - Package preparation
- `module-release-checklist` - Pre-submission checklist
- `module-security-standards` - Security requirements
- `module-testing-strategies` - Testing requirements
