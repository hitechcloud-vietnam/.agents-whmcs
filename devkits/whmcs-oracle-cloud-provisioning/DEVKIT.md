# WHMCS Oracle Cloud Infrastructure Devkit

## Overview
A provisioning module for WHMCS that automates Oracle Cloud Infrastructure (OCI) compute instance creation with full lifecycle management.

## Features
- OCI compute instance creation
- Instance management
- VCN networking
- Subnet configuration
- Security lists
- SSH key management
- Block volume management
- Load balancer integration
- Reserved IPs
- Monitoring integration
- Compute shapes support
- Multi-compartment support

## WHMCS Integration Points
- Module: servers/oracle_cloud
- Hook: AfterModuleCreate
- Hook: AfterModuleSuspend
- Hook: AfterModuleTerminate

## Database Schema
```sql
CREATE TABLE mod_oci_instances (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    instance_id VARCHAR(100) NOT NULL,
    compartment_id VARCHAR(100),
    availability_domain VARCHAR(50),
    shape VARCHAR(50),
    subnet_id VARCHAR(100),
    vcn_id VARCHAR(100),
    primary_ip VARCHAR(45),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service (service_id),
    INDEX idx_instance (instance_id)
);
```

## Configuration Fields
```php
function oracle_cloud_ConfigOptions(array $params): array {
    return [
        'Shape' => ['Type' => 'dropdown', 'Options' => 'VM.Standard2.1,VM.Standard2.2,VM.Standard.E4.Flex'],
        'Availability Domain' => ['Type' => 'dropdown'],
        'Image OCID' => ['Type' => 'text'],
    ];
}
```

## Testing Checklist
- [ ] Instance creation
- [ ] Suspend/resume
- [ ] Termination
- [ ] Networking
- [ ] Storage

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- Oracle Cloud SDK
