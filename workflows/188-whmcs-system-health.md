---
name: whmcs-system-health
description: Perform system health check in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, health, monitoring, system]
---

# WHMCS System Health Check Workflow

## Purpose
Step-by-step guide for performing system health checks.

## Prerequisites
- WHMCS admin access
- System access
- Monitoring tools

## Step 1: Check System Status
- Navigate to Utilities > System Health
- Check WHMCS health status
- Review system status indicators
- Check for warnings

## Step 2: Verify Database Health
- Check database connection
- Verify table integrity
- Check query performance
- Review database size
- Monitor slow queries

## Step 3: Check Server Resources
- CPU usage
- Memory usage
- Disk space
- I/O operations
- Network activity

## Step 4: Review Application Status
- Check PHP version
- Verify extensions
- Check memory limit
- Review error logs
- Verify cache status

## Step 5: Test Core Functions
- Test login
- Test ordering
- Test payment processing
- Test email sending
- Test cron execution

## Step 6: Check Integration Status
- Verify payment gateways
- Check registrar modules
- Test server connections
- Verify API access
- Check webhooks

## Step 7: Generate Health Report
- Document findings
- Identify issues
- Prioritize actions
- Create remediation plan
- Schedule follow-up

## Health Check Areas
- Database Health
- Server Resources
- Application Status
- Integrations
- Security
- Performance

## Related Workflows
- whmcs-system-monitor
- whmcs-system-optimize
- whmcs-monitoring-setup