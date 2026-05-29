# WHMCS Release Notes Workflow

## Overview
This workflow guides you through creating release notes for WHMCS module releases.

## Prerequisites
- Changelog
- Feature documentation
- Audience awareness

## Step-by-Step Guide

### Step 1: Release Notes Template
```markdown
# Release Notes: Module Name v2.0.0

**Release Date:** January 15, 2024
**WHMCS Compatibility:** 8.0 - 8.x
**Download:** [Link]

## Highlights

[Brief overview of major changes]

## New Features

### Feature 1
Description of the feature and its benefits.

**Usage:**
```
Code example or usage instructions
```

### Feature 2
...

## Improvements

- Performance improvement: X% faster sync
- UI improvements in admin area
- Better error handling

## Bug Fixes

- Fixed: Issue with webhook handling
- Fixed: Error on client creation
- Fixed: Dashboard widget not loading

## Breaking Changes

**Important:** This release includes breaking changes.

1. Old API endpoints have been deprecated
2. Configuration format has changed
3. Minimum WHMCS version is now 8.0

### Migration Steps
1. Backup your configuration
2. Update module
3. Visit Settings > Migrate

## Upgrading

1. Deactivate the current module
2. Upload the new version
3. Activate the module
4. Run any pending migrations

## Known Issues

- None

## Support

For support, please contact:
- Email: support@example.com
- Docs: https://docs.example.com

---

Thank you for using Module Name!
```

### Step 2: Audience-Specific Notes
```markdown
# For Administrators
- New admin dashboard widget
- Improved reporting
- Better performance

# For Developers
- New API endpoints
- Webhook improvements
- Developer documentation updated
```

### Step 3: Distribute Release Notes
```bash
# Email announcement
./scripts/send-release-email.sh v2.0.0

# Update marketplace listing
./scripts/update-marketplace.sh v2.0.0

# Post on forum/social
./scripts/announce-release.sh v2.0.0
```

## Release Notes Checklist

### Content
- [ ] Version clearly stated
- [ ] Release date included
- [ ] New features documented
- [ ] Breaking changes noted
- [ ] Upgrade instructions clear

### Distribution
- [ ] Sent to users
- [ ] Marketplace updated
- [ ] Documentation updated
