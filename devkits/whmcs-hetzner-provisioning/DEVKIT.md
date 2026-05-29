# WHMCS Hetzner Cloud Provisioning Devkit

## Overview
A provisioning module for WHMCS that automates Hetzner Cloud server creation, management, and monitoring with integrated billing support.

## Features
- Hetzner Cloud server creation
- Server management
- Image selection
- SSH key management
- Firewall management
- Network configuration
- Load balancer support
- Block storage
- Snapshot management
- Rescue mode
- Reverse DNS management
- Multi-location support

## WHMCS Integration Points
- Module: servers/hetzner
- Hook: AfterModuleCreate
- Hook: AfterModuleSuspend
- Hook: AfterModuleTerminate

## Database Schema
```sql
CREATE TABLE mod_hetzner_servers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    server_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    location VARCHAR(50),
    server_type VARCHAR(50),
    image VARCHAR(100),
    datacenter VARCHAR(50),
    ipv4_address VARCHAR(45),
    ipv6_address VARCHAR(50),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service (service_id),
    INDEX idx_server (server_id)
);
```

## Configuration Fields
```php
function hetzner_ConfigOptions(array $params): array {
    return [
        'Server Type' => ['Type' => 'dropdown', 'Options' => 'cx22,cx32,cx42,cx52'],
        'Location' => ['Type' => 'dropdown', 'Options' => 'fsn1,nbg1,hel1,ash'],
        'Image' => ['Type' => 'dropdown', 'Options' => 'ubuntu-22.04,debian-12,centos-7'],
    ];
}
```

## Testing Checklist
- [ ] Server creation
- [ ] Suspend/resume
- [ ] Termination
- [ ] Firewall rules
- [ ] Snapshots

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- Hetzner Cloud API
