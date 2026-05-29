---
name: whmcs-system-alert
description: Configure system alerts in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, alerts, notifications, monitoring]
---

# WHMCS System Alert Configuration Workflow

## Purpose
Step-by-step guide for configuring system alerts.

## Prerequisites
- WHMCS admin access
- Alert requirements identified
- Notification channels configured

## Step 1: Access Alert Settings
- Navigate to Configuration > System > Alerts
- Or Setup > Notifications
- View alert categories

## Step 2: Configure Alert Types
Set alerts for:
- System errors
- Security alerts
- Payment failures
- Domain expirations
- Service terminations
- Cron failures

## Step 3: Set Alert Thresholds
Define trigger points:
- Error count threshold
- Response time threshold
- Resource usage threshold
- Failed payment threshold
- Time-based triggers

## Step 4: Configure Alert Channels
- Email notifications
- SMS notifications
- Slack/Discord webhooks
- Push notifications
- Admin panel alerts

## Step 5: Set Alert Recipients
- Primary admin
- Support team
- Billing team
- On-call personnel
- Custom recipients

## Step 6: Test Alerts
- Send test alert
- Verify delivery
- Check formatting
- Confirm recipients receive
- Adjust if needed

## Step 7: Manage Alert Rules
- Enable/disable alerts
- Adjust thresholds
- Modify recipients
- Review alert history
- Optimize alert noise

## Alert Categories
- System Alerts
- Security Alerts
- Billing Alerts
- Domain Alerts
- Service Alerts
- Performance Alerts

## Related Workflows
- whmcs-system-monitor
- whmcs-system-log
- whmcs-monitoring-setup