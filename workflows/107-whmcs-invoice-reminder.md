---
name: whmcs-invoice-reminder
description: Send invoice reminder in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, reminder, billing]
---

# WHMCS Invoice Reminder Workflow

## Purpose
Step-by-step guide for sending invoice reminders in WHMCS.

## Prerequisites
- WHMCS admin access
- Unpaid invoice
- Email template configured
- Client email address

## Step 1: Access Invoice Reminder
- Navigate to WHMCS Admin > Billing > Invoices
- Find unpaid invoice
- Open invoice details

## Step 2: Verify Reminder Prerequisites
- Confirm invoice is overdue or due soon
- Check client email is valid
- Verify reminder hasn't been sent recently

## Step 3: Select Reminder Template
- WHMCS provides default templates:
  - First Reminder
  - Second Reminder
  - Late Fee Notice
  - Final Notice
- Or custom template configured

## Step 4: Preview Reminder
- View email content preview
- Check personalization tokens:
  - {$client_name}
  - {$invoice_num}
  - {$invoice_total}
  - {$due_date}
- Adjust if needed

## Step 5: Customize Message (Optional)
- Add personal note
- Include payment instructions
- Add contact information
- Modify tone as appropriate

## Step 6: Send Reminder
- Click "Send Reminder" button
- System sends email via configured mailer
- Confirmation displayed
- Reminder logged in history

## Step 7: Follow-Up Actions
- Schedule next reminder if needed
- Escalate to collections if overdue
- Create support ticket for inquiry
- Update client notes

## Automated Reminders
Configure in WHMCS:
- Setup > Automation > Invoice Reminders
- Set reminder schedule:
  - X days before due
  - On due date
  - X days after due
  - Before late fee

## Reminder Types
1. Advance Notice: Before due date
2. Due Date Reminder: On due date
3. First Overdue: 1-7 days overdue
4. Second Overdue: 7-14 days overdue
5. Final Notice: Before suspension
6. Suspension Notice: Service suspension warning

## Related Workflows
- whmcs-invoice-email
- whmcs-invoice-payment
- whmcs-system-email