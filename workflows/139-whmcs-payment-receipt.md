---
name: whmcs-payment-receipt
description: Generate payment receipt in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, payment, receipt, billing]
---

# WHMCS Payment Receipt Generation Workflow

## Purpose
Step-by-step guide for generating payment receipts in WHMCS.

## Prerequisites
- WHMCS admin access
- Completed payment
- Receipt template configured

## Step 1: Access Payment Record
- Navigate to WHMCS Admin > Billing > Transactions
- Search for payment
- Open transaction details

## Step 2: Generate Receipt
- Click "Generate Receipt"
- Select receipt template
- Preview receipt content

## Step 3: Configure Receipt
- Set receipt number
- Add payment details
- Include transaction reference
- Add company information

## Step 4: Review Receipt Content
Check receipt includes:
- Receipt number
- Payment date
- Amount paid
- Payment method
- Invoice number(s)
- Client information
- Company details

## Step 5: Send Receipt
- Click "Email Receipt"
- Enter recipient email
- Add personal message
- Click "Send"

## Step 6: Download Receipt
- Click "Download PDF"
- Save to local system
- Store for records

## Step 7: Receipt Options
Available formats:
- PDF receipt
- Email receipt
- Print receipt
- Combined invoice + receipt

## Receipt Template Customization
- Access via Configuration > Billing > Invoice Templates
- Customize receipt layout
- Add logo and branding
- Modify fields displayed

## Automatic Receipts
Configure automatic receipt:
- Setup > Automation > Notifications
- Enable auto-receipt
- Set receipt template
- Configure triggers

## Related Workflows
- whmcs-invoice-payment
- whmcs-payment-accept
- whmcs-invoice-print