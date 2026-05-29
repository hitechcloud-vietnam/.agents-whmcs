# WHMCS Module Release Workflow

## Description
Release and distribute WHMCS modules professionally.

## Steps

### Step 1: Pre-Release Checklist
```bash
# Code quality
- [ ] All tests passing
- [ ] Code reviewed
- [ ] Documentation complete
- [ ] Changelog updated
- [ ] Version bumped

# Security
- [ ] No hardcoded credentials
- [ ] Input validation reviewed
- [ ] CSRF protection in place
- [ ] SQL injection prevention verified
```

### Step 2: Version Bump
```bash
# Update version in module
# 1.0.0 -> 1.0.1 (patch)
# 1.0.0 -> 1.1.0 (minor)
# 1.0.0 -> 2.0.0 (major)

# Update VERSION file
echo "1.0.0" > VERSION

# Update module.php
sed -i "s/'version' => '[0-9.]*'/'version' => '1.0.0'/" module.php
```

### Step 3: Update Changelog
```markdown
# Changelog

## [1.0.0] - 2024-01-15

### Added
- Initial release
- Feature A
- Feature B

### Changed
- Improvement in X

### Fixed
- Bug in Y
```

### Step 4: Create Git Tag
```bash
git add -A
git commit -m "Release v1.0.0"
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin main --tags
```

### Step 5: Build Release Package
```bash
# Create release package
mkdir -p release
cd release

# Copy module files
cp -r ../module/* .

# Create ZIP
zip -r module-v1.0.0.zip .

# Create checksums
sha256sum module-v1.0.0.zip > module-v1.0.0.zip.sha256
```

### Step 6: Create GitHub Release
```bash
# Using GitHub CLI
gh release create v1.0.0 \
    --title "Version 1.0.0" \
    --notes "Release notes here" \
    --draft

# Upload assets
gh release upload v1.0.0 module-v1.0.0.zip
gh release upload v1.0.0 module-v1.0.0.zip.sha256
```

### Step 7: Announce Release
- Update module documentation
- Announce on social media
- Email to subscribers
- Update WHMCS Marketplace listing

## Release Types
| Type | Version Bump | Description |
|------|-------------|-------------|
| Patch | 1.0.0 -> 1.0.1 | Bug fixes only |
| Minor | 1.0.0 -> 1.1.0 | New features, backward compatible |
| Major | 1.0.0 -> 2.0.0 | Breaking changes |

## Tags
- release
- versioning
- distribution