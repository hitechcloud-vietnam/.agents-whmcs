# WHMCS Disable Notifications Workflow

## Purpose
Turn off specific or all notifications in WHMCS.

## Disable Individual Notifications

### Step 1: Locate Notification
1. Navigate to: Configuration > System > Notifications
2. Find notification to disable

### Step 2: Disable Notification
1. Click on notification
2. Set "Status" to "Inactive"
3. Or toggle "Enabled" off
4. Save changes

### Step 3: Verify
1. Trigger event
2. Confirm no notification sent
3. Check logs

## Disable by Channel

### Email Notifications
1. Go to: Configuration > System > Notifications
2. Select "Providers" tab
3. Toggle Email to disabled

### Slack Notifications
1. Disable webhook integration
2. Or remove channel routing
3. Verify no Slack messages

### All Channels
1. Disable entire notification
2. All channels stop sending

## Disable by Event Type

### Client Events
```
Settings > Notifications > Client Events
Toggle off: Created, Updated, Deleted
```

### Order Events
```
Settings > Notifications > Order Events
Toggle off: New Order, Cancelled, etc.
```

### Invoice Events
```
Settings > Notifications > Invoice Events
Toggle off: Created, Paid, Overdue, etc.
```

## Client-Level Disable

### Allow Clients to Opt-Out
1. Navigate to: Configuration > System > Notifications
2. Enable "Client Preferences"
3. Allow opt-out for notification types

### Client Opt-Out Process
1. Client goes to Account Settings
2. Selects "Notification Preferences"
3. Unchecks unwanted notifications
4. Changes saved

## Admin Notifications

### Disable Admin Alerts
1. Go to: Configuration > System > Notifications
2. Select "Admin" tab
3. Choose which alerts to disable
4. Save

### Critical vs Non-Critical
- Keep critical: Security, Fraud, System errors
- Disable non-critical: General updates, low-priority

## Scheduled Disable

### Temporary Disable
1. Disable notification
2. Set auto-enable time
3. Or manually enable later

### Maintenance Mode
1. Disable all notifications
2. Perform maintenance
3. Re-enable all
4. Verify working

## Disable Notification Sounds

### Email Sound
1. In WHMCS settings
2. Disable sound notifications

### Browser Notifications
1. Disable in browser
2. Or whitelist specific pages

## Best Practices

### Selective Disable
- Disable only what's not needed
- Don't blanket disable all
- Document why disabled
- Review periodically

### Communication
- Notify users of changes
- Provide alternative channels
- Maintain critical alerts

## Re-Enabling

### Step 1: Find Disabled
1. Navigate to: Configuration > System > Notifications
2. Filter by "Inactive" status

### Step 2: Enable
1. Click on notification
2. Set status to "Active"
3. Save
4. Test

## Related Workflows
- whmcs-notification-create
- whmcs-notification-preferences
- whmcs-notification-log