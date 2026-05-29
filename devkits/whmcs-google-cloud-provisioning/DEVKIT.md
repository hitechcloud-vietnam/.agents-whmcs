# WHMCS Google Cloud Platform Compute Devkit

## Overview
A provisioning module for WHMCS that automates Google Cloud Platform compute engine instance creation with full lifecycle management and monitoring.

## Features
- GCP compute instance creation
- Instance management
- Managed instance groups
- Cloud load balancing
- Cloud Storage integration
- VPC network configuration
- Firewall rules
- Static IP management
- Cloud Monitoring integration
- Cloud Logging integration
- Snapshot management
- Multi-zone support

## WHMCS Integration Points
- Module: servers/gcp_compute
- Hook: AfterModuleCreate
- Hook: AfterModuleSuspend
- Hook: AfterModuleTerminate

## Database Schema
```sql
CREATE TABLE mod_gcp_instances (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    project_id VARCHAR(255) NOT NULL,
    instance_name VARCHAR(255) NOT NULL,
    zone VARCHAR(50) NOT NULL,
    machine_type VARCHAR(50),
    image_project VARCHAR(100),
    image_family VARCHAR(100),
    network_name VARCHAR(255),
    subnetwork_name VARCHAR(255),
    internal_ip VARCHAR(45),
    external_ip VARCHAR(45),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service (service_id),
    INDEX idx_instance (instance_name)
);
```

## Configuration Fields
```php
function gcp_compute_ConfigOptions(array $params): array {
    return [
        'Machine Type' => ['Type' => 'dropdown', 'Options' => 'e2-medium,e2-standard-2,e2-standard-4'],
        'Zone' => ['Type' => 'dropdown', 'Options' => 'us-central1-a,us-east1-b,europe-west1-b'],
        'Image' => ['Type' => 'dropdown', 'Options' => 'ubuntu-2204-lts,debian-12-bookworm,centos-7'],
    ];
}
```

## Testing Checklist
- [ ] Instance creation
- [ ] Suspend/resume
- [ ] Termination
- [ ] Networking
- [ ] Monitoring

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- Google Cloud SDK
