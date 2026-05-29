# WHMCS Scheduled Task Runner Devkit

## Overview
A scheduled task runner for WHMCS that manages cron jobs, task scheduling, dependency management, and execution monitoring with queue support.

## Features
- Task scheduling
- Cron job management
- Task dependencies
- Queue management
- Execution monitoring
- Failure handling
- Task retry
- Task priorities
- Time-based scheduling
- Event-based scheduling
- Task templates
- Execution history

## WHMCS Integration Points
- Module: addon/ScheduleTasks
- Hook: DailyCronJob
- Hook: HourlyCronJob

## Database Schema
```sql
CREATE TABLE mod_scheduled_tasks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    handler_class VARCHAR(255) NOT NULL,
    schedule_type ENUM('cron', 'interval', 'once') DEFAULT 'cron',
    cron_expression VARCHAR(100),
    interval_seconds INT,
    run_at DATETIME,
    priority INT DEFAULT 0,
    timeout_seconds INT DEFAULT 300,
    retry_count INT DEFAULT 0,
    retry_delay INT DEFAULT 60,
    is_active TINYINT(1) DEFAULT 1,
    last_run_at TIMESTAMP,
    next_run_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_task_queue (
    id INT AUTO_INCREMENT PRIMARY KEY,
    task_id INT NOT NULL,
    status ENUM('pending', 'running', 'completed', 'failed', 'retrying') DEFAULT 'pending',
    payload JSON,
    priority INT DEFAULT 0,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    error_message TEXT,
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_status (status)
);

CREATE TABLE mod_task_execution_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    task_id INT NOT NULL,
    queue_id INT,
    execution_time_ms INT,
    memory_used INT,
    peak_memory INT,
    status ENUM('success', 'failed', 'timeout') NOT NULL,
    output TEXT,
    error_trace TEXT,
    executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_task (task_id)
);
```

## Testing Checklist
- [ ] Task scheduling
- [ ] Cron execution
- [ ] Queue processing
- [ ] Retry mechanism
- [ ] Timeout handling

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
