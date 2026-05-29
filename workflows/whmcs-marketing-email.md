# WHMCS Marketing Email Workflow

## Purpose
Create and manage marketing emails for client engagement.

## Email Marketing Setup

### Step 1: Choose Platform
Option A: WHMCS Native
- Built-in email templates
- Basic segmentation
- Limited analytics

Option B: External Integration
- Mailchimp
- SendGrid
- ConvertKit
- Advanced features

### Step 2: Configure Integration
If using external:
1. Install/activate module
2. Configure API credentials
3. Sync client list
4. Import custom fields

## Creating Marketing Emails

### Step 1: Design Email
1. Use template builder
2. Add images/logos
3. Write compelling copy
4. Add clear CTA

### Step 2: Personalization
```
Variables:
- {$client_first_name}
- {$client_company}
- {$product_name}
- {$renewal_date}

Dynamic content:
{if $client_group == "VIP"}Special offer for you!{/if}
```

### Step 3: Set Links
```
Required links:
- View in browser
- Unsubscribe
- Privacy policy

Optional links:
- Product page
- Account dashboard
- Support center
```

## Email Types

### Newsletter
- Regular updates
- Industry news
- Company announcements
- Feature highlights

### Promotional
- Special offers
- Discount codes
- Limited time deals
- Product launches

### Transactional
- Order confirmations
- Receipts
- Shipping updates
- Account updates

## Segmentation

### By Behavior
```
- Active users (last 30 days)
- Inactive users (no login 60+ days)
- Recent purchasers
- High spenders
```

### By Preferences
```
- Product type preferences
- Communication preferences
- Content interests
- Engagement level
```

### By Lifecycle
```
- New customers
- Existing customers
- At-risk customers
- Churned customers
```

## Sending Best Practices

### Subject Lines
- Under 50 characters
- Personal when possible
- Create urgency
- Avoid spam words

### Content
```
Header: Clear branding
Hero: Main image/offer
Body: 2-3 short paragraphs
CTA: One primary button
Footer: Contact info, unsubscribe
```

### Preview
- Test in multiple clients
- Check mobile display
- Verify images
- Test links

## Metrics

### Tracking
```
- Open rate: Opened / Delivered
- Click rate: Clicks / Delivered
- Conversion rate: Conversions / Sent
- Bounce rate: Bounced / Sent
- Unsubscribe rate: Unsubscribed / Sent
```

### Benchmarks
```
- Open rate: 20-30%
- Click rate: 2-5%
- Conversion: 1-3%
- Bounce: <2%
- Unsubscribe: <0.5%
```

## Automation

### Drip Sequences
```
1. Welcome (Day 0)
2. Product tour (Day 3)
3. Case study (Day 7)
4. Special offer (Day 14)
5. Last chance (Day 21)
```

### Triggered Emails
```
- Cart abandonment
- Browse abandonment
- Purchase follow-up
- Review request
```

## Related Workflows
- whmcs-marketing-campaign
- whmcs-email-template-create
- whmcs-email-tracking