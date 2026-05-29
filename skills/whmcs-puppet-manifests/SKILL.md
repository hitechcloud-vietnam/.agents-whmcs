---
name: whmcs-puppet-manifests
description: Puppet setup for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Puppet Manifests Skill

## Overview
This skill provides patterns for Puppet manifest management in WHMCS.

## Implementation Patterns

### Puppet Manifest Manager
```php
<?php
/**
 * WHMCS Puppet Manifests
 * Manages Puppet manifests
 */

namespace WHMCS\Module\DevOps\Puppet;

class PuppetManifestManager {
    /**
     * Generate manifest
     */
    public function generateManifest(array $params): string {
        return <<<PP
class whmcs::server {
    package { 'nginx':
        ensure => installed,
    }

    service { 'nginx':
        ensure => running,
        require => Package['nginx'],
    }
}
PP;
    }
}
```

## Best Practices

1. **Module Organization**: Use modules properly
2. **Hiera Integration**: Use Hiera for data
3. **Testing**: Use rspec-puppet
4. **Catalog Testing**: Validate catalogs
5. **Environment Isolation**: Use environments

## Related Skills

- whmcs-ansible-playbooks
- whmcs-chef-cookbooks
- whmcs-config-management
- whmcs-infrastructure-code