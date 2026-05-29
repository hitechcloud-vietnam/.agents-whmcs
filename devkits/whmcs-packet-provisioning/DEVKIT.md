# WHMCS Packet/Hosto Provisioning Devkit

## Overview
A provisioning module for WHMCS that automates Packet (now Hosto) bare metal server deployment, management, and monitoring with integrated billing.

## Features
- Bare metal server deployment
- Server management
- OS installation
- Hardware specifications
- IPMI management
- Bandwidth monitoring
- Raid configuration
- SSH key management
- Network configuration
- User data scripting
- Remote console access
- Multi-region support

## WHMCS Integration Points
- Module: servers/packet
- Hook: AfterModuleCreate
- Hook: AfterModuleSuspend
- Hook: AfterModuleTerminate

## Database Schema
```sql
CREATE TABLE mod_packet_servers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    server_id VARCHAR(100) NOT NULL,
    plan VARCHAR(50),
    facility VARCHAR(50),
    operating_system VARCHAR(100),
    ip_addresses JSON,
    root_password_encrypted TEXT,
    ipmi_ip VARCHAR(45),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service (service_id),
    INDEX idx_server (server_id)
);
```

## Configuration Fields
```php
function packet_ConfigOptions(array $params): array {
    return [
        'Plan' => ['Type' => 'dropdown', 'Options' => 't1.small,x1.small,x1.large'],
        'Facility' => ['Type' => 'dropdown', 'Options' => 'ewr1,sjc1,ams1,nrt1'],
        'OS' => ['Type' => 'dropdown', 'Options' => 'ubuntu_22_04,debian_12,centos_7'],
    ];
}
```

## Testing Checklist
- [ ] Server deployment
- [ ] Suspend/resume
- [ ] Termination
- [ ] OS reinstallation
- [ ] IPMI access

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- Packet/Hosto API
