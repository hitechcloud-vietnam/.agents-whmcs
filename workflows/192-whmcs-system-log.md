---
name: whmcs-system-log
description: Manage system logs in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, logs, system, debugging]
---

# WHMCS System Log Management Workflow

## Purpose
Step-by-step guide for managing system logs.

## Prerequisites
- WHMCS admin access
- Log access permissions
- Log analysis requirements

## Step 1: Access System Logs
- Navigate to Utilities > Logs
- View available logs:
  - Activity Log
  - Admin Log
  - Module Log
  - Email Log
  - API Log
  - Error Log

## Step 2: Configure Log Settings
- Set log retention period
- Enable/disable logging
- Configure log levels
- Set log rotation
- Configure log storage

## Step 3: Review Activity Log
- Check admin actions
- Review client activities
- Monitor system changes
- Track configuration changes
- Verify user actions

## Step 4: Review Error Log
- Check PHP errors
- Review module errors
- Identify database issues
- Monitor API errors
- Track payment errors

## Step 5: Analyze Logs
- Search for specific events
- Filter by date range
- Filter by user
- Filter by severity
- Filter by module

## Step 6: Export Logs
- Export for analysis
- Generate log reports
- Archive old logs
- Create audit reports
- Store for compliance

## Step 7: Implement Log Cleanup
- Delete old logs
- Archive critical logs
- Set automatic cleanup
- Verify cleanup works
- Document retention

## Log Types
- Activity Log: User actions
- Admin Log: Admin operations
- Module Log: Module operations
- Email Log: Email history
- API Log: API requests
- Error Log: System errors

## Related Workflows
- whmcs-system-monitor
- whmcs-system-alert
- whmcs-log-analysis