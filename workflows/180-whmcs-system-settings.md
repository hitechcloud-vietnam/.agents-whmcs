---
name: whmcs-system-settings
description: Configure WHMCS system settings
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, system, settings, configuration]
---

# WHMCS System Configuration Workflow

## Purpose
Step-by-step guide for configuring WHMCS system settings.

## Prerequisites
- WHMCS admin access
- System requirements known
- Configuration plan

## Step 1: Access System Settings
- Navigate to Setup > General
- View configuration sections:
  - Basic Settings
  - Security
  - Mail
  - Payments
  - Domains
  - Support

## Step 2: Configure Basic Settings
- Set company name
- Configure URL and paths
- Set timezone
- Configure date format
- Set language defaults
- Configure currency

## Step 3: Configure Security Settings
- Set admin directory
- Configure 2FA
- Set session timeouts
- Configure IP restrictions
- Set password policies
- Enable CSRF protection

## Step 4: Configure Email Settings
- Set email transport:
  - SMTP
  - PHP mail
  - SendGrid
  - Other
- Configure sender details
- Set email templates
- Configure bounce handling

## Step 5: Configure Display Settings
- Set admin theme
- Configure client theme
- Set logo and branding
- Configure template settings
- Set language files

## Step 6: Configure Regional Settings
- Set default country
- Configure states/provinces
- Set tax rules
- Configure currencies
- Set number format
- Set decimal places

## Step 7: Save and Test
- Save all settings
- Clear cache
- Test changes
- Verify functionality

## Configuration Areas
- General Settings
- Security Settings
- Email Configuration
- Payment Configuration
- Domain Settings
- Support Settings
- Automation Settings

## Related Workflows
- whmcs-system-security
- whmcs-system-email
- whmcs-system-cron