---
name: whmcs-invoice-batch
description: Batch invoice generation in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, batch, billing]
---

# WHMCS Batch Invoice Generation Workflow

## Purpose
Step-by-step guide for generating multiple invoices in batch in WHMCS.

## Prerequisites
- WHMCS admin access
- Products/services configured
- Billing cycles set
- Automation cron configured

## Step 1: Prepare for Batch Generation
- Navigate to WHMCS Admin > Billing > Invoices
- Click "Generate Invoices"
- Or use System > Automation > Generate Invoices

## Step 2: Select Batch Options
- Generation scope:
  - All due invoices
  - Specific product groups
  - Selected clients
  - Date range
- Include items:
  - Recurring services
  - Addon products
  - Custom charges

## Step 3: Configure Invoice Settings
- Invoice date: Current date (default)
- Due date: Based on payment terms
- Payment terms: Set default days
- Invoice prefix: Configure if needed
- Auto-increment: Yes/No

## Step 4: Apply Filters
- Client status: Active only
- Billing cycle: Monthly, Quarterly, etc.
- Product type: Specific products
- Exclude: Suspended services, trial products

## Step 5: Preview Batch
- View estimated invoice count
- Review total revenue
- Check for errors in preview
- Adjust filters if needed

## Step 6: Execute Batch Generation
- Click "Generate Invoices"
- Monitor progress bar
- View generation results:
  - Invoices created
  - Errors encountered
  - Warnings (duplicate check)

## Step 7: Post-Generation Actions
- Review generated invoices
- Check for issues
- Send invoices in batch:
  - Select all new invoices
  - Click "Send Selected"
  - Confirm batch send
- Update reports

## Cron-Based Automation
Configure in Setup > Automation:
- Daily invoice generation at midnight
- Auto-send invoices option
- Generate reminders automatically
- Late fee application

## Batch Operations Available
- Generate all due invoices
- Generate specific product group
- Generate selected client invoices
- Generate renewal notices

## Monitoring
- Check cron logs
- Monitor failed generations
- Track invoice totals
- Review error reports

## Related Workflows
- whmcs-invoice-creation
- whmcs-invoice-email
- whmcs-cron-configuration
- whmcs-billing-automation