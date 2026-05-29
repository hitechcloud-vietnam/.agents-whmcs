# WHMCS Marketing Push Workflow

## Purpose
Implement push notification marketing in WHMCS.

## Push Notification Providers

### Available Options
```
- OneSignal
- Firebase Cloud Messaging
- PushWoosh
- WebEngage
```

## Setup

### Step 1: Choose Provider
1. Create account with provider
2. Create app (iOS, Android, Web)
3. Get credentials:
   - App ID
   - API Key
   - Server Key

### Step 2: Configure in WHMCS
1. Navigate to: Configuration > System > Notifications
2. Select push provider
3. Enter credentials
4. Set app configuration

### Step 3: Enable Client Push
1. Allow client opt-in
2. Set default preferences
3. Configure notification types

## Push Marketing Types

### Promotional
```
- Special offers
- New products
- Sales events
- Exclusive deals
```

### Informational
```
- Account updates
- Service changes
- New features
- Industry news
```

### Re-engagement
```
- Inactivity alerts
- Cart reminders
- Win-back campaigns
- Feedback requests
```

## Creating Push Campaigns

### Step 1: Design Notification
```
Format:
Title: Max 50 characters
Body: Max 100 characters
Icon: Brand icon
Image: Optional hero image
Action: Click behavior
```

### Step 2: Set Targeting
```
Options:
- All subscribers
- Segment by:
  - Device type
  - Location
  - Interests
  - Behavior
```

### Step 3: Schedule
```
Timing:
- Immediate
- Scheduled
- Event-triggered
```

## Push Content Examples

### Flash Sale
```
Title: Flash Sale!
Body: 50% off today only. Tap to shop.
Image: sale-banner.jpg
Action: Open sale page
```

### New Feature
```
Title: New Feature Available
Body: Check out our new dashboard. Learn more.
Image: feature-preview.jpg
Action: Open what's new page
```

### Abandoned Cart
```
Title: You left something behind
Body: Complete your order in 24 hours.
Action: Open cart
```

## Personalization

### Variables
```
{$client_first_name}
{$product_name}
{$discount_code}
{$link}
```

### Conditional Content
```
{if $client_group == "VIP"}
   Exclusive offer for you!
{/if}
```

## Optimization

### Timing
```
Best times:
- 10 AM - 12 PM (morning)
- 6 PM - 8 PM (evening)

Avoid:
- Late night
- Early morning
- Meal times
```

### Frequency
```
Recommended:
- 1-2 per week
- Max 5 per week
- Exclude quiet hours
```

## Metrics

### Tracking
```
- Delivered: Successfully sent
- Received: Device received
- Clicked: Notification opened
- Converted: Action taken
```

### Benchmarks
```
- Delivery rate: 95%+
- Click rate: 5-10%
- Conversion: 1-3%
```

## Compliance

### Requirements
```
- Clear opt-in
- Easy opt-out
- Relevant content
- No deceptive practices
```

## Related Workflows
- whmcs-notification-push
- whmcs-marketing-campaign
- whmcs-marketing-email