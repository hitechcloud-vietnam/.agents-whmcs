# WHMCS Webhook Automation Devkit

## Overview
A webhook automation system for WHMCS that enables outbound webhooks for events, inbound webhook processing, webhook logging, and retry mechanisms.

## Features
- Outbound webhooks
- Event triggers
- Custom payload templates
- HTTP authentication
- Retry mechanisms
- Webhook logging
- Inbound webhook processing
- Webhook signature verification
- Payload transformation
- Header customization
- SSL certificate handling
- Webhook testing tool

## WHMCS Integration Points
- Module: addon/WebhookAutomation
- Hook: Multiple event hooks
- API integration

## Database Schema
```sql
CREATE TABLE mod_webhook_endpoints (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    url VARCHAR(1000) NOT NULL,
    method ENUM('GET', 'POST', 'PUT', 'PATCH') DEFAULT 'POST',
    auth_type ENUM('none', 'basic', 'bearer', 'api_key') DEFAULT 'none',
    auth_config JSON,
    headers JSON,
    retry_count INT DEFAULT 3,
    retry_delay INT DEFAULT 60,
    is_active TINYINT(1) DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_webhook_triggers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    endpoint_id INT NOT NULL,
    whmcs_event VARCHAR(100) NOT NULL,
    payload_template TEXT,
    condition_filter JSON,
    is_active TINYINT(1) DEFAULT 1,
    INDEX idx_endpoint (endpoint_id)
);

CREATE TABLE mod_webhook_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    trigger_id INT NOT NULL,
    event_type VARCHAR(100),
    request_url VARCHAR(1000),
    request_headers JSON,
    request_body TEXT,
    response_code INT,
    response_body TEXT,
    retry_count INT DEFAULT 0,
    status ENUM('pending', 'success', 'failed', 'retrying') DEFAULT 'pending',
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_trigger (trigger_id),
    INDEX idx_status (status)
);

CREATE TABLE mod_inbound_webhooks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    webhook_key VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(255),
    handler_class VARCHAR(255),
    is_active TINYINT(1) DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## API Endpoints
- POST /api/webhooks/endpoints - Create endpoint
- GET /api/webhooks/endpoints/:id/logs - Get logs
- POST /api/webhooks/test - Test webhook
- POST /api/webhooks/inbound/:key - Inbound webhook

## Testing Checklist
- [ ] Outbound webhook sending
- [ ] Retry mechanism
- [ ] Inbound processing
- [ ] Signature verification
- [ ] Logging accuracy

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
