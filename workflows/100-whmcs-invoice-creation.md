---
name: whmcs-invoice-creation
description: Create invoice in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, billing]
---

# WHMCS Invoice Creation Workflow

## Purpose
Step-by-step guide for creating invoices in WHMCS system.

## Prerequisites
- WHMCS admin access
- Valid client account
- Product/service configuration

## Step 1: Access Invoice Module
- Navigate to WHMCS Admin > Billing > Invoices
- Click "Create New Invoice" button

## Step 2: Select Client
- Search and select client from dropdown
- Verify client details: billing address, tax ID
- Confirm client currency matches invoice currency

## Step 3: Add Invoice Items
- Add line items: products, services, or custom charges
- Enter description for each item
- Set quantity and unit price
- Apply tax if applicable

## Step 4: Configure Invoice Settings
- Set invoice date (due date defaults based on terms)
- Select payment terms
- Set invoice status: Draft or Sent
- Add internal notes (optional)

## Step 5: Review and Finalize
- Review all line items
- Verify tax calculations
- Check subtotal, tax, and total
- Click "Save Invoice"

## Step 6: Send Invoice (Optional)
- Select email template
- Preview email content
- Click "Send Invoice" to email client

## Post-Creation Actions
- Record invoice ID for reference
- Add to accounts receivable tracking
- Set reminder for follow-up if unpaid

## Related Workflows
- whmcs-invoice-payment
- whmcs-invoice-reminder
- whmcs-invoice-email