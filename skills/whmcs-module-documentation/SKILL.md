# WHMCS Module Documentation Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for writing comprehensive module documentation.

## When to Use

- Documenting new modules
- Preparing user guides
- API documentation

## Documentation Structure

```markdown
# Module Name

Brief description of what the module does.

## Features

- Feature 1
- Feature 2
- Feature 3

## Requirements

- WHMCS 8.0+
- PHP 8.1+
- Required extensions

## Installation

1. Download module
2. Upload to modules/
3. Activate in WHMCS Admin
4. Configure settings

## Configuration

Describe each configuration option:
- `Option Name`: Description

## Usage

Step-by-step usage guide with examples.

## API Reference

```php
function exampleFunction(array $params): string {
    // Description
}
```

## Troubleshooting

Common issues and solutions.

## Support

Contact information for support.
```

## README Template

```markdown
# {Module Name}

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)]
[![WHMCS](https://img.shields.io/badge/WHMCS-8.0+-green.svg)]
[![PHP](https://img.shields.io/badge/PHP-8.1+-purple.svg)]

## Quick Start

```bash
# Install
cp -r module /var/www/html/modules/addons/

# Activate
# Navigate to WHMCS Admin > System > Module Addons
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| api_key | - | API key |
| enabled | yes | Enable module |
```

---

**Related Skills:**
- whmcs-deployment
- whmcs-module-packaging
