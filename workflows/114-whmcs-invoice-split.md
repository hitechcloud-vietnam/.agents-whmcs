---
name: whmcs-invoice-split
description: Split invoice in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, split, billing]
---

# WHMCS Invoice Split Workflow

## Purpose
Step-by-step guide for splitting a single invoice into multiple invoices in WHMCS.

## Prerequisites
- WHMCS admin access
- Large invoice with multiple items
- Client request or billing requirement
- No payments applied

## Step 1: Verify Split Eligibility
- Invoice status: Unpaid or Draft
- Multiple line items present
- No payments allocated
- No credits applied
- No related transactions

## Step 2: Access Split Function
- Navigate to WHMCS Admin > Billing > Invoices
- Open target invoice
- Click "Split Invoice" button
- Confirm action initiation

## Step 3: Define Split Strategy
Option A - By Category:
- Group items by type
- Separate services and products
- Separate recurring and one-time

Option B - By Amount:
- Split evenly
- Split by percentage
- Custom amount distribution

Option C - By Product Group:
- Hosting items together
- Domain items together
- Addon items together

## Step 4: Configure New Invoices
- Number of target invoices: 2+
- Each invoice needs:
  - Line items assignment
  - Date selection
  - Due date setting
  - Status (Draft/Sent)

## Step 5: Assign Line Items
- Drag and drop method:
  - Select items for Invoice 1
  - Select items for Invoice 2
  - Continue until all assigned
- Preview item distribution

## Step 6: Review Split Preview
- View all resulting invoices
- Check totals for each
- Verify no items unassigned
- Confirm split makes sense

## Step 7: Execute Split
- Click "Split Invoice" button
- System creates new invoices
- Original invoice marked "Split"
- Link records maintained

## Step 8: Post-Split Actions
- Review all new invoices
- Send to client if appropriate
- Update client notes
- Document split in audit trail

## Split Use Cases
- Client requests separate invoices
- Different payment methods
- Different billing addresses
- Department allocation

## Limitations
- Cannot split paid invoice
- Cannot split with applied payments
- All resulting invoices same client
- Original invoice reference preserved

## Related Workflows
- whmcs-invoice-merge
- whmcs-invoice-creation
- whmcs-invoice-editing