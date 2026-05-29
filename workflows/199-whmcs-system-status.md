---
name: whmcs-system-status
description: Check system status in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, status, system, monitoring]
---

# WHMCS System Status Workflow

## Purpose
Step-by-step guide for checking WHMCS system status.

## Prerequisites
- WHMCS admin access
- System monitoring setup

## Step 1: Access System Status
- Navigate to Utilities > System Health
- View overall system status
- Check status indicators

## Step 2: Review Status Dashboard
- Check all systems green
- Verify critical services
- Check integration status
- Review alert summary

## Step 3: Check Service Status
Individual service checks:
- Database connection
- Cron execution
- Payment gateways
- Registrar modules
- Email system

## Step 4: View Health Indicators
- System health score
- Performance metrics
- Error counts
- Response times
- Resource usage

## Step 5: Review Recent Issues
- Check recent alerts
- Review resolved issues
- View ongoing problems
- Check maintenance notices

## Step 6: Check Dependencies
- PHP extensions
- Required services
- External connections
- API accessibility
- Third-party services

## Step 7: Generate Status Report
- Document system status
- Note any issues
- Create action items
- Share status summary
- Archive for records

## Status Categories
- All Systems Operational
- Degraded Performance
- Partial Outage
- Major Outage
- Maintenance Mode

## Related Workflows
- whmcs-system-health
- whmcs-system-monitor
- whmcs-system-alert