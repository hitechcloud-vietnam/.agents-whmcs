# WHMCS Invoice Template Design Workflow

## Purpose
Guide developers through customizing invoice templates in WHMCS.

## Prerequisites
- WHMCS installation
- Invoice PDF generation knowledge
- HTML/CSS for PDF templates
- TCPDF or Dompdf understanding

## Steps

### Phase 1: Invoice Template Structure

1. Invoice template locations
   ```
   /whmcs/templates/invoice/
   ├── invoice-pdf.html
   ├── invoice-pdf-modern.html
   └── invoice-email.html
   ```

2. PDF generation system
   ```
   WHMCS Invoice PDFs:
   - TCPDF library (legacy)
   - Dompdf (modern)
   - PDF template system
   - Custom fonts support
   ```

### Phase 2: PDF Invoice Basics

1. Basic invoice template
   ```html
   <!DOCTYPE html>
   <html>
   <head>
       <meta charset="UTF-8">
       <style>
           body { font-family: 'DejaVu Sans', sans-serif; }
           .invoice-header { text-align: center; padding: 20px; }
           .invoice-items { width: 100%; border-collapse: collapse; }
           .invoice-items th { background: #f0f0f0; padding: 10px; text-align: left; }
           .invoice-items td { padding: 10px; border-bottom: 1px solid #ddd; }
           .invoice-total { text-align: right; margin-top: 20px; }
       </style>
   </head>
   <body>
       <!-- Invoice content -->
   </body>
   </html>
   ```

### Phase 3: Invoice Header

1. Company header
   ```html
   <table width="100%" cellpadding="0" cellspacing="0" style="margin-bottom: 30px;">
       <tr>
           <td width="50%">
               <img src="{$logo_url}" alt="{$companyname}" style="max-height: 60px;">
           </td>
           <td width="50%" style="text-align: right;">
               <h1 style="font-size: 24px; color: #333; margin: 0;">INVOICE</h1>
               <p style="color: #666; margin: 5px 0;">#{$invoice_number}</p>
           </td>
       </tr>
   </table>
   ```

2. Invoice info block
   ```html
   <table width="100%" cellpadding="0" cellspacing="0" style="margin-bottom: 30px;">
       <tr>
           <td width="50%">
               <strong>From:</strong><br>
               {$companyname}<br>
               {$companyaddress|nl2br}
           </td>
           <td width="50%" style="text-align: right;">
               <strong>Invoice Date:</strong> {$invoice_date}<br>
               <strong>Due Date:</strong> {$due_date}<br>
               <strong>Status:</strong> {$status}
           </td>
       </tr>
   </table>
   ```

### Phase 4: Client Information

1. Client details
   ```html
   <table width="100%" cellpadding="0" cellspacing="0" style="margin-bottom: 30px; background: #f9f9f9; padding: 15px;">
       <tr>
           <td>
               <strong>Bill To:</strong><br>
               {$client_name}<br>
               {if $client_company}{$client_company}<br>{/if}
               {$client_address|nl2br}
           </td>
       </tr>
   </table>
   ```

### Phase 5: Invoice Items Table

1. Items table structure
   ```html
   <table class="invoice-items" width="100%" cellpadding="5" cellspacing="0" style="border: 1px solid #ddd; margin-bottom: 20px;">
       <thead>
           <tr style="background: #333; color: #fff;">
               <th style="padding: 12px; text-align: left;">Description</th>
               <th style="padding: 12px; text-align: center;">Qty</th>
               <th style="padding: 12px; text-align: right;">Unit Price</th>
               <th style="padding: 12px; text-align: right;">Total</th>
           </tr>
       </thead>
       <tbody>
           {foreach $lineitems as $item}
           <tr style="border-bottom: 1px solid #eee;">
               <td style="padding: 10px;">{$item.description}</td>
               <td style="padding: 10px; text-align: center;">{$item.quantity}</td>
               <td style="padding: 10px; text-align: right;">{$item.unitprice}</td>
               <td style="padding: 10px; text-align: right;">{$item.total}</td>
           </tr>
           {/foreach}
       </tbody>
   </table>
   ```

### Phase 6: Invoice Totals

1. Totals section
   ```html
   <table width="100%" cellpadding="0" cellspacing="0" style="margin-top: 20px;">
       <tr>
           <td width="60%"></td>
           <td width="40%">
               <table width="100%" cellpadding="5" cellspacing="0">
                   <tr>
                       <td style="padding: 8px;">Subtotal:</td>
                       <td style="padding: 8px; text-align: right;">{$subtotal}</td>
                   </tr>
                   {if $tax_rate}
                   <tr>
                       <td style="padding: 8px;">Tax ({$tax_rate}%):</td>
                       <td style="padding: 8px; text-align: right;">{$tax_amount}</td>
                   </tr>
                   {/if}
                   <tr style="font-weight: bold; font-size: 16px;">
                       <td style="padding: 12px; border-top: 2px solid #333;">Total:</td>
                       <td style="padding: 12px; border-top: 2px solid #333; text-align: right;">{$total}</td>
                   </tr>
                   {if $credit}
                   <tr style="color: green;">
                       <td style="padding: 8px;">Credit Applied:</td>
                       <td style="padding: 8px; text-align: right;">-{$credit}</td>
                   </tr>
                   <tr style="font-weight: bold;">
                       <td style="padding: 12px; border-top: 2px solid #333;">Balance Due:</td>
                       <td style="padding: 12px; border-top: 2px solid #333; text-align: right;">{$balance_due}</td>
                   </tr>
                   {/if}
               </table>
           </td>
       </tr>
   </table>
   ```

### Phase 7: Payment Information

1. Payment instructions
   ```html
   <div style="background: #f5f5f5; padding: 20px; margin-top: 30px; border-radius: 5px;">
       <h3 style="margin-top: 0;">Payment Information</h3>
       {if $payment_methods}
       <p>Please pay using one of the following methods:</p>
       <ul>
           {foreach $payment_methods as $method}
           <li>{$method}</li>
           {/foreach}
       </ul>
       {/if}
       <p><strong>Bank Transfer:</strong><br>
       Bank: Example Bank<br>
       Account: 123456789<br>
       Routing: 123456789</p>
   </div>
   ```

### Phase 8: Invoice Footer

1. Footer content
   ```html
   <div style="text-align: center; margin-top: 40px; padding-top: 20px; border-top: 1px solid #ddd;">
       <p style="color: #666; font-size: 12px;">
           {$companyname} | {$companyaddress}<br>
           <a href="mailto:{$company_email}" style="color: #666;">{$company_email}</a>
       </p>
       <p style="color: #999; font-size: 11px;">
           Thank you for your business!
       </p>
   </div>
   ```

### Phase 9: Modern Invoice Design

1. Gradient header design
   ```css
   .invoice-header {
       background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
       color: white;
       padding: 30px;
       border-radius: 8px;
       margin-bottom: 30px;
   }
   ```

2. Card-style design
   ```css
   .invoice-card {
       border: 1px solid #e0e0e0;
       border-radius: 12px;
       box-shadow: 0 2px 10px rgba(0,0,0,0.05);
       overflow: hidden;
   }
   
   .invoice-card .card-header {
       background: #f8f9fa;
       padding: 20px;
       border-bottom: 1px solid #e0e0e0;
   }
   ```

### Phase 10: Invoice Variables

1. Available variables
   ```
   Invoice Variables:
   - {$invoice_number}
   - {$invoice_date}
   - {$due_date}
   - {$status}
   - {$subtotal}
   - {$tax_rate}
   - {$tax_amount}
   - {$total}
   - {$balance_due}
   - {$payment_methods}
   - {$lineitems}
   
   Client Variables:
   - {$client_name}
   - {$client_email}
   - {$client_address}
   ```

## Related Workflows
- whmcs-email-template-design
- whmcs-logo-branding
- whmcs-color-scheme
- whmcs-template-modification