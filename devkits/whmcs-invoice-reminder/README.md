# WHMCS Invoice Reminder Module

Automated invoice reminder scheduler with customizable templates.

## Features

- Custom reminder templates
- Due date reminders
- Overdue notices
- Final notice support
- Delivery tracking
- Statistics

## Installation

Copy module to `/path/to/whmcs/modules/servers/invoicereminder/` and activate.

## Usage

```php
// Create reminder template
invoicereminder_CreateTemplate(array(
    'name' => '7 Day Reminder',
    'reminder_type' => 'reminder',
    'days_offset' => -7,
    'subject' => 'Invoice #{invoice_num} Due Soon',
    'body' => 'Hello {first_name}, your invoice is due on {due_date}'
));

// Process reminders (run via cron)
$processed = invoicereminder_ProcessReminders();

// Get sent reminders
$reminders = invoicereminder_GetSentReminders($invoiceId);

// Get statistics
$stats = invoicereminder_GetStatistics(30);
```

## API Functions

| Function | Description |
|----------|-------------|
| `invoicereminder_CreateTemplate()` | Create reminder template |
| `invoicereminder_ProcessReminders()` | Process pending reminders |
| `invoicereminder_GetSentReminders()` | Get reminders for invoice |
| `invoicereminder_GetStatistics()` | Get reminder statistics |
