# WHMCS Email Open/Click Tracking Workflow

## Purpose
Track email opens and link clicks for campaign analytics.

## Tracking Methods

### Method 1: Built-in Tracking
1. Navigate to: Configuration > System > Settings > Mail
2. Enable "Track Email Opens"
3. Enable "Track Email Clicks"

### Method 2: External Service Integration
Use third-party services:
- Mailchimp
- SendGrid
- Postmark
- Sendinblue

## Open Tracking

### How It Works
1. Embed 1x1 transparent pixel image
2. Unique per recipient
3. Logs when image loaded
4. Requires images enabled in client

### Data Captured
- Open timestamp
- Client IP
- Email client
- Device type
- Location (if available)

### Limitations
- Not 100% accurate
- Blocked by image blocking
- Privacy concerns

## Click Tracking

### How It Works
1. Replace all links with tracking URLs
2. Log click event
3. Redirect to actual destination
4. Include UTM parameters if needed

### Data Captured
- Click timestamp
- URL clicked
- Client IP
- Email client
- Device type

### Configuration
```
Click Tracking Settings:
- Rewrite all links: Yes
- Add UTM parameters: Yes/No
- UTM Source: whmcs
- UTM Medium: email
- UTM Campaign: (template name)
```

## Analytics Dashboard

### Access Reports
1. Navigate to: Utilities > Logs > Email Statistics
2. Select date range
3. Filter by template

### Metrics Available
```
- Total Sent
- Delivered
- Opened (unique/total)
- Clicked (unique/total)
- Bounced
- Unsubscribed
- Conversion Rate
```

## Integration with Analytics

### Google Analytics Integration
1. Add UTM parameters to links
2. Create Goals for conversions
3. Import data to GA

### UTM Parameters
```
utm_source=whmcs
utm_medium=email
utm_campaign=template_name
utm_content=link_name
```

## Privacy Considerations
- Get consent for tracking
- Provide option to opt-out
- Comply with GDPR/CCPA
- Clear disclosure in privacy policy

## Related Workflows
- whmcs-email-tracking
- whmcs-report-email
- whmcs-marketing-email