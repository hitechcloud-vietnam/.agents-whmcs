# WHMCS System Health Monitoring Devkit

## Overview
A comprehensive health monitoring system for WHMCS that tracks server health, service status, and alerts administrators to issues with configurable thresholds.

## Features
- Server health metrics
- Service status monitoring
- Disk space tracking
- Memory usage monitoring
- Database health
- Cron job monitoring
- API response times
- Error rate tracking
- Custom health checks
- Alert thresholds
- Alert notifications
- Health reports

## WHMCS Integration Points
- Module: addon/HealthMonitor
- Hook: DailyCronJob
- Hook: HourlyCronJob

## Database Schema
```sql
CREATE TABLE mod_health_metrics (
    id INT AUTO_INCREMENT PRIMARY KEY,
    metric_type VARCHAR(100) NOT NULL,
    metric_value DECIMAL(10,2) NOT NULL,
    unit VARCHAR(50),
    recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_type_time (metric_type, recorded_at)
);

CREATE TABLE mod_health_checks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    check_name VARCHAR(255) NOT NULL,
    check_type VARCHAR(100),
    enabled TINYINT(1) DEFAULT 1,
    warning_threshold DECIMAL(10,2),
    critical_threshold DECIMAL(10,2),
    check_interval INT DEFAULT 60,
    last_check_at TIMESTAMP,
    last_status ENUM('ok', 'warning', 'critical', 'unknown') DEFAULT 'unknown'
);

CREATE TABLE mod_health_alerts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    check_id INT NOT NULL,
    severity ENUM('info', 'warning', 'critical') NOT NULL,
    message TEXT,
    metric_value DECIMAL(10,2),
    acknowledged TINYINT(1) DEFAULT 0,
    acknowledged_by INT,
    acknowledged_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_health_checks_history (
    id INT AUTO_INCREMENT PRIMARY KEY,
    check_id INT NOT NULL,
    status ENUM('ok', 'warning', 'critical', 'error') NOT NULL,
    response_time_ms INT,
    details TEXT,
    checked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_check_time (check_id, checked_at)
);
```

## API Endpoints
- GET /api/health/status - Get current status
- GET /api/health/metrics - Get metrics
- POST /api/health/checks/:id/acknowledge - Acknowledge alert
- GET /api/health/reports - Generate reports

## Testing Checklist
- [ ] Metric collection
- [ ] Threshold checking
- [ ] Alert generation
- [ ] Notification sending
- [ ] Report generation

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
