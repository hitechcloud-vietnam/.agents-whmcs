# WHMCS Notification Preferences Workflow

## Purpose
Allow clients to control their notification preferences.

## Enable Client Preferences

### Step 1: Configure in WHMCS
1. Navigate to: Configuration > System > Notifications
2. Select "Client Preferences" tab
3. Enable client opt-in feature

### Step 2: Set Available Options
```
Available Preferences:
- Marketing emails: On/Off
- Product updates: On/Off
- Invoice reminders: On/Off (always sent)
- Security alerts: On/Off (always sent)
```

### Step 3: Configure Default State
```
Default Settings:
- New clients: All enabled (except marketing)
- Existing clients: Maintain current settings
```

## Client Preference Page

### Access
1. Client logs in
2. Goes to Account Settings
3. Finds "Notification Preferences"

### Options
```
Email Notifications:
[ ] Marketing emails
[x] Product announcements
[x] Invoice notifications

SMS Notifications:
[ ] Order updates
[x] Payment reminders

Push Notifications:
[x] New messages
[ ] Promotions
```

## Notification Categories

### Marketing
- Promotional emails
- Special offers
- Newsletter

### Transactional
- Order confirmations
- Invoice notifications
- Shipping updates

### Service
- Product updates
- Maintenance notices
- Security alerts

### Support
- Ticket updates
- Response notifications
- Satisfaction surveys

## Per-Product Preferences

### Option 1: All Products
Client sets global preferences

### Option 2: Per Product
```
Product A:
[ ] Email updates
[ ] SMS reminders

Product B:
[x] Email updates
[x] SMS reminders
```

## Implementation

### Database Fields
```
tblclients.notification_preferences
- marketing_email: enum('on','off')
- product_updates: enum('on','off')
- security_alerts: enum('on','off')
```

### Hook Implementation
```php
add_hook('NotificationSend', 1, function($vars) {
    $clientId = $vars['client_id'];
    $notificationType = $vars['type'];

    if (!checkClientPreference($clientId, $notificationType)) {
        return false; // Don't send
    }
});
```

## Privacy Compliance

### GDPR
- Explicit consent required for marketing
- Easy opt-out
- Data retention policies

### CAN-SPAM
- Clear unsubscribe mechanism
- Physical address required
- Honest subject lines

## Monitoring Preferences

### Analytics
- Track opt-out rates
- Monitor preference changes
- Analyze by segment

### Reporting
```
Total Clients: 1000
- Marketing Enabled: 600 (60%)
- Product Updates: 900 (90%)
- SMS Enabled: 200 (20%)
```

## Related Workflows
- whmcs-notification-create
- whmcs-email-unsubscribe
- whmcs-notification-groups