# WHMCS White Label Configuration Workflow

## Purpose

Comprehensive guide to white labeling WHMCS for resellers and branded deployments, including custom branding, domain configuration, and removal of WHMCS references.

## Prerequisites

- WHMCS professional or unlimited license
- Access to branding assets (logos, colors, fonts)
- Custom domain or subdomain
- Template development access

## Workflow Steps

### Step 1: Brand Identity Definition

Define complete brand identity:

```php
// Configuration for white label branding

return [
    'brand' => [
        'name' => 'Your Hosting Company',
        'tagline' => 'Reliable Cloud Solutions',
        'website' => 'https://yourcompany.com',
        
        // Contact information
        'support_email' => 'support@yourcompany.com',
        'support_phone' => '+1-800-555-0123',
        'support_hours' => '24/7 Support',
        
        // Legal
        'company_name' => 'Your Company Inc.',
        'company_address' => '123 Business St, City, State 12345',
        'tax_id' => 'XX-XXXXXXX',
        
        // Social media
        'social' => [
            'facebook' => 'https://facebook.com/yourcompany',
            'twitter' => 'https://twitter.com/yourcompany',
            'linkedin' => 'https://linkedin.com/company/yourcompany',
        ],
    ],
    
    'colors' => [
        'primary' => '#2563EB',      // Main brand color
        'secondary' => '#1E40AF',    // Darker shade
        'accent' => '#06B6D4',       // Highlights
        'background' => '#F8FAFC',   // Page background
        'text' => '#1E293B',         // Body text
        'text_light' => '#64748B',  // Secondary text
        'success' => '#10B981',      // Success states
        'warning' => '#F59E0B',       // Warnings
        'danger' => '#EF4444',       // Errors
    ],
    
    'logo' => [
        'header' => '/assets/brand/logo-header.svg',
        'footer' => '/assets/brand/logo-footer.svg',
        'email' => '/assets/brand/logo-email.png',
        'favicon' => '/assets/brand/favicon.ico',
        'app_icon' => '/assets/brand/icon-192.png',
    ],
];
```

### Step 2: Template Branding Configuration

Configure template for white labeling:

```twig
{# templates/default/partials/header.html.twig #}

{% set brand = getBrandConfig() %}

<header class="site-header" style="--primary-color: {{ brand.colors.primary }}">
    <div class="container">
        <a href="{{ baseUrl }}" class="brand-logo">
            <img src="{{ brand.logo.header }}" 
                 alt="{{ brand.name }}" 
                 class="logo-img"
                 loading="eager">
        </a>
        
        <nav class="main-nav">
            {% if client %}
                {# Authenticated navigation #}
                <a href="{{ constant('WEB_ROOT') }}/clientarea.php">Dashboard</a>
                <a href="{{ constant('WEB_ROOT') }}/clientarea.php?action=services">Services</a>
                <a href="{{ constant('WEB_ROOT') }}/clientarea.php?action=domains">Domains</a>
                <a href="{{ constant('WEB_ROOT') }}/supporttickets.php">Support</a>
                <a href="{{ constant('WEB_ROOT') }}/clientarea.php?action=account">Account</a>
            {% else %}
                {# Public navigation #}
                <a href="{{ constant('WEB_ROOT') }}/">Home</a>
                <a href="{{ constant('WEB_ROOT') }}/cart.php">Order</a>
                <a href="{{ constant('WEB_ROOT') }}/knowledgebase.php">Knowledge Base</a>
                <a href="{{ constant('WEB_ROOT') }}/contact.php">Contact</a>
                <a href="{{ constant('WEB_ROOT') }}/login.php" class="btn btn-primary">
                    Client Login
                </a>
            {% endif %}
        </nav>
    </div>
</header>

<style>
    .site-header {
        background: linear-gradient(135deg, {{ brand.colors.primary }} 0%, {{ brand.colors.secondary }} 100%);
    }
    
    .btn-primary {
        background-color: {{ brand.colors.primary }};
        border-color: {{ brand.colors.primary }};
    }
    
    .btn-primary:hover {
        background-color: {{ brand.colors.secondary }};
        border-color: {{ brand.colors.secondary }};
    }
</style>
```

### Step 3: Dynamic Brand Configuration

Implement dynamic brand loading:

```php
// includes/functions/brand.php

/**
 * Get brand configuration
 */
function getBrandConfig(): array
{
    static $config = null;
    
    if ($config !== null) {
        return $config;
    }
    
    // Check for custom brand settings
    $brandSettings = Capsule::table('tblconfiguration')
        ->whereIn('setting', ['BrandName', 'BrandColors', 'BrandLogo'])
        ->pluck('value', 'setting')
        ->toArray();
    
    if (!empty($brandSettings)) {
        $config = [
            'name' => $brandSettings['BrandName'] ?? 'WHMCS',
            'colors' => json_decode($brandSettings['BrandColors'] ?? '{}', true),
            'logo' => json_decode($brandSettings['BrandLogo'] ?? '{}', true),
        ];
    } else {
        // Default to system configuration
        $config = [
            'name' => get_config('CompanyName'),
            'colors' => [
                'primary' => get_config('PrimaryColor') ?: '#1d8eed',
                'secondary' => get_config('SecondaryColor') ?: '#1581c7',
            ],
            'logo' => [
                'header' => get_config('LogoURL') ?: '/assets/img/logo.png',
            ],
        ];
    }
    
    return $config;
}

/**
 * Apply brand to email templates
 */
function applyBrandToEmail(string $templateContent, array $brand = null): string
{
    $brand = $brand ?? getBrandConfig();
    
    $replacements = [
        '{{BRAND_NAME}}' => $brand['name'],
        '{{BRAND_LOGO}}' => '<img src="' . $brand['logo']['email'] . '" alt="' . $brand['name'] . '">',
        '{{BRAND_PRIMARY_COLOR}}' => $brand['colors']['primary'] ?? '#1d8eed',
        '{{SUPPORT_EMAIL}}' => $brand['support_email'] ?? get_config('Email'),
        '{{SUPPORT_URL}}' => $brand['website'] ?? get_config('SystemURL'),
        '{{COMPANY_NAME}}' => $brand['company_name'] ?? get_config('CompanyName'),
    ];
    
    return str_replace(array_keys($replacements), array_values($replacements), $templateContent);
}

/**
 * Generate branded email footer
 */
function getBrandedEmailFooter(): string
{
    $brand = getBrandConfig();
    
    return <<<HTML
    <div style="background-color: {$brand['colors']['background']}; padding: 30px; text-align: center; border-top: 3px solid {$brand['colors']['primary']};">
        <img src="{$brand['logo']['footer']}" alt="{$brand['name']}" style="max-height: 50px; margin-bottom: 20px;">
        <p style="color: {$brand['colors']['text_light']}; font-size: 12px; margin: 5px 0;">
            {$brand['company_name']}<br>
            {$brand['company_address']}
        </p>
        <p style="color: {$brand['colors']['text_light']}; font-size: 12px; margin: 10px 0;">
            <a href="{$brand['website']}" style="color: {$brand['colors']['primary']};">Website</a> |
            <a href="{$brand['website']}/privacy" style="color: {$brand['colors']['primary']};">Privacy Policy</a> |
            <a href="{$brand['website']}/terms" style="color: {$brand['colors']['primary']};">Terms of Service</a>
        </p>
    </div>
    HTML;
}
```

### Step 4: Remove WHMCS Branding

Systematic removal of WHMCS references:

```php
// Remove WHMCS branding from various locations

/**
 * Hook to modify powered by text
 */
add_hook('ClientAreaFooterOutput', 1, function($vars) {
    return ''; // Remove powered by WHMCS
});

/**
 * Remove WHMCS from page titles
 */
add_hook('ClientAreaPageTitle', 1, function($vars) {
    $title = $vars['title'] ?? '';
    $title = str_replace(' - WHMCS', '', $title);
    $title = str_replace(' | WHMCS', '', $title);
    return ['title' => $title];
});

/**
 * Replace WHMCS links in footer
 */
add_hook('ClientAreaFooter', 1, function($vars) {
    return [
        'copyright' => '&copy; ' . date('Y') . ' ' . getBrandConfig()['name'] . '. All rights reserved.',
    ];
});

/**
 * Admin area - remove WHMCS branding
 */
add_hook('AdminAreaFooter', 1, function($vars) {
    return '<script>
        // Remove WHMCS branding elements
        document.querySelectorAll(".powered-by-whmcs, .whmcs-footer").forEach(el => el.remove());
    </script>';
});

/**
 * Custom favicon
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $brand = getBrandConfig();
    return '<link rel="icon" type="image/x-icon" href="' . $brand['logo']['favicon'] . '">';
});

/**
 * Custom CSS to hide WHMCS branding
 */
function getBrandingRemovalCSS(): string
{
    return <<<CSS
    /* Remove WHMCS powered by */
    .powered-by, .powered-by-whmcs, .whmcs-branding {
        display: none !important;
    }
    
    /* Remove WHMCS footer links */
    #footer p:last-child a[href*="whmcs"] {
        display: none;
    }
    
    /* Remove WHMCS from meta */
    meta[name="generator"] {
        display: none;
    }
    CSS;
}
```

### Step 5: Domain Configuration

Set up custom domain for white label:

```php
// Custom domain configuration

/**
 * Allow custom domain mapping
 */
add_hook('SystemURLSetup', 1, function($vars) {
    // Check for custom domain
    $customDomain = $_SERVER['HTTP_HOST'] ?? '';
    
    // Map domain to brand
    $brandMapping = [
        'billing.yourcompany.com' => 'primary',
        'store.yourcompany.com' => 'secondary',
        'portal.client-domain.com' => 'client_123',
    ];
    
    $brandSlug = $brandMapping[$customDomain] ?? null;
    
    if ($brandSlug) {
        // Load brand configuration
        $brand = loadBrandBySlug($brandSlug);
        
        // Override system URLs
        return [
            'SystemURL' => 'https://' . $customDomain . '/',
            'SystemSSLURL' => 'https://' . $customDomain . '/',
            'Domain' => $customDomain,
        ];
    }
});

/**
 * SSL configuration for custom domains
 */
function configureCustomDomainSSL(string $domain): bool
{
    // Let's Encrypt or commercial SSL
    $sslConfig = [
        'provider' => 'letsencrypt',
        'domain' => $domain,
        'challenges_path' => '/var/www/whmcs/.well-known/acme-challenge',
    ];
    
    // Request certificate
    $result = shell_exec("certbot certonly --webroot -w {$sslConfig['challenges_path']} -d {$domain} -d www.{$domain}");
    
    return strpos($result, 'Congratulations') !== false;
}

/**
 * Reverse proxy configuration
 */
function configureReverseProxy(string $domain, string $whmcsInstance): array
{
    return [
        'nginx_config' => <<<NGINX
server {
    listen 80;
    server_name {$domain};
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name {$domain};
    
    ssl_certificate /etc/ssl/certs/{$domain}.crt;
    ssl_certificate_key /etc/ssl/private/{$domain}.key;
    
    location / {
        proxy_pass http://127.0.0.1:8080; # WHMCS instance port
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
NGINX,
    ];
}
```

### Step 6: Support Portal Branding

Customize support interface:

```php
// Support ticket interface branding

/**
 * Custom ticket form
 */
function getBrandedTicketForm(): string
{
    $brand = getBrandConfig();
    
    return <<<HTML
    <div class="support-form" style="border-top: 4px solid {$brand['colors']['primary']};">
        <h2 style="color: {$brand['colors']['primary']};">
            Contact {$brand['name']} Support
        </h2>
        
        <div class="contact-info" style="background: {$brand['colors']['background']}; padding: 20px; margin: 20px 0;">
            <p><strong>Email:</strong> {$brand['support_email']}</p>
            <p><strong>Phone:</strong> {$brand['support_phone']}</p>
            <p><strong>Hours:</strong> {$brand['support_hours']}</p>
        </div>
        
        <form method="post" action="supporttickets.php">
            <!-- Form fields -->
        </form>
    </div>
    HTML;
}

/**
 * Custom ticket notification
 */
function sendBrandedTicketNotification(int $ticketId, string $event): void
{
    $brand = getBrandConfig();
    
    $ticket = Capsule::table('tbltickets')
        ->where('id', $ticketId)
        ->first();
    
    $client = Capsule::table('tblclients')
        ->where('id', $ticket->userid)
        ->first();
    
    $subject = "[Ticket #{$ticketId}] {$ticket->subject}";
    
    $message = <<<HTML
    Dear {$client->firstname} {$client->lastname},
    
    {$brand['name']} Support Team
    
    ---
    Ticket #{$ticketId}: {$ticket->subject}
    Status: {$ticket->status}
    
    {$brand['company_name']}
    {$brand['support_email']}
    {$brand['support_phone']}
    HTML;
    
    send_email($client->email, $subject, $message);
}
```

## Best Practices

1. **Complete rebrand** - Replace all WHMCS references consistently
2. **Custom domain** - Use your own domain for full white label
3. **Professional templates** - Invest in quality design
4. **Consistent colors** - Use brand colors throughout
5. **Update legal pages** - Terms, privacy, refund policies
6. **Support documentation** - Custom KB and FAQs
7. **Test thoroughly** - Check every page for missed references
8. **SSL everywhere** - Secure all branded domains

## Common Pitfalls to Avoid

1. **Incomplete removal** - WHMCS references remain visible
2. **Inconsistent branding** - Different styles across pages
3. **Missing legal pages** - Required for compliance
4. **Poor quality logos** - Use vector formats
5. **Wrong color contrast** - Accessibility issues
6. **Ignoring mobile** - Responsive design essential
7. **Hardcoded values** - Use configuration variables
8. **Forgetting email templates** - Match web branding
