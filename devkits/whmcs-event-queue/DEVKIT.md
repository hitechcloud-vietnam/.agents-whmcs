# WHMCS Async Event Processing Devkit

## Overview
An asynchronous event processing system for WHMCS that queues events, processes them in background workers, and handles event distribution with retry logic.

## Features
- Event queuing
- Background processing
- Event handlers
- Priority queues
- Event filtering
- Batch processing
- Dead letter queue
- Event replay
- Monitoring dashboard
- Rate limiting
- Concurrency control
- Event scheduling

## WHMCS Integration Points
- Module: addon/EventQueue
- Hook: Multiple hooks
- Queue worker process

## Database Schema
```sql
CREATE TABLE mod_event_queue (
    id INT AUTO_INCREMENT PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    event_data JSON NOT NULL,
    priority INT DEFAULT 0,
    status ENUM('pending', 'processing', 'completed', 'failed', 'dead_letter') DEFAULT 'pending',
    attempts INT DEFAULT 0,
    max_attempts INT DEFAULT 3,
    handler_class VARCHAR(255),
    scheduled_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_status (status),
    INDEX idx_type (event_type)
);

CREATE TABLE mod_event_handlers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    handler_class VARCHAR(255) NOT NULL,
    priority INT DEFAULT 0,
    is_active TINYINT(1) DEFAULT 1,
    config JSON
);

CREATE TABLE mod_dead_letter_events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    original_event_id INT NOT NULL,
    event_type VARCHAR(100),
    event_data JSON,
    error_message TEXT,
    failed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Testing Checklist
- [ ] Event queuing
- [ ] Background processing
- [ ] Retry logic
- [ ] Dead letter handling
- [ ] Event replay

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- Redis for queue (optional)
