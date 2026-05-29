---
name: whmcs-system-email
description: Configure system email in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, email, configuration, system]
---

# WHMCS System Email Configuration Workflow

## Purpose
Step-by-step guide for configuring WHMCS email.

## Prerequisites
- WHMCS admin access
- Email service ready
- DNS configured

## Step 1: Access Email Settings
- Navigate to Configuration > Email
- View email configuration
- Check current setup

## Step 2: Configure Email Transport
Select email method:
- PHP mail (default)
- SMTP relay
- SendGrid
- Amazon SES
- Mailgun
- Custom SMTP

## Step 3: Configure SMTP
If using SMTP:
- Enter SMTP host
- Enter SMTP port
- Set encryption (TLS/SSL)
- Enter username
- Enter password
- Set from email

## Step 4: Configure Email Settings
- Set default sender name
- Set default sender email
- Configure reply-to address
- Set email separator
- Configure HTML/Plain text

## Step 5: Configure Bounce Handling
- Set bounce email address
- Configure bounce rules
- Enable bounce processing
- Set bounce threshold
- Configure auto-delete

## Step 6: Configure Spam Protection
- Enable DKIM
- Configure SPF
- Set DMARC records
- Enable spam filtering
- Configure email limits

## Step 7: Test Email Configuration
- Send test email
- Verify delivery
- Check spam score
- Verify bounce handling
- Monitor email queue

## Email Best Practices
- Use SMTP for reliability
- Set up proper authentication
- Monitor deliverability
- Track bounce rates
- Test email templates

## Related Workflows
- whmcs-system-email
- whmcs-email-config
- whmcs-email-server