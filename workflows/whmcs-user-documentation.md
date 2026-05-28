# WHMCS User Documentation Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Create comprehensive user documentation for WHMCS modules.

## Documentation Sections

### 1. Overview
```markdown
## Overview

{Module Name} provides [brief description].

### Features
- Feature A
- Feature B
- Feature C
```

### 2. Installation
```markdown
## Installation

1. Download the latest release
2. Upload to `modules/addons/` (or appropriate directory)
3. Navigate to Admin > System > Module Addons
4. Click "Activate" for {Module Name}
5. Configure settings
```

### 3. Configuration Reference
```markdown
## Configuration

| Setting | Description | Default |
|---------|-------------|---------|
| API Key | Your API key | - |
| Test Mode | Enable testing | No |
```

### 4. Usage Guide
```markdown
## Usage

### For Admins
1. Step-by-step admin tasks
2. With screenshots descriptions

### For Clients
1. Client-facing features
2. How to use the module
```

### 5. API Reference
```markdown
## API Reference

### Methods

#### `createAccount()`
Creates a new account.

Parameters:
- `domain` (string): Domain name
- `plan` (string): Selected plan

Returns: 'success' or error string
```

### 6. Troubleshooting
```markdown
## Troubleshooting

### Common Issues

**Q: Module not appearing?**
A: Check file permissions and naming.

**Q: API errors?**
A: Verify API credentials.
```

## Documentation Template
```php
/**
 * Documentation Generator
 * Generates Markdown docs from module metadata
 */

class DocGenerator {
    public function generate(): string {
        $meta = require __DIR__ . '/module.json';

        return <<<MD
# {$meta['name']}

{$meta['description']}

## Version
{$meta['version']}

## Requirements
- WHMCS {$meta['whmcs_version']}+
- PHP {$meta['php_version']}+

## Installation

[Installation instructions]

## Configuration

{$this->generateConfigTable()}

## API Reference

{$this->generateApiDocs()}

## Changelog

{$this->getChangelog()}
MD;
    }
}
```

---

**Related Skills:**
- whmcs-module-documentation
- whmcs-deployment
