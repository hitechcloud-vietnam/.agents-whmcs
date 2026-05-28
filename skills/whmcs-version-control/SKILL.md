# WHMCS Module Version Control Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for managing module versions with Git.

## When to Use

- Version management
- Release preparation
- Changelog maintenance

## Git Workflow

### Branch Structure
```
main                 ← Production
├── develop          ← Development
├── feature/*        ← Feature branches
├── hotfix/*         ← Hotfix branches
└── release/*        ← Release branches
```

### Version Tagging
```bash
# Create version tag
git tag -a v1.2.0 -m "Release version 1.2.0"
git tag -a v1.2.0-beta -m "Beta release"

# Push tags
git push origin v1.2.0

# List tags
git tag -l
git tag --sort=-version:refname
```

### CHANGELOG Maintenance
```bash
# Generate changelog
git-changelog -o CHANGELOG.md

# Or manual format:
## [1.2.0] - 2024-05-28
### Added
- New feature A
- New feature B

### Changed
- Improved performance

### Fixed
- Bug fix for issue #123
```

### Release Script
```bash
#!/bin/bash
VERSION=$1

# Verify version format
if ! [[ $VERSION =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "Invalid version format"
    exit 1
fi

# Create release branch
git checkout -b release/$VERSION

# Update version in module
sed -i "s/version.*=.*'[0-9.]*'/version' => '$VERSION'/" module.php

# Create tag
git tag -a v$VERSION -m "Release version $VERSION"

# Push
git push origin release/$VERSION
git push origin v$VERSION
```

### Semver Guidelines
- **MAJOR**: Breaking changes
- **MINOR**: New features (backward compatible)
- **PATCH**: Bug fixes

---

**Related Skills:**
- whmcs-module-packaging
- whmcs-deployment
