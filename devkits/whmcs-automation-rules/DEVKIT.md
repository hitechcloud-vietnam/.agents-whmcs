# WHMCS Rule-Based Automation Devkit

## Overview
A comprehensive rule-based automation system for WHMCS that executes actions based on configurable triggers and conditions with extensive workflow capabilities.

## Features
- Visual rule builder
- Multiple trigger types
- Condition evaluation
- Action execution
- Rule templates
- Rule scheduling
- Rule testing
- Execution logging
- Rule priority
- Enable/disable rules
- Rule import/export
- Workflow automation

## WHMCS Integration Points
- Module: addon/AutomationRules
- Hook: Multiple hooks supported
- Cron job integration

## Database Schema
```sql
CREATE TABLE mod_automation_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    trigger_type VARCHAR(100) NOT NULL,
    trigger_config JSON NOT NULL,
    conditions JSON,
    actions JSON,
    is_active TINYINT(1) DEFAULT 1,
    execution_count INT DEFAULT 0,
    last_executed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE mod_automation_execution_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    rule_id INT NOT NULL,
    trigger_event VARCHAR(255),
    conditions_met TINYINT(1),
    actions_executed JSON,
    execution_time_ms INT,
    status ENUM('success', 'failed', 'skipped') NOT NULL,
    error_message TEXT,
    executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_rule (rule_id)
);

CREATE TABLE mod_automation_templates (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    category VARCHAR(100),
    trigger_type VARCHAR(100) NOT NULL,
    template_config JSON NOT NULL,
    usage_count INT DEFAULT 0,
    is_featured TINYINT(1) DEFAULT 0
);
```

## API Endpoints
- GET /api/automation/rules - List rules
- POST /api/automation/rules - Create rule
- PUT /api/automation/rules/:id - Update rule
- DELETE /api/automation/rules/:id - Delete rule
- POST /api/automation/rules/:id/test - Test rule
- GET /api/automation/logs - Get execution logs

## Testing Checklist
- [ ] Rule creation
- [ ] Trigger execution
- [ ] Condition evaluation
- [ ] Action execution
- [ ] Logging accuracy

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
