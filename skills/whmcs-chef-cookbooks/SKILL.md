---
name: whmcs-chef-cookbooks
description: Chef configuration for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Chef Cookbooks Skill

## Overview
This skill provides patterns for Chef cookbook management in WHMCS.

## Implementation Patterns

### Chef Cookbook Manager
```php
<?php
/**
 * WHMCS Chef Cookbooks
 * Manages Chef cookbooks
 */

namespace WHMCS\Module\DevOps\Chef;

class ChefCookbookManager {
    /**
     * Generate cookbook
     */
    public function generateCookbook(array $params): array {
        $cookbookPath = "/var/lib/chef/cookbooks/{$params['name']}";

        $metadata = <<<RB
name '#{$params['name']}'
description '#{$params['description']}'
version '#{$params['version']}'
maintainer '#{$params['maintainer']}'
maintainer_email '#{$params['email']}'
license '#{$params['license']}'
RB;

        return [
            'path' => $cookbookPath,
            'metadata' => $metadata
        ];
    }
}
```

## Best Practices

1. **Idempotent Recipes**: Write idempotent recipes
2. **Test Cookbooks**: Use Test Kitchen
3. **Cookbook Versioning**: Version cookbooks properly
4. **Dependencies**: Manage cookbook dependencies
5. **Documentation**: Document recipes

## Related Skills

- whmcs-ansible-playbooks
- whmcs-puppet-manifests
- whmcs-config-management
- whmcs-infrastructure-code