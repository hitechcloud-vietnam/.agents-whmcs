# WHMCS Vultr VPS Provisioning Devkit

## Overview
A provisioning module for WHMCS that automates Vultr VPS instance creation, management, and monitoring with full billing integration.

## Features
- VPS instance creation
- Instance management
- ISO management
- Snapshot management
- SSH key management
- Firewall groups
- Load balancers
- Block storage
- DNS management
- Reserved IPs
- Server backups
- Multi-location support

## WHMCS Integration Points
- Module: servers/vultr
- Hook: AfterModuleCreate
- Hook: AfterModuleSuspend
- Hook: AfterModuleTerminate

## Database Schema
```sql
CREATE TABLE mod_vultr_instances (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    subid INT NOT NULL,
    location VARCHAR(50),
    plan VARCHAR(50),
    os VARCHAR(100),
    firewall_group_id VARCHAR(50),
    main_ip VARCHAR(45),
    internal_ip VARCHAR(45),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service (service_id),
    INDEX idx_subid (subid)
);
```

## Configuration Fields
```php
function vultr_ConfigOptions(array $params): array {
    return [
        'Plan' => ['Type' => 'dropdown', 'Options' => 'vc-2c-4gb,vc-4c-8gb,vc-8c-16gb,vc-16c-32gb'],
        'Location' => ['Type' => 'dropdown', 'Options' => 'New Jersey,Los Angeles,London,Tokyo'],
        'OS' => ['Type' => 'dropdown', 'Options' => 'Ubuntu 22.04 LTS,Ubuntu 24.04 LTS,Debian 12,CentOS 7'],
    ];
}
```

## Testing Checklist
- [ ] Instance creation
- [ ] Suspend/resume
- [ ] Termination
- [ ] Firewall rules
- [ ] Snapshots

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- Vultr API v2
