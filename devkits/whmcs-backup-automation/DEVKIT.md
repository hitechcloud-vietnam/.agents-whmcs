# WHMCS Automated Backup System Devkit

## Overview
An automated backup system for WHMCS that handles database backups, file backups, off-site storage, backup rotation, and restore capabilities.

## Features
- Database backup
- File backup
- Incremental backups
- Full backups
- Backup compression
- Off-site storage (S3, FTP, etc.)
- Backup rotation
- Backup scheduling
- Backup verification
- Restore capabilities
- Backup encryption
- Backup monitoring

## WHMCS Integration Points
- Module: addon/BackupAutomation
- Hook: DailyCronJob
- Storage integration

## Database Schema
```sql
CREATE TABLE mod_backup_jobs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    backup_type ENUM('full', 'incremental', 'database_only', 'files_only') DEFAULT 'full',
    schedule_type ENUM('daily', 'weekly', 'monthly') DEFAULT 'daily',
    schedule_time TIME,
    schedule_day INT,
    retention_count INT DEFAULT 7,
    compression_enabled TINYINT(1) DEFAULT 1,
    encryption_enabled TINYINT(1) DEFAULT 0,
    storage_destination VARCHAR(255),
    is_active TINYINT(1) DEFAULT 1,
    last_run_at TIMESTAMP,
    next_run_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_backup_history (
    id INT AUTO_INCREMENT PRIMARY KEY,
    job_id INT NOT NULL,
    backup_file VARCHAR(500),
    file_size BIGINT,
    compressed_size BIGINT,
    status ENUM('in_progress', 'completed', 'failed', 'verified') DEFAULT 'pending',
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    verified_at TIMESTAMP,
    checksum VARCHAR(64),
    error_message TEXT,
    INDEX idx_job (job_id)
);

CREATE TABLE mod_backup_storage (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    storage_type ENUM('local', 'ftp', 's3', 'azure', 'google_cloud') NOT NULL,
    config JSON NOT NULL,
    is_default TINYINT(1) DEFAULT 0,
    is_active TINYINT(1) DEFAULT 1,
    used_bytes BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_backup_files (
    id INT AUTO_INCREMENT PRIMARY KEY,
    backup_id INT NOT NULL,
    file_path VARCHAR(500),
    file_type VARCHAR(50),
    size BIGINT,
    included_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## API Endpoints
- POST /api/backups/jobs - Create backup job
- GET /api/backups/jobs/:id - Get job status
- POST /api/backups/jobs/:id/run - Run backup now
- POST /api/backups/:id/restore - Restore backup
- GET /api/backups/history - Get backup history

## Testing Checklist
- [ ] Database backup
- [ ] File backup
- [ ] Compression
- [ ] Storage upload
- [ ] Backup rotation
- [ ] Restore verification

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- zip extension
- Storage provider SDK
