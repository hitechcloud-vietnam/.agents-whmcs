# WHMCS Park Page Documentation

## Overview

Park pages (also known as "coming soon" or "placeholder" pages) display when a domain is registered but not yet pointed to a hosting service. This documentation covers setup, configuration, and management of park pages in WHMCS.

## Park Page Types

### Default System Park Page

- Simple branded placeholder
- Displays domain registration confirmation
- Includes basic navigation
- Shows "Coming Soon" message

### Custom Park Page

- Fully branded appearance
- Custom messaging and imagery
- Email collection capability
- Social media integration

### For Sale Page

- Domain offered for sale
- Price display
- Buy now functionality
- Contact form for offers

### Under Construction

- Construction/maintenance status
- Progress indicator
- Estimated launch date
- Contact information

## Configuration

### Enable Park Pages

Navigate to: **Configuration > Domain Settings > Parked Domains**

```php
// Park Page Configuration
$parkConfig = [
    'enabled' => true,
    'default_type' => 'custom',
    'allow_customer_selection' => true,
    'park_new_registrations' => true,
    'park_transfer_in' => false,
    'auto_expire_parking' => 365,      // days
    'redirect_on_hosting' => true
];
```

### Park Page Templates

```php
// Template Configuration
$templateConfig = [
    'system_default' => [
        'template' => 'default-park',
        'customizable' => false
    ],
    'custom_brand' => [
        'template' => 'custom-park',
        'logo_url' => 'https://cdn.example.com/logo.png',
        'primary_color' => '#2563eb',
        'secondary_color' => '#64748b'
    ],
    'for_sale' => [
        'template' => 'for-sale',
        'price' => '$500',
        'accept_offers' => true,
        'buy_now_url' => 'https://store.example.com/domain/example.com'
    ]
];
```

## Template Structure

### Directory Structure

```
/whmcs/templates/park/
├── default-park/
│   ├── template.html
│   ├── styles.css
│   ├── script.js
│   └── images/
│       └── logo.png
├── custom-park/
│   ├── template.html
│   ├── styles.css
│   └── script.js
└── for-sale/
    ├── template.html
    ├── styles.css
    └── script.js
```

### Template Variables

| Variable | Description | Example |
|----------|-------------|---------|
| {$domain} | Domain name | example.com |
| {$registrar} | Registrar name | Enom |
| {$registration_date} | Registration date | 2024-01-15 |
| {$expiry_date} | Expiry date | 2025-01-15 |
| {$owner_name} | Owner name | John Doe |
| {$owner_email} | Owner email | john@example.com |
| {$company_name} | Hosting company | Acme Hosting |
| {$company_logo} | Company logo URL | https://cdn.example.com/logo.png |
| {$company_email} | Contact email | support@example.com |
| {$company_phone} | Phone number | +1-555-123-4567 |
| {$park_page_title} | Custom title | Coming Soon |
| {$park_page_message} | Custom message | Site under construction |

### Default Park Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{$domain} - Coming Soon</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            color: #fff;
        }
        .container {
            text-align: center;
            max-width: 600px;
            padding: 2rem;
        }
        .logo {
            max-width: 200px;
            margin-bottom: 2rem;
        }
        h1 {
            font-size: 2.5rem;
            margin-bottom: 1rem;
        }
        .domain-name {
            font-size: 1.5rem;
            opacity: 0.9;
            margin-bottom: 2rem;
        }
        .message {
            font-size: 1.125rem;
            line-height: 1.6;
            opacity: 0.85;
            margin-bottom: 2rem;
        }
        .contact-info {
            font-size: 0.875rem;
            opacity: 0.7;
        }
        .contact-info a {
            color: #fff;
            text-decoration: underline;
        }
    </style>
</head>
<body>
    <div class="container">
        {if $company_logo}
        <img src="{$company_logo}" alt="{$company_name}" class="logo">
        {/if}
        <h1>{$park_page_title|default:'Coming Soon'}</h1>
        <p class="domain-name">{$domain}</p>
        <p class="message">
            {$park_page_message|default:'This domain is registered but the website is not yet live.
            Please check back soon or contact us for more information.'}
        </p>
        <div class="contact-info">
            <p>Contact: <a href="mailto:{$company_email}">{$company_email}</a></p>
            {if $company_phone}
            <p>Phone: {$company_phone}</p>
            {/if}
        </div>
    </div>
</body>
</html>
```

## Custom Park Pages

### Create Custom Template

**Admin > Configuration > Domain Settings > Park Pages > Create Template**

```
+------------------------------------------+
| Create Park Page Template                 |
+------------------------------------------+
| Template Name: [Custom Landing____]     |
| Template Type: [Standard__________]     |
|                                          |
| Branding:                                |
| Logo URL:  [https://cdn.example.com/...] |
| Primary Color: [#2563eb_________]       |
| Background: [Gradient______________]     |
|                                          |
| Content:                                 |
| Title: [Coming Soon______________]       |
| Message:                                 |
| [This domain is registered but the...]   |
|                                          |
| Features:                                |
| [x] Show contact information            |
| [x] Show registration date               |
| [ ] Enable email capture                |
| [ ] Show social links                   |
|                                          |
| [Save Template] [Preview]                |
+------------------------------------------+
```

### Email Capture Form

```html
<div class="email-capture">
    <h3>Get Notified When We Launch</h3>
    <form action="/api/park/subscribe" method="POST">
        <input type="email" name="email" placeholder="Enter your email" required>
        <input type="hidden" name="domain" value="{$domain}">
        <button type="submit">Notify Me</button>
    </form>
    <p class="privacy-note">We respect your privacy. Unsubscribe anytime.</p>
</div>

<style>
.email-capture {
    background: rgba(255,255,255,0.1);
    padding: 2rem;
    border-radius: 8px;
    margin: 2rem 0;
}
.email-capture form {
    display: flex;
    gap: 0.5rem;
    justify-content: center;
    flex-wrap: wrap;
}
.email-capture input[type="email"] {
    padding: 0.75rem 1rem;
    border: none;
    border-radius: 4px;
    width: 250px;
}
.email-capture button {
    padding: 0.75rem 1.5rem;
    background: #fff;
    color: #2563eb;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-weight: 600;
}
</style>
```

### Social Media Links

```html
<div class="social-links">
    {if $facebook_url}
    <a href="{$facebook_url}" target="_blank" rel="noopener">
        <img src="/templates/park/icons/facebook.svg" alt="Facebook">
    </a>
    {/if}
    {if $twitter_url}
    <a href="{$twitter_url}" target="_blank" rel="noopener">
        <img src="/templates/park/icons/twitter.svg" alt="Twitter">
    </a>
    {/if}
    {if $instagram_url}
    <a href="{$instagram_url}" target="_blank" rel="noopener">
        <img src="/templates/park/icons/instagram.svg" alt="Instagram">
    </a>
    {/if}
    {if $linkedin_url}
    <a href="{$linkedin_url}" target="_blank" rel="noopener">
        <img src="/templates/park/icons/linkedin.svg" alt="LinkedIn">
    </a>
    {/if}
</div>
```

## For Sale Pages

### For Sale Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{$domain} - For Sale</title>
    <style>
        body {
            font-family: system-ui, sans-serif;
            background: linear-gradient(135deg, #10b981 0%, #059669 100%);
            min-height: 100vh;
            margin: 0;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        .sale-card {
            background: white;
            padding: 3rem;
            border-radius: 16px;
            box-shadow: 0 25px 50px rgba(0,0,0,0.2);
            text-align: center;
            max-width: 500px;
        }
        .domain-badge {
            background: #ecfdf5;
            color: #059669;
            padding: 0.5rem 1rem;
            border-radius: 100px;
            font-weight: 600;
            display: inline-block;
            margin-bottom: 1rem;
        }
        .domain-name {
            font-size: 2rem;
            font-weight: 700;
            margin-bottom: 1rem;
        }
        .price {
            font-size: 3rem;
            color: #059669;
            font-weight: 800;
            margin: 1.5rem 0;
        }
        .features {
            text-align: left;
            margin: 1.5rem 0;
        }
        .features li {
            padding: 0.5rem 0;
            list-style: none;
        }
        .features li::before {
            content: "✓ ";
            color: #059669;
            margin-right: 0.5rem;
        }
        .buy-button {
            display: inline-block;
            background: #059669;
            color: white;
            padding: 1rem 2rem;
            border-radius: 8px;
            text-decoration: none;
            font-weight: 600;
            font-size: 1.125rem;
            margin-top: 1rem;
        }
        .contact-link {
            margin-top: 1rem;
            color: #6b7280;
        }
    </style>
</head>
<body>
    <div class="sale-card">
        <span class="domain-badge">Domain For Sale</span>
        <h1 class="domain-name">{$domain}</h1>
        <div class="price">{$sale_price|default:'$500'}</div>
        <ul class="features">
            <li>Premium .com domain</li>
            <li>SEO-friendly name</li>
            <li>Brandable and memorable</li>
            <li>Registered until {$expiry_date}</li>
        </ul>
        {if $buy_now_url}
        <a href="{$buy_now_url}" class="buy-button">Buy Now</a>
        {/if}
        <p class="contact-link">
            Or make an offer: <a href="mailto:{$company_email}">{$company_email}</a>
        </p>
    </div>
</body>
</html>
```

### Offer Submission

```php
// Handle offer submission
$offer = WHMCS\Domains\ParkPage\Offer::submit([
    'domain' => $domain,
    'offer_amount' => $_POST['offer_amount'],
    'buyer_email' => $_POST['buyer_email'],
    'buyer_message' => $_POST['buyer_message'] ?? null
]);

if ($offer['success']) {
    // Send notification to domain owner
    WHMCS\Mail\Templates::send('DomainOfferNotification', [
        'domain' => $domain,
        'offer' => $offer
    ]);
}
```

## Per-Domain Parking

### Domain-Level Settings

```php
// Park specific domain
$domain->parking = [
    'enabled' => true,
    'template' => 'custom-landing',
    'title' => 'Premium Domain For Sale',
    'message' => 'This premium domain is available for purchase.',
    'price' => '$1,500',
    'buy_now_url' => 'https://store.example.com/domain/example.com',
    'accept_offers' => true,
    'contact_email' => 'sales@example.com'
];
$domain->save();
```

### Customer Selection

**Client Area > My Domains > Domain Settings > Parking**

Customers can select from available templates:

```
+------------------------------------------+
| Park Page Options                        |
+------------------------------------------+
| Current Status: Active                   |
|                                          |
| Select Template:                        |
| ( ) Default Coming Soon                  |
| (o) Custom Branded                       |
| ( ) For Sale                             |
| ( ) Under Construction                   |
|                                          |
| Template Preview:                        |
| +----------------------------------+    |
| |     [Logo]                       |    |
| |                                  |    |
| |     Coming Soon                  |    |
| |                                  |    |
| |     example.com                  |    |
| +----------------------------------+    |
|                                          |
| [Save Changes]                           |
+------------------------------------------+
```

## Analytics Integration

### Track Park Page Views

```php
// Track view
WHMCS\Domains\ParkPage\Analytics::trackView($domain, [
    'source' => $_SERVER['HTTP_REFERER'] ?? 'direct',
    'user_agent' => $_SERVER['HTTP_USER_AGENT'],
    'ip_hash' => hash('sha256', $_SERVER['REMOTE_ADDR'])
]);
```

### View Statistics

```http
GET /domains/{domain}/parking/stats
```

**Response:**

```json
{
  "domain": "example.com",
  "period": "last_30_days",
  "stats": {
    "total_views": 5420,
    "unique_visitors": 3201,
    "email_signups": 45,
    "offers_received": 3,
    "buy_now_clicks": 128
  },
  "top_referrers": [
    "google.com",
    "facebook.com",
    "twitter.com"
  ],
  "views_by_day": [
    {"date": "2024-01-01", "views": 180},
    {"date": "2024-01-02", "views": 195}
  ]
}
```

## API Integration

### Update Park Page

```http
PUT /domains/{domain}/parking
```

**Request Body:**

```json
{
  "enabled": true,
  "template": "custom-park",
  "content": {
    "title": "Coming Soon",
    "message": "Our new website is launching soon.",
    "contact_email": "info@example.com"
  },
  "features": {
    "email_capture": true,
    "social_links": true
  }
}
```

### Get Park Page Data

```http
GET /domains/{domain}/parking
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Park page not showing | Domain pointed to hosting | Unlink hosting first |
| Custom CSS not loading | Cache issue | Clear cache |
| Email capture broken | API endpoint down | Check API logs |
| For sale button broken | Missing buy_now_url | Configure URL |
| Template 404 | Template deleted | Restore or switch template |

### Debug Mode

```bash
# Check park status
whmcscli domain parking --domain=example.com

# Preview park page
whmcscli domain park-preview --domain=example.com --template=custom

# View park logs
whmcscli domain park-logs --domain=example.com
```

## See Also

- [Domain Forwarding Service](./whmcs-forwarding-service.md)
- [Domain Pricing Configuration](../products-services/domain-pricing.md)
- [Template Development Guide](../developer/templates.md)
