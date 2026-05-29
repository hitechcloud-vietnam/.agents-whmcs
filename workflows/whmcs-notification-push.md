# WHMCS Push Notifications Workflow

## Purpose
Configure and manage push notifications for mobile/web.

## Push Notification Providers

### 1. OneSignal
1. Create OneSignal account
2. Create app in dashboard
3. Get App ID and API key
4. Configure platforms (iOS, Android, Web)

### 2. Firebase Cloud Messaging (FCM)
1. Create Firebase project
2. Get server key
3. Configure Android/iOS apps

### 3. Pusher
1. Create Pusher account
2. Get app credentials
3. Set up channels

## Configuration

### Step 1: Choose Provider
1. Navigate to: Configuration > System > Notifications
2. Select "Providers" tab
3. Enable push notification provider

### Step 2: Enter Credentials
```
OneSignal:
- App ID: xxxxxxxx
- API Key: xxxxxxxx

FCM:
- Server Key: xxxxxxxx
- Sender ID: xxxxxxxx
```

### Step 3: Configure App Settings
1. Set notification icons
2. Configure click actions
3. Set sound/vibration
4. Set delivery timing

## Push Notification Setup

### Step 1: Enable Client Push
1. Navigate to: Configuration > System > Notifications
2. Enable "Client Push"
3. Configure what triggers push

### Step 2: Set Up Client Subscription
1. Enable in client profile
2. Configure push topics
3. Set frequency preferences

### Step 3: Create Push Template
```
Title: {$alert_title}
Body: {$alert_message}
Icon: company_icon.png
Click Action: {$action_url}
```

## Push Notification Variables

### Content Variables
```
{$client_name}
{$alert_title}
{$alert_message}
{$event_type}
{$action_url}
{$badge_count}
```

### Advanced Variables
```
{$image_url}
{$buttons}
{$sound}
```

## Push Best Practices

### Message Design
1. Keep title under 50 characters
2. Body under 100 characters
3. Clear call-to-action
4. Include image when relevant

### Example Notifications
```
Title: New Invoice
Body: Invoice #12345 for $99.99 is due

Title: Service Activated
Body: Your VPS is ready! Click to access

Title: Support Reply
Body: New reply on ticket #789 - View now
```

## Managing Subscriptions

### Client Preferences
1. Allow clients to opt-in/opt-out
2. Configure notification categories
3. Set quiet hours

### Admin Controls
1. Push to all clients
2. Push to specific groups
3. Schedule notifications

## Troubleshooting

### Not Receiving
- Check subscription status
- Verify device token
- Check notification permission

### Delivery Issues
- Review provider dashboard
- Check API errors
- Verify credentials

## Related Workflows
- whmcs-notification-create
- whmcs-notification-providers
- whmcs-notification-preferences