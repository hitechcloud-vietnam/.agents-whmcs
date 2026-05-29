---
name: whmcs-admin-management
description: Manage admin users in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, admin, user-management, security]
---

# WHMCS Admin User Management Workflow

## Purpose
Step-by-step guide for managing admin users in WHMCS.

## Prerequisites
- WHMCS admin access
- Admin role permissions
- User management plan

## Step 1: Access Admin Management
- Navigate to Configuration > Admin Users
- View admin list
- Check existing users

## Step 2: Create New Admin
- Click "Add New Admin"
- Enter details:
  - Username
  - Email
  - Password
  - Name
  - Role
- Configure initial settings

## Step 3: Configure Admin Role
- Select role:
  - Full Administrator
  - Support Admin
  - Billing Admin
  - Read Only
  - Custom role
- Set permissions
- Configure restrictions

## Step 4: Set Admin Preferences
- Configure appearance
- Set language preference
- Set timezone
- Configure notification preferences
- Set email preferences

## Step 5: Configure Security
- Enable 2FA
- Set allowed IP addresses
- Configure session settings
- Set password requirements
- Enable audit logging

## Step 6: Manage Existing Admins
- Edit admin details
- Change passwords
- Update roles
- Disable accounts
- Remove access

## Step 7: Review Admin Activity
- Check admin audit log
- Review login history
- Monitor permission usage
- Verify compliance

## Admin Roles
- Super Admin: Full access
- Sales Admin: Sales-focused
- Support Admin: Support focus
- Billing Admin: Financial access
- Read Only: View only

## Related Workflows
- whmcs-admin-permissions
- whmcs-admin-audit
- whmcs-system-security