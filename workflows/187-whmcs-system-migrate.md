---
name: whmcs-system-migrate
description: Migrate WHMCS to new server
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, migration, server, transfer]
---

# WHMCS System Migration Workflow

## Purpose
Step-by-step guide for migrating WHMCS to a new server.

## Prerequisites
- New server ready
- Migration plan
- Domain update capability
- Downtime window

## Step 1: Prepare Migration Plan
- Document current setup
- Identify new server specs
- Plan DNS change
- Set migration timeline
- Prepare rollback plan

## Step 2: Backup Current Installation
- Complete full backup
- Download backup files
- Verify backup integrity
- Document configuration
- Export all data

## Step 3: Prepare New Server
- Install LAMP/LEMP stack
- Configure PHP version
- Setup MySQL database
- Configure web server
- Set SSL certificate
- Test environment

## Step 4: Transfer Files
- Upload WHMCS files
- Set file permissions
- Configure directories
- Upload custom files
- Verify file integrity

## Step 5: Transfer Database
- Create database
- Import database backup
- Update configuration
- Verify connections
- Test database access

## Step 6: Update Configuration
- Update configuration.php
- Set database credentials
- Update paths
- Update URLs
- Clear cache

## Step 7: Test New Installation
- Test admin access
- Verify client area
- Check functionality
- Test ordering
- Verify integrations

## Step 8: DNS Switch
- Update DNS records
- Monitor propagation
- Verify new server active
- Test from multiple locations
- Monitor for issues

## Step 9: Post-Migration
- Verify all data
- Test payment processing
- Update monitoring
- Update SSL
- Document changes

## Related Workflows
- whmcs-migration-guide
- whmcs-system-backup
- whmcs-system-restore