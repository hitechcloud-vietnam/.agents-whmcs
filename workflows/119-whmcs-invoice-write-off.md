---
name: whmcs-invoice-write-off
description: Write-off invoice in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, write-off, billing]
---

# WHMCS Invoice Write-Off Workflow

## Purpose
Step-by-step guide for writing off bad debts/invoices in WHMCS.

## Prerequisites
- WHMCS admin access
- Unpaid/overdue invoice
- Write-off authorization
- Documentation of collection attempts

## Step 1: Verify Write-Off Eligibility
- Invoice overdue (typically 90+ days)
- Collection attempts exhausted
- Client unresponsive
- Bankruptcy/business closure
- Economic hardship documented

## Step 2: Review Collection History
- Check payment attempts
- Review client communication
- Verify reminder history
- Assess debt recoverability

## Step 3: Obtain Authorization
- Manager approval required
- Document write-off reason
- Set approval threshold
- Follow company policy

## Step 4: Configure Write-Off
- Navigate to invoice
- Click "Write-Off" button
- Select write-off type:
  - Full write-off
  - Partial write-off
  - Settlement amount
- Enter final amount

## Step 5: Account Treatment
Choose accounting treatment:
A. Bad Debt Write-Off:
   - Expense to P&L
   - Remove from A/R
   - Tax deduction (if applicable)

B. Partial Settlement:
   - Accept reduced amount
   - Document agreement
   - Write-off difference

C. Suspended Account:
   - Keep on books
   - Future recovery possible
   - Reassess periodically

## Step 6: Execute Write-Off
- Confirm write-off amount
- Add write-off notes
- Click "Write-Off Invoice"
- System marks invoice written off

## Step 7: Post-Write-Off Actions
- Update client status
- Document in accounting
- Update reports
- Log in audit trail
- Set review for recovery

## Write-Off Documentation
Required records:
- Original invoice amount
- Payment history
- Collection attempts
- Authorization approval
- Write-off reason
- Date written off

## Write-Off Prevention
Before write-off:
- Offer payment plan
- Send final notice
- Contact collections agency
- Negotiate settlement
- Apply late fees

## Tax Implications
- Consult accountant
- Document for tax purposes
- Some jurisdictions allow deduction
- Maintain backup documentation

## Related Workflows
- whmcs-invoice-void
- whmcs-invoice-credit-note
- whmcs-invoice-payment-reversal