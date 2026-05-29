# WHMCS AWS EC2 Provisioning Module Devkit

## Overview
A complete provisioning module for WHMCS that automates AWS EC2 instance creation, management, and lifecycle operations with comprehensive billing integration.

## Features
- EC2 instance creation
- Instance lifecycle management
- AMI selection and deployment
- Security group management
- VPC and subnet integration
- Key pair management
- Auto-scaling support
- EBS volume management
- Load balancer integration
- CloudWatch monitoring
- Cost tracking and reporting
- Multi-region support

## WHMCS Integration Points
- Module: servers/aws_ec2
- Hook: AfterModuleCreate
- Hook: AfterModuleSuspend
- Hook: AfterModuleTerminate

## File Structure
```
whmcs-aws-ec2-provisioning/
├── DEVKIT.md
├── aws_ec2.php
├── includes/
│   ├── EC2Service.php
│   ├── AWSClient.php
│   └── InstanceManager.php
├── config/
│   └── regions.php
└── templates/
    ├── clientarea.tpl
    └── admin_config.tpl
```

## Database Schema
```sql
CREATE TABLE mod_aws_ec2_instances (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    instance_id VARCHAR(100) NOT NULL,
    region VARCHAR(50) NOT NULL,
    instance_type VARCHAR(50),
    ami_id VARCHAR(100),
    vpc_id VARCHAR(100),
    subnet_id VARCHAR(100),
    security_groups JSON,
    key_name VARCHAR(255),
    private_ip VARCHAR(45),
    public_ip VARCHAR(45),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service (service_id),
    INDEX idx_instance (instance_id)
);
```

## Configuration Fields
```php
function aws_ec2_ConfigOptions(array $params): array {
    return [
        'Instance Type' => [
            'Type' => 'dropdown',
            'Options' => 't2.micro,t2.small,t2.medium,t3.micro,t3.small,m5.large,m5.xlarge'
        ],
        'Region' => ['Type' => 'dropdown', 'Options' => 'us-east-1,us-west-2,eu-west-1,ap-southeast-1'],
        'AMI ID' => ['Type' => 'text'],
    ];
}
```

## Implementation Example
```php
function aws_ec2_CreateAccount(array $params): string {
    $ec2 = new EC2Service($params['servermoduleparams']);
    $instance = $ec2->runInstance([
        'InstanceType' => $params['configoption1'],
        'Region' => $params['configoption2'],
        'ImageId' => $params['configoption3'],
    ]);
    logActivity("EC2 Instance created: " . $instance['InstanceId']);
    return 'success';
}
```

## Testing Checklist
- [ ] Instance creation
- [ ] Suspend/resume
- [ ] Termination
- [ ] Password change
- [ ] Package upgrade

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- AWS SDK for PHP
