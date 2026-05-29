---
name: whmcs-system-security
description: Configure system security settings in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, security, configuration, system]
---

# WHMCS System Security Configuration Workflow

## Purpose
Step-by-step guide for configuring WHMCS security settings.

## Prerequisites
- WHMCS admin access
- Security requirements identified
- Security best practices knowledge

## Step 1: Access Security Settings
- Navigate to Configuration > Security
- View security options
- Review current settings

## Step 2: Configure Admin Security
- Change admin directory
- Enable two-factor authentication
- Set session timeout
- Configure IP restrictions
- Set password policy

## Step 3: Configure Client Security
- Enable client 2FA
- Set password requirements
- Configure login limits
- Enable captcha
- Set session management

## Step 4: Configure Data Security
- Enable HTTPS enforcement
- Configure SSL settings
- Set cookie security
- Enable CSRF protection
- Configure XSS protection

## Step 5: Configure File Security
- Set file permissions
- Configure upload restrictions
- Enable secure file serving
- Set allowed file types
- Configure upload limits

## Step 6: Configure Network Security
- Set up firewall rules
- Configure API restrictions
- Enable rate limiting
- Set up DDoS protection
- Configure WAF settings

## Step 7: Security Monitoring
- Enable security logging
- Configure alert thresholds
- Set up intrusion detection
- Monitor failed logins
- Review security reports

## Security Checklist
- Change admin URL
- Enable 2FA everywhere
- Use strong passwords
- Enable SSL
- Restrict IP access
- Enable logging
- Regular security audits

## Related Workflows
- whmcs-security-hardening
- whmcs-security-audit
- whmcs-access-control