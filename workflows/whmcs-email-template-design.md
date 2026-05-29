# WHMCS Email Template Design Workflow

## Purpose
Guide developers through creating and customizing email templates in WHMCS.

## Prerequisites
- WHMCS installation
- HTML/CSS email development knowledge
- Email template system understanding

## Steps

### Phase 1: Email Template Structure

1. Email template locations
   ```
   /whmcs/templates/email/
   ├── header.html
   ├── footer.html
   ├── client/
   │   ├── welcome.html
   │   ├── verification.html
   │   └── password-reset.html
   └── admin/
       ├── new-order.html
       └── invoice-notification.html
   ```

2. Email template system
   ```
   WHMCS Email System:
   - Markdown-based templates (new)
   - HTML-based templates (legacy)
   - Dynamic variables
   - Conditional content
   ```

### Phase 2: Email Template Basics

1. HTML email structure
   ```html
   <!DOCTYPE html>
   <html>
   <head>
       <meta charset="UTF-8">
       <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>{$subject}</title>
   </head>
   <body style="margin: 0; padding: 0; background-color: #f4f4f4;">
       <table role="presentation" width="100%" cellpadding="0" cellspacing="0">
           <tr>
               <td align="center" style="padding: 40px 0;">
                   <table role="presentation" width="600" cellpadding="0" cellspacing="0" style="background: #ffffff; border-radius: 8px; overflow: hidden;">
                       <!-- Email content -->
                   </table>
               </td>
           </tr>
       </table>
   </body>
   </html>
   ```

2. Inline CSS requirement
   ```html
   <!-- All styles must be inline for email compatibility -->
   <div style="font-family: Arial, sans-serif; font-size: 16px; color: #333333;">
       Email content here
   </div>
   ```

### Phase 3: Email Header Design

1. Header template
   ```html
   <!-- In email header template -->
   <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="background: #1a1a2e;">
       <tr>
           <td align="center" style="padding: 30px 20px;">
               <img src="{$email_logo_url}" alt="{$companyname}" style="max-width: 200px; height: auto;">
           </td>
       </tr>
   </table>
   ```

2. Gradient header
   ```html
   <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);">
       <tr>
           <td align="center" style="padding: 40px 20px;">
               <h1 style="color: #ffffff; font-size: 24px; font-weight: 600; margin: 0;">
                   {$subject}
               </h1>
           </td>
       </tr>
   </table>
   ```

### Phase 4: Email Body Design

1. Content area
   ```html
   <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="background: #ffffff;">
       <tr>
           <td style="padding: 40px;">
               <h2 style="color: #212529; font-size: 20px; font-weight: 600; margin: 0 0 20px;">
                   Hello {$client_first_name},
               </h2>
               <p style="color: #495057; font-size: 16px; line-height: 1.6; margin: 0 0 20px;">
                   {$message}
               </p>
           </td>
       </tr>
   </table>
   ```

2. Button styling (inline CSS)
   ```html
   <table role="presentation" cellpadding="0" cellspacing="0" style="margin: 30px 0;">
       <tr>
           <td align="center">
               <a href="{$button_url}" style="display: inline-block; padding: 14px 30px; background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: #ffffff; text-decoration: none; font-size: 16px; font-weight: 600; border-radius: 6px;">
                   {$button_text}
               </a>
           </td>
       </tr>
   </table>
   ```

### Phase 5: Table-Based Data

1. Data table in email
   ```html
   <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="border: 1px solid #dee2e6; border-radius: 8px; overflow: hidden; margin: 20px 0;">
       <tr style="background: #f8f9fa;">
           <th style="padding: 12px 16px; text-align: left; font-size: 12px; font-weight: 600; color: #495057; text-transform: uppercase;">Item</th>
           <th style="padding: 12px 16px; text-align: right; font-size: 12px; font-weight: 600; color: #495057; text-transform: uppercase;">Amount</th>
       </tr>
       {foreach $items as $item}
       <tr>
           <td style="padding: 12px 16px; border-top: 1px solid #dee2e6; font-size: 14px; color: #212529;">{$item.name}</td>
           <td style="padding: 12px 16px; border-top: 1px solid #dee2e6; text-align: right; font-size: 14px; color: #212529;">{$item.price}</td>
       </tr>
       {/foreach}
   </table>
   ```

### Phase 6: Email Footer Design

1. Footer template
   ```html
   <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="background: #f8f9fa; border-top: 1px solid #dee2e6;">
       <tr>
           <td align="center" style="padding: 30px 20px;">
               <p style="color: #6c757d; font-size: 14px; margin: 0 0 10px;">
                   {$companyname}
               </p>
               <p style="color: #adb5bd; font-size: 12px; margin: 0 0 20px;">
                   {$companyaddress}
               </p>
               <p style="color: #adb5bd; font-size: 12px; margin: 0;">
                   <a href="{$WEB_ROOT}/unsubscribe.php" style="color: #667eea;">Unsubscribe</a>
                   |
                   <a href="{$WEB_ROOT}/privacy.php" style="color: #667eea;">Privacy Policy</a>
               </p>
           </td>
       </tr>
   </table>
   ```

### Phase 7: Email Variables

1. Common variables
   ```
   Client Variables:
   - {$client_first_name}
   - {$client_full_name}
   - {$client_email}
   - {$client_company_name}
   
   System Variables:
   - {$companyname}
   - {$company_logo_url}
   - {$company_address}
   - {$current_date}
   - {$WEB_ROOT}
   
   Order Variables:
   - {$order_number}
   - {$order_total}
   - {$order_date}
   ```

2. Conditional content
   ```smarty
   {if $client_company_name}
       <p>Company: {$client_company_name}</p>
   {/if}
   ```

### Phase 8: Responsive Email Design

1. Mobile responsive
   ```html
   <style>
       @media only screen and (max-width: 600px) {
           .email-container {
               width: 100% !important;
           }
           .email-content {
               padding: 20px !important;
           }
           .email-button {
               width: 100% !important;
               display: block !important;
           }
       }
   </style>
   ```

2. Fluid widths
   ```html
   <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="max-width: 600px; margin: 0 auto;">
   ```

### Phase 9: Email Testing

1. Testing checklist
   ```
   Email Testing:
   - Test in Gmail, Outlook, Apple Mail
   - Test on mobile devices
   - Check images display correctly
   - Verify links work
   - Check spam score
   - Validate HTML/CSS
   ```

2. Tools
   ```
   - Litmus (email testing)
   - Mailtrap (email preview)
   - Putsmail (test emails)
   - W3C HTML validator
   ```

## Related Workflows
- whmcs-invoice-template-design
- whmcs-logo-branding
- whmcs-color-scheme
- whmcs-template-modification