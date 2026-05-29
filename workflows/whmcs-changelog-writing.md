# WHMCS Changelog Writing Workflow

## Overview
This workflow guides you through creating and maintaining changelogs for WHMCS modules.

## Prerequisites
- Git history
- Conventional commits
- Changelog generator (optional)

## Step-by-Step Guide

### Step 1: Conventional Commits
```bash
# Commit types
feat:     New feature
fix:      Bug fix
docs:     Documentation
style:    Formatting
refactor: Code refactoring
perf:     Performance
test:     Tests
chore:    Maintenance

# Examples
git commit -m "feat: add client sync feature"
git commit -m "fix: resolve webhook timeout issue"
git commit -m "docs: update API documentation"
```

### Step 2: Create Changelog File
```markdown
# Changelog

All notable changes to this module will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/).

## [2.0.0] - 2024-01-15

### Added
- New dashboard widget for client statistics
- Support for bulk operations
- Webhook event filtering

### Changed
- Improved sync performance by 50%
- Updated API to v2 endpoints
- Refactored configuration system

### Deprecated
- Legacy API endpoints (will be removed in v3.0)
- Old widget format

### Removed
- Support for WHMCS 7.x
- Deprecated configuration options

### Fixed
- Webhook timeout handling
- Client sync error on empty results
- Dashboard widget rendering

### Security
- Updated dependency versions
- Enhanced webhook signature verification
```

### Step 3: Auto-Generate Changelog
```bash
# Using conventional-changelog
npm install -g conventional-changelog-cli

# Generate changelog
conventional-changelog -p angular -i CHANGELOG.md -s

# Or with preset
conventional-changelog -p keepachangelog -i CHANGELOG.md -s
```

### Step 4: Include in Release
```bash
# Generate release notes for GitHub
git tag v2.0.0
git push origin v2.0.0

# Create GitHub release with changelog
gh release create v2.0.0 \
    --title "Version 2.0.0" \
    --notes "$(cat CHANGELOG.md | head -50)"
```

## Changelog Checklist

### Content
- [ ] Version and date included
- [ ] Changes categorized
- [ ] Breaking changes noted
- [ ] Migration steps included

### Format
- [ ] Consistent formatting
- [ ] Links to issues
- [ ] Consistent terminology
