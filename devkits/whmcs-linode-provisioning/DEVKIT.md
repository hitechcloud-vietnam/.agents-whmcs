# WHMCS Linode VPS Provisioning Devkit

## Overview
A provisioning module for WHMCS that automates Linode VPS instance creation, management, and lifecycle operations with integrated billing.

## Features
- Linode instance creation
- Instance management
- StackScript integration
- Image management
- VLAN networking
- NodeBalancer support
- Block storage
- Backups automation
- Longview monitoring
- DNS manager integration
- Firewall management
- Multi-datacenter support

## WHMCS Integration Points
- Module: servers/linode
- Hook: AfterModuleCreate
- Hook: AfterModuleSuspend
- Hook: AfterModuleTerminate

## Database Schema
```sql
CREATE TABLE mod_linode_instances (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    linode_id INT NOT NULL,
    region VARCHAR(50),
    type VARCHAR(50),
    image VARCHAR(100),
    vpc_id INT,
    ipv4 VARCHAR(45),
    ipv6 VARCHAR(50),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service (service_id),
    INDEX idx_linode (linode_id)
);
```

## Configuration Fields
```php
function linode_ConfigOptions(array $params): array {
    return [
        'Plan' => ['Type' => 'dropdown', 'Options' => 'Linode 1GB,Linode 2GB,Linode 4GB,Linode 8GB'],
        'Region' => ['Type' => 'dropdown', 'Options' => 'us-east,us-west,eu-west,ap-south'],
        'Image' => ['Type' => 'dropdown', 'Options' => 'linode/ubuntu22.04,linode/debian12,linode/centos7'],
    ];
}
```

## Testing Checklist
- [ ] Instance creation
- [ ] Suspend/resume
- [ ] Termination
- [ ] Networking
- [ ] Backups

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- Linode API v4
