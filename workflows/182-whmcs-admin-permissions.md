---
name: whmcs-admin-permissions
description: Configure admin role permissions in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, admin, permissions, roles]
---

# WHMCS Admin Permissions Configuration Workflow

## Purpose
Step-by-step guide for configuring admin role permissions.

## Prerequisites
- WHMCS admin access
- Custom role needed
- Permission plan defined

## Step 1: Access Role Management
- Navigate to Configuration > Admin Roles
- View existing roles
- Click "Create New Role"

## Step 2: Configure Role Basics
- Enter role name
- Set role description
- Select base permissions
- Configure restrictions

## Step 3: Set Module Permissions
Grant/deny access to:
- Products/Services
- Support Tickets
- Billing/Invoices
- Domains
- Reports
- System Settings

## Step 4: Configure Action Permissions
Set allowed actions:
- Create/Edit/Delete
- View Only
- No Access
- Team Management
- Audit Access

## Step 5: Set Resource Restrictions
- Restrict client access
- Restrict product access
- Restrict report access
- Set department limitations
- Configure ticket limitations

## Step 6: Test Permissions
- Login as role user
- Verify access levels
- Test restricted areas
- Check action limits
- Confirm proper restrictions

## Step 7: Document Role
- Document role purpose
- List permissions
- Set approval workflow
- Update documentation

## Permission Categories
- Module Access
- Action Permissions
- Resource Restrictions
- Client Access
- Report Access

## Related Workflows
- whmcs-admin-management
- whmcs-admin-audit
- whmcs-system-security