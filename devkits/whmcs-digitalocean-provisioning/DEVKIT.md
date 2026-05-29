# WHMCS DigitalOcean Droplets Devkit

## Overview
A provisioning module for WHMCS that automates DigitalOcean droplet creation, management, and monitoring with integrated billing support.

## Features
- Droplet creation
- Droplet management
- Snapshot management
- SSH key management
- VPC networking
- Floating IPs
- Load balancers
- Block storage
- Monitoring integration
- DNS integration
- Backups automation
- Multi-region support

## WHMCS Integration Points
- Module: servers/digitalocean
- Hook: AfterModuleCreate
- Hook: AfterModuleSuspend
- Hook: AfterModuleTerminate

## Database Schema
```sql
CREATE TABLE mod_do_droplets (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    droplet_id INT NOT NULL,
    region VARCHAR(50),
    size_slug VARCHAR(50),
    image_slug VARCHAR(100),
    vpc_uuid VARCHAR(100),
    ipv4_address VARCHAR(45),
    ipv6_address VARCHAR(45),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service (service_id),
    INDEX idx_droplet (droplet_id)
);
```

## Configuration Fields
```php
function digitalocean_ConfigOptions(array $params): array {
    return [
        'Size' => ['Type' => 'dropdown', 'Options' => 's-1vcpu-1gb,s-2vcpu-2gb,s-4vcpu-8gb'],
        'Region' => ['Type' => 'dropdown', 'Options' => 'nyc1,nyc3,sfo3,ams3,fra1,sgp1'],
        'Image' => ['Type' => 'dropdown', 'Options' => 'ubuntu-22-04-lts,ubuntu-24-04-lts,debian-12'],
    ];
}
```

## Testing Checklist
- [ ] Droplet creation
- [ ] Suspend/resume
- [ ] Termination
- [ ] Snapshot creation
- [ ] Networking

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- DigitalOcean API v2
