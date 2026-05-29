# WHMCS Notification Test Workflow

## Purpose
Test notification configurations to ensure proper delivery and formatting.

## Testing Methods

### Method 1: Built-in Test
1. Navigate to: Configuration > System > Notifications
2. Select notification
3. Click "Send Test"
4. Choose channel
5. Enter test recipient
6. Send

### Method 2: Preview
1. Edit notification
2. Click "Preview"
3. Select test data
4. View formatted output

### Method 3: Trigger Test
1. Perform action that triggers notification
2. Check delivery
3. Verify content

## Test Checklist

### Channel Testing
- [ ] Email sends correctly
- [ ] Slack message formatted
- [ ] Discord embed displays
- [ ] Telegram bot works
- [ ] Push notification received

### Content Testing
- [ ] Variables replace correctly
- [ ] Links work
- [ ] Images display
- [ ] Formatting preserved
- [ ] Mobile display correct

### Timing Testing
- [ ] Immediate send works
- [ ] Delayed send correct
- [ ] Scheduled send on time
- [ ] Digest collects properly

## Test Scenarios

### Scenario 1: New Order Notification
1. Create test order
2. Trigger new order notification
3. Verify Slack message
4. Check embed format
5. Test CTA button

### Scenario 2: Invoice Overdue
1. Set test invoice to overdue
2. Trigger notification
3. Verify all channels
4. Check priority routing

### Scenario 3: High-Value Alert
1. Create order > $1000
2. Trigger notification
3. Verify manager receives
4. Check escalation

## Test Data Preparation

### Create Test Client
1. Navigate to: Clients > Add New Client
2. Create test data
3. Note client ID

### Create Test Order
1. Use test client
2. Add products
3. Process payment
4. Use for testing

## Automated Testing

### Cron Test
1. Ensure cron running
2. Check automation log
3. Verify scheduled notifications
4. Monitor delivery

### Integration Test
1. Test all webhook endpoints
2. Verify API calls
3. Check third-party integration

## Test Results Documentation

### Log Test Results
```
Date: 2024-01-15
Notification: New Order
Channels: Email, Slack, Discord
Results:
  - Email: PASS
  - Slack: PASS
  - Discord: PASS
Notes: All working correctly
```

## Common Issues

### Variables Not Replacing
- Check syntax {$$var_name}
- Verify variable exists
- Test with debug mode

### Channel Not Working
- Verify provider config
- Test connection
- Check credentials

### Formatting Broken
- Check markdown/HTML
- Test in different clients
- Verify escape characters

## Related Workflows
- whmcs-notification-create
- whmcs-notification-providers
- whmcs-notification-log