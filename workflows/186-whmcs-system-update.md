---
name: whmcs-system-update
description: Update WHMCS system
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, update, upgrade, system]
---

# WHMCS System Update Workflow

## Purpose
Step-by-step guide for updating WHMCS.

## Prerequisites
- WHMCS admin access
- Backup completed
- Update plan prepared

## Step 1: Check Current Version
- Login to WHMCS admin
- Check version in footer
- Review release notes
- Identify update needed

## Step 2: Review Update Requirements
- Read update changelog
- Check system requirements
- Verify PHP version
- Check MySQL version
- Review breaking changes

## Step 3: Prepare for Update
- Complete full backup
- Test in staging environment
- Review custom code impacts
- Document customizations
- Notify team

## Step 4: Download Update
- Login to WHMCS client area
- Download latest version
- Verify download integrity
- Extract update files

## Step 5: Apply Update
- Upload update files
- Run update installer
- Follow on-screen instructions
- Complete update process
- Clear cache

## Step 6: Verify Update
- Check version number
- Test core functionality
- Verify modules work
- Check custom code
- Test integrations

## Step 7: Post-Update Tasks
- Update custom code
- Run database updates
- Clear all caches
- Verify cron jobs
- Update monitoring

## Update Best Practices
- Always backup before update
- Test in staging first
- Update during low traffic
- Monitor for errors
- Keep custom code updated

## Related Workflows
- whmcs-upgrade-procedure
- whmcs-system-backup
- whmcs-system-restore