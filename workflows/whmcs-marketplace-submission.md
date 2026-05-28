# WHMCS Module Marketplace Submission Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for packaging and submitting modules to the WHMCS Marketplace.

## When to Use

- Preparing modules for marketplace distribution
- Creating release packages
- Managing module versioning

## Marketplace Submission Patterns

### Module Package Structure

```
{moudulename}/
├── module.json           # Marketplace metadata
├── CHANGELOG.md         # Version history
├── README.md           # Installation guide
├── LICENSE.md          # License file
├── {moudulename}/
│   └── {moudulename}.php  # Main module file
├── templates/
├── lang/
├── hooks.php (if addon)
└── logo.png (80x80px)
```

### module.json Metadata

```json
{
    "name": "{ModuleName}",
    "slug": "modulename",
    "version": "1.0.0",
    "description": "Brief description of the module",
    "type": "server|gateway|registrar|addon|notification",
    "author": {
        "name": "Your Name",
        "email": "email@example.com",
        "website": "https://example.com"
    },
    "whmcs_version": "8.0|8.5|8.6|8.7|8.8",
    "php_version": "8.1|8.2",
    "requirements": {
        "extensions": ["curl", "json"]
    },
    "screenshots": [
        "screenshots/admin-dashboard.png",
        "screenshots/client-area.png"
    ],
    "tags": ["vps", "cloud", "automation"],
    "features": [
        "Feature 1",
        "Feature 2",
        "Feature 3"
    ],
    "pricing": {
        "type": "paid|free|fremium",
        "amount": 9.99,
        "currency": "USD",
        "billing_cycle": "monthly|annual|lifetime"
    }
}
```

### Version Tagging

```bash
# Create version tag
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0

# Create release package
mkdir -p releases/{moudulename}-v1.0.0
cp -r modules/{moudulename} releases/{moudulename}-v1.0.0/
cp module.json CHANGELOG.md README.md LICENSE.md releases/{moudulename}-v1.0.0/
cd releases && zip -r {moudulename}-v1.0.0.zip {moudulename}-v1.0.0
```

### CHANGELOG Format

```markdown
# Changelog

## [1.0.0] - 2024-05-28

### Added
- Initial release
- Feature A
- Feature B

### Changed
- Improved performance
- Updated UI

### Fixed
- Bug with configuration
- Error handling

## [0.9.0] - 2024-05-01

### Added
- Beta features
```

### README Template

```markdown
# {ModuleName}

Brief description of what the module does.

## Requirements

- WHMCS 8.0+
- PHP 8.1+
- cURL extension

## Installation

1. Download the latest release
2. Extract the archive
3. Upload the module folder to `/modules/`
4. Navigate to WHMCS Admin > System > Module Addons
5. Activate the module

## Configuration

1. Go to Module Settings
2. Enter your API credentials
3. Configure options as needed

## Features

- Feature description
- Feature description

## Support

For support, contact: email@example.com

## License

See LICENSE.md file for details.
```

### Submission Checklist

```
Pre-Submission:
□ Module follows WHMCS coding standards
□ All functions use proper naming convention {module}_*
□ No hardcoded credentials or sensitive data
□ Proper error handling implemented
□ CSRF protection on all forms
□ Input validation on all user inputs
□ SQL injection prevention
□ XSS protection (escape all output)
□ Secure password handling
□ Proper file permissions documentation

Documentation:
□ Complete README.md
□ Installation instructions
□ Configuration guide
□ Usage examples
□ Troubleshooting section
□ Changelog with version history

Testing:
□ Tested on latest WHMCS version
□ Tested on minimal PHP version
□ Tested on maximum PHP version
□ No console errors
□ All buttons work
□ Forms submit correctly
□ Module activates/deactivates properly
□ Data persists after deactivate/reactivate

Marketplace:
□ module.json metadata complete
□ 80x80px logo (PNG/JPG)
□ Screenshots (800x600px preferred)
□ Tag descriptions
□ Free trial option (if applicable)
□ License agreement
```

### Security Review Checklist

```php
Security Checklist for Submission:
□ No eval() or execute() with user input
□ No system() or exec() with user input
□ No SQL queries without prepared statements
□ No file operations without path validation
□ No include() with user-provided paths
□ API credentials stored encrypted
□ Session handling secure
□ Cookie settings secure
□ CSRF tokens on all forms
□ Output escaped properly
□ No debug output in production
□ Error messages don't expose paths
```

---

**Related Skills:**
- whmcs-security-hardening
- whmcs-deployment
- whmcs-testing-qa
