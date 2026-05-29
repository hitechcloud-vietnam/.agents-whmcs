# WHMCS Notification Create Workflow

## Purpose
Create custom notifications for system events in WHMCS.

## Prerequisites
- WHMCS admin access
- Basic understanding of notification events

## Steps

### Step 1: Access Notifications
1. Log into WHMCS admin area
2. Navigate to: Configuration > System > Notifications
3. Click "Create Notification"

### Step 2: Configure Basic Settings
1. **Name**: Descriptive notification name
2. **Event Type**: Select trigger event
3. **Description**: Brief description
4. **Status**: Active/Inactive

### Step 3: Select Event Triggers
```
Available Events:
- Client Created
- Invoice Created
- Invoice Paid
- Order Created
- Service Created
- Domain Created
- Ticket Created
- Ticket Replied
- Password Changed
- Module Hook
```

### Step 4: Configure Conditions (Optional)
1. Enable conditions
2. Add rules:
   - Client group equals "VIP"
   - Product type contains "VPS"
   - Invoice amount greater than "100"
3. Combine with AND/OR logic

### Step 5: Set Channels
Select notification channels:
- Email
- Admin Notification Center
- Slack
- Discord
- Webhook
- SMS (if configured)

### Step 6: Configure Template
1. Select existing or create new
2. Set subject line
3. Write message body
4. Use notification variables

### Step 7: Set Priority & Timing
1. Set priority level (Low/Medium/High/Critical)
2. Configure timing:
   - Immediate
   - Delayed (X hours after event)
   - Scheduled (specific time)

### Step 8: Test & Activate
1. Click "Save Changes"
2. Use "Send Test Notification"
3. Verify delivery
4. Activate if test successful

## Notification Variables
```
{$client_name}
{$client_email}
{$event_type}
{$event_date}
{$event_data}
{$alert_message}
{$priority}
```

## Related Workflows
- whmcs-notification-providers
- whmcs-notification-triggers
- whmcs-notification-templates