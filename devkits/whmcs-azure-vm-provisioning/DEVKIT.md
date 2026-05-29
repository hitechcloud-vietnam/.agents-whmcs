# WHMCS Azure VM Provisioning Module Devkit

## Overview
A comprehensive provisioning module for WHMCS that automates Microsoft Azure Virtual Machine creation, management, and monitoring with integrated billing.

## Features
- Azure VM creation
- Resource group management
- Virtual network configuration
- Storage account integration
- Managed disks
- Availability sets
- Load balancer integration
- Azure Monitor integration
- Azure AD integration
- Backup automation
- Auto-scaling
- Multi-region support

## WHMCS Integration Points
- Module: servers/azure_vm
- Hook: AfterModuleCreate
- Hook: AfterModuleSuspend
- Hook: AfterModuleTerminate

## Database Schema
```sql
CREATE TABLE mod_azure_vms (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    resource_group VARCHAR(255) NOT NULL,
    vm_name VARCHAR(255) NOT NULL,
    location VARCHAR(50) NOT NULL,
    vm_size VARCHAR(50),
    image_publisher VARCHAR(100),
    image_offer VARCHAR(100),
    image_sku VARCHAR(100),
    admin_username VARCHAR(255),
    private_ip VARCHAR(45),
    public_ip VARCHAR(45),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service (service_id)
);
```

## Configuration Fields
```php
function azure_vm_ConfigOptions(array $params): array {
    return [
        'VM Size' => ['Type' => 'dropdown', 'Options' => 'Standard_B1s,Standard_B2s,Standard_D2s_v3'],
        'Location' => ['Type' => 'dropdown', 'Options' => 'eastus,westus2,uksouth,southeastasia'],
        'Image' => ['Type' => 'dropdown', 'Options' => 'UbuntuLTS,CentOS-LVM,WindowsServer2019'],
    ];
}
```

## Testing Checklist
- [ ] VM creation
- [ ] Suspend/resume
- [ ] Termination
- [ ] Networking
- [ ] Storage

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- Azure SDK for PHP
