# WHMCS Scaleway Provisioning Devkit

## Overview
A provisioning module for WHMCS that automates Scaleway cloud server creation, management, and monitoring with integrated billing support.

## Features
- Scaleway instance creation
- Instance management
- Flexible instances
- GPU instances
- Devis-based instances
- SSD and volume management
- VPC networking
- Elastic metal servers
- Kubernetes integration
- Load balancer support
- Snapshot management
- Multi-zone support

## WHMCS Integration Points
- Module: servers/scaleway
- Hook: AfterModuleCreate
- Hook: AfterModuleSuspend
- Hook: AfterModuleTerminate

## Database Schema
```sql
CREATE TABLE mod_scaleway_instances (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    server_id VARCHAR(100) NOT NULL,
    zone VARCHAR(50) NOT NULL,
    type VARCHAR(50),
    image VARCHAR(100),
    commercial_type VARCHAR(50),
    ip_address VARCHAR(45),
    private_ip VARCHAR(45),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service (service_id),
    INDEX idx_server (server_id)
);
```

## Configuration Fields
```php
function scaleway_ConfigOptions(array $params): array {
    return [
        'Instance Type' => ['Type' => 'dropdown', 'Options' => 'DEV1-S,DEV1-M,DEV1-L,STARDUST1-S'],
        'Zone' => ['Type' => 'dropdown', 'Options' => 'fr-par-1,fr-par-2,nl-ams-1,pl-waw-1'],
        'Image' => ['Type' => 'dropdown', 'Options' => 'ubuntu-jammy,debian-bookworm,centos-7'],
    ];
}
```

## Testing Checklist
- [ ] Instance creation
- [ ] Suspend/resume
- [ ] Termination
- [ ] Volume management
- [ ] Networking

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- Scaleway API
