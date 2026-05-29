# WHMCS Marketing SMS Workflow

## Purpose
Implement SMS marketing campaigns in WHMCS.

## Prerequisites

### SMS Provider Setup
1. Choose SMS provider:
   - Twilio
   - Nexmo/Vonage
   - ClickSend
   - msg91

2. Configure in WHMCS:
   - Get API credentials
   - Set sender ID
   - Purchase credits

## SMS Marketing Setup

### Step 1: Enable SMS
1. Navigate to: Configuration > System > Notifications
2. Select SMS provider
3. Enter credentials
4. Test connection

### Step 2: Collect Phone Numbers
```
Sources:
- Client signup form
- Preference center
- Landing pages
- Contest entries
```

### Step 3: Compliance
- Get explicit consent
- Document opt-in
- Provide opt-out
- Respect quiet hours

## SMS Campaign Types

### Promotional
```
- Limited time offers
- Flash sales
- Exclusive deals
- New product launches
```

###Transactional
```
- Order confirmations
- Shipping updates
- Payment reminders
- Account alerts
```

### Engagement
```
- Re-engagement campaigns
- Loyalty rewards
- Referral requests
- Feedback requests
```

## Creating SMS Campaigns

### Step 1: Define Message
```
Guidelines:
- Max 160 characters (single SMS)
- Include sender name
- Clear call-to-action
- Link to landing page
```

### Step 2: Build Short URLs
Use URL shortener:
```
- Bitly
- TinyURL
- Custom short domain
```

### Step 3: Set Audience
```
Segmentation:
- Opted-in clients only
- By product ownership
- By activity level
- By location
```

## Message Templates

### Flash Sale
```
FLASH SALE! 50% off annual plans today only.
Use code: FLASH50
Claim at: example.com/flash
- Company Name
```

### Win-back
```
Hi {$first_name}, we miss you!
Use code WELCOME30 for 30% off.
Valid 7 days. Shop now: link
```

### Appointment Reminder
```
Reminder: Your service review is tomorrow at {$time}.
Reply STOP to unsubscribe.
- Company Name
```

## Timing

### Best Times
```
B2C:
- 10 AM - 2 PM (local time)
- Tuesday - Thursday (best days)

B2B:
- 8 AM - 10 AM (weekdays)
- Tuesday - Wednesday (best days)
```

### Frequency
```
- Promotional: Max 2/month
- Transactional: As needed
- Do not over-message
```

## Tracking

### Metrics
```
- Delivered: Successfully delivered
- Failed: Delivery failed
- Clicked: Links clicked
- Opted-out: Unsubscribed
```

### UTM Tracking
```
Add tracking:
example.com/promo?utm_source=sms&utm_medium=sms
```

## Compliance

### Regulations
- TCPA (USA)
- GDPR (EU)
- CASL (Canada)
- Local laws

### Requirements
```
- Explicit consent
- Easy opt-out
- Clear sender ID
- No misleading content
```

## Cost Management

### Budget Controls
- Set monthly limits
- Monitor spend
- Track ROI
- Optimize frequency

## Related Workflows
- whmcs-notification-sms
- whmcs-marketing-campaign
- whmcs-coupon-create