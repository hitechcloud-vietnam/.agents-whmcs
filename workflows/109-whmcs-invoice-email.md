---
name: whmcs-invoice-email
description: Email invoice to client in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, email, billing]
---

# WHMCS Invoice Email Workflow

## Purpose
Step-by-step guide for emailing invoices to clients in WHMCS.

## Prerequisites
- WHMCS admin access
- Invoice ID
- Client email address
- Email template configured
- Mail system operational

## Step 1: Access Invoice Email
- Navigate to WHMCS Admin > Billing > Invoices
- Locate and open target invoice
- Click "Email Invoice" button

## Step 2: Select Email Template
- Choose from available templates:
  - Invoice - Standard
  - Invoice - With Payment Link
  - Invoice - Reminder
  - Custom templates
- Preview template content

## Step 3: Configure Email Recipients
- Primary: Client email (default)
- CC: Additional recipients
- BCC: Internal copies
- Verify email addresses valid

## Step 4: Customize Email Content
- Edit subject line if needed
- Add personal message
- Include payment instructions
- Add contact information
- Modify body content

## Step 5: Attach Files
- Invoice PDF (auto-attached)
- Additional documents:
  - Contract
  - Terms of Service
  - Receipt copy
- Limit attachment size

## Step 6: Send Email
- Click "Send Email" button
- System validates recipient
- WHMCS sends via configured mailer
- Confirmation displayed

## Step 7: Email Verification
- Check sent folder (if BCC to self)
- Verify client receipt
- Monitor for bounce-backs
- Log email in client history

## Automated Invoice Emails
Configure in System > Automation:
- Invoice Created
- Invoice Reminder (multiple)
- Invoice Overdue
- Payment Received
- Refund Processed

## Email Template Variables
{$client_name}
{$invoice_num}
{$invoice_date}
{$invoice_due_date}
{$invoice_total}
{$invoice_balance}
{$payment_link}
{$company_name}
{$company_logo}

## Troubleshooting
- Check spam folders
- Verify email limits
- Test email configuration
- Monitor delivery status

## Related Workflows
- whmcs-invoice-print
- whmcs-invoice-reminder
- whmcs-system-email