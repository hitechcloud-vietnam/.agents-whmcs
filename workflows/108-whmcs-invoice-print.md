---
name: whmcs-invoice-print
description: Print invoice in WHMCS
trigger: agent invoke
type: workflow
author: Claude Agent
created: 2026-05-29
tags: [whmcs, invoice, print, billing]
---

# WHMCS Invoice Print Workflow

## Purpose
Step-by-step guide for printing invoices in WHMCS.

## Prerequisites
- WHMCS admin access
- Invoice ID
- Printer configured
- PDF viewer (for PDF export)

## Step 1: Locate Invoice
- Navigate to WHMCS Admin > Billing > Invoices
- Search and open target invoice
- Verify invoice details are correct

## Step 2: Access Print Options
- Click "Print" button in invoice toolbar
- Dropdown shows options:
  - Print Invoice
  - Print with Payment Details
  - Print with Notes
  - PDF Export

## Step 3: Select Print Template
- Default template (standard layout)
- Custom templates (if configured)
- Pre-printed stationery option
- Check paper size compatibility

## Step 4: Preview Invoice
- System generates preview
- Review layout and formatting
- Check all line items present
- Verify totals and taxes
- Review header/footer

## Step 5: Configure Print Settings
- Paper size: A4, Letter, Legal
- Orientation: Portrait, Landscape
- Margins: Standard, Narrow
- Color: Color or Grayscale

## Step 6: Print Invoice
- Click "Print" in browser dialog
- Select appropriate printer
- Configure paper source
- Print single or multiple copies

## Step 7: PDF Export Option
- Click "Export PDF" button
- Save PDF to desired location
- Email to client
- Store for records

## Print Variations
- With Payment Slip
- With Bank Details
- Tax Invoice Format
- Simplified Format
- Detailed Itemized

## Bulk Printing
- Select multiple invoices
- Use "Print Queue" feature
- Generate combined PDF
- Batch print operation

## Related Workflows
- whmcs-invoice-email
- whmcs-invoice-export
- whmcs-invoice-template-design