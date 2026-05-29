# WHMCS Notification Templates Workflow

## Purpose
Create and manage templates for notification messages.

## Template Structure

### Template Components
1. **Title/Subject**: Notification title
2. **Body**: Main content
3. **Variables**: Dynamic placeholders
4. **Actions**: Call-to-action buttons

## Access Template Editor

### Step 1: Navigate to Templates
1. Go to: Configuration > System > Notifications
2. Select "Templates" tab
3. Choose notification type

### Step 2: Edit Template
1. Click on template name
2. Edit content
3. Save changes

## Template Variables

### Common Variables
```
{$client_name}
{$client_email}
{$event_type}
{$event_timestamp}
{$alert_title}
{$alert_message}
{$priority}
{$source_url}
```

### Event-Specific Variables
```
Invoice:
{$invoice_number}
{$invoice_amount}
{$invoice_due_date}

Order:
{$order_number}
{$order_products}
{$order_total}

Support:
{$ticket_number}
{$ticket_subject}
{$ticket_status}
```

## Template Examples

### Simple Alert
```
Title: New Order Received
Body:
Order #{$order_number} has been placed.

Customer: {$client_name}
Products: {$order_products}
Total: {$order_total}

View Order: {$source_url}
```

### Critical Alert
```
Title: {$priority}: {$alert_title}
Body:
{$alert_message}

Client: {$client_name}
Time: {$event_timestamp}

Action Required: {$action_url}
```

### Daily Digest
```
Title: Daily Summary - {$date}
Body:
Total Orders: {$orders_today}
Total Revenue: {$revenue_today}
New Clients: {$new_clients}

Top Products:
{$top_products}

View Full Report: {$report_url}
```

## Formatting

### Markdown Support
```
**Bold text**
*Italic text*
- Bullet points
1. Numbered lists
[Link Text](url)
```

### Rich Content
```
Buttons:
[Button Text](url:primary)
[Cancel](url:secondary)

Images:
![Alt Text](image_url)
```

## Testing Templates
1. Click "Preview"
2. Select sample data
3. Review rendered output
4. Test on different channels

## Best Practices
- Keep titles short (50 chars)
- Use clear call-to-action
- Include relevant data only
- Personalize when possible

## Related Workflows
- whmcs-notification-create
- whmcs-notification-template-vars