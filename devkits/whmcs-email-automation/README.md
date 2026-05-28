# WHMCS Email Automation Module

Automated email sequences triggered by events with scheduling, personalization, and tracking.

## Features

- Email sequences with multiple steps
- Event-triggered enrollment
- Delay between emails
- Personalization tokens
- Delivery tracking (opens, clicks)
- Unsubscription handling
- Statistics

## Installation

Copy module to `/path/to/whmcs/modules/servers/emailautomation/` and activate.

## Usage

```php
// Create email sequence
emailautomation_CreateSequence(array(
    'name' => 'New Client Welcome',
    'trigger_event' => 'client.created',
    'emails' => array(
        array('subject' => 'Welcome!', 'body' => 'Hello {first_name}!', 'delay_hours' => 0),
        array('subject' => 'Getting Started', 'body' => 'Let us show you...', 'delay_hours' => 24),
        array('subject' => 'Tips & Tricks', 'body' => 'Here are tips...', 'delay_hours' => 72),
    ),
));

// Trigger on event
emailautomation_TriggerEvent('client.created', $userId);

// Process pending emails (run via cron)
$processed = emailautomation_ProcessSequences();

// Get enrollments
$enrollments = emailautomation_GetEnrollments($userId);

// Unsubscribe
emailautomation_Unenroll($sequenceKey, $userId);
```

## Personalization Tokens

- `{first_name}` - Client first name
- `{last_name}` - Client last name
- `{email}` - Client email

## API Functions

| Function | Description |
|----------|-------------|
| `emailautomation_CreateSequence()` | Create email sequence |
| `emailautomation_GetSequence()` | Get sequence |
| `emailautomation_UpdateSequence()` | Update sequence |
| `emailautomation_EnrollUser()` | Enroll client |
| `emailautomation_TriggerEvent()` | Trigger enrollment |
| `emailautomation_ProcessSequences()` | Process pending emails |
| `emailautomation_GetEnrollments()` | Get user enrollments |
| `emailautomation_Unenroll()` | Unsubscribe user |
