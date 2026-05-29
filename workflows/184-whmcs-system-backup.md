---
name: whmcs-system-backup
description: Perform system backup in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, backup, system, disaster-recovery]
---

# WHMCS System Backup Workflow

## Purpose
Step-by-step guide for performing WHMCS backups.

## Prerequisites
- WHMCS admin access
- Backup storage configured
- Scheduled backup plan

## Step 1: Plan Backup Strategy
- Determine backup scope:
  - Full backup
  - Database only
  - Files only
- Set backup frequency:
  - Daily
  - Weekly
  - Real-time
- Define retention period

## Step 2: Configure Backup Settings
- Navigate to Configuration > System Backup
- Set backup destination:
  - Local directory
  - Remote server
  - Cloud storage (S3, Dropbox)
- Configure compression
- Set encryption if needed

## Step 3: Execute Manual Backup
- Click "Run Backup Now"
- Select backup type
- Monitor backup progress
- Verify completion

## Step 4: Verify Backup Integrity
- Check backup file created
- Verify file size
- Test backup restoration
- Confirm all data included

## Step 5: Configure Automated Backup
- Setup > Automation > Backup
- Set backup schedule
- Configure retention
- Enable notifications

## Step 6: Store Backup Securely
- Store off-site
- Encrypt backup files
- Limit access to backups
- Test restore from backup

## Step 7: Document Backup
- Record backup dates
- Document backup locations
- Note restoration procedures
- Update disaster recovery plan

## Backup Components
- Database (MySQL)
- Configuration files
- Uploaded files
- Email templates
- Custom code

## Related Workflows
- whmcs-system-restore
- whmcs-backup-restore
- whmcs-disaster-recovery