# WHMCS Meta Tags Configuration Workflow

## Purpose
Configure comprehensive meta tags in WHMCS for better SEO and social sharing.

## Prerequisites
- WHMCS installation
- Template access
- Hook system knowledge

## Step-by-Step Process

### Step 1: Create Meta Tags Hook File

**Create hooks/meta_tags.php:**
```php
<?php
/**
 * WHMCS Meta Tags Configuration
 * Comprehensive meta tag management
 */

use WHMCS\Config\Setting;

// Company info helpers
function getCompanyName() {
    return Setting::getValue('CompanyName') ?: 'Your Company';
}

function getSystemUrl() {
    return rtrim(Setting::getValue('SystemURL'), '/');
}

function getSystemSSLUrl() {
    return rtrim(Setting::getValue('SystemSSLURL'), '/') ?: getSystemUrl();
}

function getLogoUrl() {
    return getSystemUrl() . '/assets/img/logo.png';
}

function getDefaultDescription() {
    return Setting::getValue('MetaDescription') ?: 'Professional web hosting and cloud services';
}

function getDefaultKeywords() {
    return Setting::getValue('MetaKeywords') ?: 'web hosting, vps, cloud, domains';
}
```

### Step 2: Master Meta Tags Hook

```php
<?php
/**
 * Add comprehensive meta tags
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $meta = '';
    $title = '';
    $description = '';
    $keywords = '';
    $canonical = '';
    $ogTags = '';
    $twitterTags = '';
    
    $systemUrl = getSystemUrl();
    $companyName = getCompanyName();
    $currentUrl = getCurrentFullUrl();
    
    // Generate canonical URL
    $canonical = '<link rel="canonical" href="' . htmlspecialchars($canonical) . '">' . "\n";
    
    // Page-specific configuration
    $pageConfig = getPageMetaConfig($vars);
    $title = $pageConfig['title'] ?? generateDefaultTitle($vars, $companyName);
    $description = $pageConfig['description'] ?? getDefaultDescription();
    $keywords = $pageConfig['keywords'] ?? getDefaultKeywords();
    
    // Basic Meta Tags
    $meta .= '<title>' . htmlspecialchars($title) . '</title>' . "\n";
    $meta .= '<meta name="description" content="' . htmlspecialchars($description) . '">' . "\n";
    $meta .= '<meta name="keywords" content="' . htmlspecialchars($keywords) . '">' . "\n";
    $meta .= '<meta name="author" content="' . htmlspecialchars($companyName) . '">' . "\n";
    $meta .= '<meta name="robots" content="' . ($pageConfig['robots'] ?? 'index, follow') . '">' . "\n";
    $meta .= '<meta name="viewport" content="width=device-width, initial-scale=1.0">' . "\n";
    
    // Geographic Meta Tags
    $meta .= '<meta name="geo.region" content="US-CA">' . "\n";
    $meta .= '<meta name="geo.placename" content="' . htmlspecialchars($companyName) . '">' . "\n";
    
    // Open Graph Tags
    $ogTags .= '<meta property="og:title" content="' . htmlspecialchars($title) . '">' . "\n";
    $ogTags .= '<meta property="og:description" content="' . htmlspecialchars($description) . '">' . "\n";
    $ogTags .= '<meta property="og:type" content="' . ($pageConfig['og_type'] ?? 'website') . '">' . "\n";
    $ogTags .= '<meta property="og:url" content="' . htmlspecialchars($currentUrl) . '">' . "\n";
    $ogTags .= '<meta property="og:site_name" content="' . htmlspecialchars($companyName) . '">' . "\n";
    $ogTags .= '<meta property="og:locale" content="en_US">' . "\n";
    
    if (!empty($pageConfig['og_image'])) {
        $ogTags .= '<meta property="og:image" content="' . htmlspecialchars($pageConfig['og_image']) . '">' . "\n";
        $ogTags .= '<meta property="og:image:secure_url" content="' . htmlspecialchars(str_replace('http://', 'https://', $pageConfig['og_image'])) . '">' . "\n";
        $ogTags .= '<meta property="og:image:width" content="1200">' . "\n";
        $ogTags .= '<meta property="og:image:height" content="630">' . "\n";
        $ogTags .= '<meta property="og:image:alt" content="' . htmlspecialchars($title) . '">' . "\n";
    }
    
    // Twitter Card Tags
    $twitterTags .= '<meta name="twitter:card" content="' . (!empty($pageConfig['og_image']) ? 'summary_large_image' : 'summary') . '">' . "\n";
    $twitterTags .= '<meta name="twitter:title" content="' . htmlspecialchars($title) . '">' . "\n";
    $twitterTags .= '<meta name="twitter:description" content="' . htmlspecialchars($description) . '">' . "\n";
    $twitterTags .= '<meta name="twitter:site" content="@' . ($pageConfig['twitter_handle'] ?? 'yourtwitter') . '">' . "\n";
    
    if (!empty($pageConfig['og_image'])) {
        $twitterTags .= '<meta name="twitter:image" content="' . htmlspecialchars($pageConfig['og_image']) . '">' . "\n";
    }
    
    return $meta . $canonical . $ogTags . $twitterTags;
});

function getCurrentFullUrl() {
    $protocol = (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off') ? 'https' : 'http';
    return $protocol . '://' . ($_SERVER['HTTP_HOST'] ?? '') . ($_SERVER['REQUEST_URI'] ?? '/');
}

function generateDefaultTitle($vars, $companyName) {
    $pageTitle = $vars['pagetitle'] ?? '';
    return !empty($pageTitle) ? $pageTitle . ' - ' . $companyName : $companyName . ' - Professional Web Hosting';
}
```

### Step 3: Page-Specific Configuration

```php
<?php
/**
 * Get meta configuration for specific pages
 */
function getPageMetaConfig($vars) {
    $config = [];
    $filename = $vars['filename'] ?? '';
    
    switch ($filename) {
        case '':
        case 'index':
            $config = [
                'title' => 'Premium Web Hosting & Cloud Solutions | ' . getCompanyName(),
                'description' => 'Reliable web hosting, VPS, cloud servers, and domain registration. 99.9% uptime, 24/7 support, 30-day money back guarantee.',
                'keywords' => 'web hosting, vps hosting, cloud servers, domain registration, ssl certificates',
                'og_type' => 'website',
                'og_image' => getSystemUrl() . '/images/og-homepage.jpg',
                'robots' => 'index, follow'
            ];
            break;
            
        case 'cart':
            $config = [
                'title' => 'Shopping Cart - Complete Your Order | ' . getCompanyName(),
                'description' => 'Review your order and complete your purchase securely.',
                'keywords' => 'shopping cart, checkout, order',
                'og_type' => 'website',
                'robots' => 'noindex, follow'
            ];
            break;
            
        case 'register':
            $config = [
                'title' => 'Create Account - Client Portal | ' . getCompanyName(),
                'description' => 'Create your client account to manage services, invoices, and support tickets.',
                'keywords' => 'register, signup, create account, client portal',
                'og_type' => 'website',
                'robots' => 'noindex, follow'
            ];
            break;
            
        case 'login':
            $config = [
                'title' => 'Client Login | ' . getCompanyName(),
                'description' => 'Login to your client area to manage your hosting services.',
                'keywords' => 'login, client area, member login',
                'og_type' => 'website',
                'robots' => 'noindex, follow'
            ];
            break;
            
        case 'announcements':
            $config = [
                'title' => 'Latest News & Announcements | ' . getCompanyName(),
                'description' => 'Stay updated with the latest news, maintenance updates, and special offers.',
                'keywords' => 'news, announcements, updates, maintenance',
                'og_type' => 'website'
            ];
            break;
            
        case 'knowledgebase':
            $config = [
                'title' => 'Knowledge Base & Help Articles | ' . getCompanyName(),
                'description' => 'Find answers to common questions and helpful guides for our services.',
                'keywords' => 'knowledge base, help, documentation, guides, tutorials',
                'og_type' => 'website'
            ];
            break;
            
        case 'contact':
            $config = [
                'title' => 'Contact Us - Get Support | ' . getCompanyName(),
                'description' => 'Contact our support team for assistance. Available 24/7 via live chat, email, and phone.',
                'keywords' => 'contact, support, help, phone, email, chat',
                'og_type' => 'website',
                'og_image' => getSystemUrl() . '/images/og-contact.jpg'
            ];
            break;
            
        case 'affiliates':
            $config = [
                'title' => 'Affiliate Program - Earn Commission | ' . getCompanyName(),
                'description' => 'Join our affiliate program and earn commissions for every customer you refer.',
                'keywords' => 'affiliate, commission, referral, earn money',
                'og_type' => 'website',
                'og_image' => getSystemUrl() . '/images/og-affiliates.jpg'
            ];
            break;
            
        case 'serverstatus':
            $config = [
                'title' => 'System Status - Network Uptime | ' . getCompanyName(),
                'description' => 'Check the current status of our servers and network infrastructure.',
                'keywords' => 'server status, uptime, network status, system status',
                'robots' => 'index, follow'
            ];
            break;
            
        case 'domainchecker':
            $config = [
                'title' => 'Domain Name Search - Find Your Perfect Domain | ' . getCompanyName(),
                'description' => 'Search and register domain names. Great prices, free DNS, and domain transfer services.',
                'keywords' => 'domain names, domain registration, domain search, domain transfer',
                'og_type' => 'website',
                'og_image' => getSystemUrl() . '/images/og-domains.jpg'
            ];
            break;
            
        default:
            // Use WHMCS default for other pages
            if (!empty($vars['pagetitle'])) {
                $config['title'] = $vars['pagetitle'] . ' - ' . getCompanyName();
            }
            $config['description'] = getDefaultDescription();
            $config['keywords'] = getDefaultKeywords();
            break;
    }
    
    // Handle product pages
    if ($filename === 'cart' && isset($vars['productinfo'])) {
        $product = $vars['productinfo'];
        $config['title'] = $product['name'] . ' - Order Now | ' . getCompanyName();
        $config['description'] = substr($product['description'], 0, 160);
        if (!empty($product['image'])) {
            $config['og_image'] = getSystemUrl() . '/' . $product['image'];
        }
    }
    
    return $config;
}
```

### Step 4: Article/Announcement Meta Tags

```php
<?php
/**
 * Add meta tags for articles and announcements
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $meta = '';
    
    // Announcements
    if (isset($vars['announcement']) && is_array($vars['announcement'])) {
        $announcement = $vars['announcement'];
        $title = $announcement['title'] ?? 'Announcement';
        $date = $announcement['date'] ?? date('Y-m-d');
        $summary = $announcement['summary'] ?? '';
        
        $meta .= '<title>' . htmlspecialchars($title) . ' - News | ' . getCompanyName() . '</title>' . "\n";
        $meta .= '<meta name="description" content="' . htmlspecialchars(substr($summary, 0, 160)) . '">' . "\n";
        $meta .= '<meta property="og:type" content="article">' . "\n";
        $meta .= '<meta property="article:published_time" content="' . $date . 'T00:00:00+00:00">' . "\n";
        $meta .= '<meta property="article:section" content="Announcements">' . "\n";
    }
    
    // Knowledgebase articles
    if (isset($vars['kbarticle']) && is_array($vars['kbarticle'])) {
        $article = $vars['kbarticle'];
        $title = $article['title'] ?? 'Knowledgebase Article';
        
        $meta .= '<title>' . htmlspecialchars($title) . ' - Knowledge Base | ' . getCompanyName() . '</title>' . "\n";
        $meta .= '<meta name="description" content="' . htmlspecialchars(substr($article['article'] ?? '', 0, 160)) . '">' . "\n";
    }
    
    return ['custom_meta' => $meta];
});
```

### Step 5: Mobile-Specific Meta Tags

```php
<?php
/**
 * Mobile-specific meta tags
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $mobile = '';
    
    // WebApp capable
    $mobile .= '<meta name="mobile-web-app-capable" content="yes">' . "\n";
    $mobile .= '<meta name="apple-mobile-web-app-capable" content="yes">' . "\n";
    $mobile .= '<meta name="apple-mobile-web-app-status-bar-style" content="default">' . "\n";
    $mobile .= '<meta name="apple-mobile-web-app-title" content="' . htmlspecialchars(getCompanyName()) . '">' . "\n";
    
    // Theme color
    $mobile .= '<meta name="theme-color" content="#0055cc">' . "\n";
    $mobile .= '<meta name="msapplication-navbutton-color" content="#0055cc">' . "\n";
    
    // Format detection
    $mobile .= '<meta name="format-detection" content="telephone=no">' . "\n";
    
    return $mobile;
});
```

### Step 6: Verification Meta Tags

```php
<?php
/**
 * Search engine verification tags
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $verification = '';
    
    // Google Search Console
    $googleVerification = Setting::getValue('GoogleSiteVerification');
    if ($googleVerification) {
        $verification .= '<meta name="google-site-verification" content="' . htmlspecialchars($googleVerification) . '">' . "\n";
    }
    
    // Bing Webmaster Tools
    $bingVerification = Setting::getValue('BingSiteVerification');
    if ($bingVerification) {
        $verification .= '<meta name="msvalidate.01" content="' . htmlspecialchars($bingVerification) . '">' . "\n";
    }
    
    // Yandex
    $yandexVerification = Setting::getValue('YandexSiteVerification');
    if ($yandexVerification) {
        $verification .= '<meta name="yandex-verification" content="' . htmlspecialchars($yandexVerification) . '">' . "\n";
    }
    
    // Pinterest
    $pinterestVerification = Setting::getValue('PinterestSiteVerification');
    if ($pinterestVerification) {
        $verification .= '<meta name="p:domain_verify" content="' . htmlspecialchars($pinterestVerification) . '">' . "\n";
    }
    
    return $verification;
});
```

### Step 7: Hreflang Configuration

```php
<?php
/**
 * Add hreflang tags for multilingual sites
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $hreflang = '';
    $systemUrl = getSystemUrl();
    $currentPath = parse_url($_SERVER['REQUEST_URI'] ?? '/', PHP_URL_PATH);
    
    // Default language
    $hreflang .= '<link rel="alternate" hreflang="x-default" href="' . $systemUrl . $currentPath . '">' . "\n";
    $hreflang .= '<link rel="alternate" hreflang="en" href="' . $systemUrl . $currentPath . '">' . "\n";
    
    // Additional languages (customize as needed)
    $languages = [
        'en' => $systemUrl,
        'es' => 'https://es.' . str_replace('www.', '', parse_url($systemUrl, PHP_URL_HOST)) . $currentPath,
        'fr' => 'https://fr.' . str_replace('www.', '', parse_url($systemUrl, PHP_URL_HOST)) . $currentPath,
        'de' => 'https://de.' . str_replace('www.', '', parse_url($systemUrl, PHP_URL_HOST)) . $currentPath,
    ];
    
    foreach ($languages as $lang => $url) {
        if ($lang === 'en') continue; // Already added
        $hreflang .= '<link rel="alternate" hreflang="' . $lang . '" href="' . htmlspecialchars($url) . '">' . "\n";
    }
    
    return $hreflang;
});
```

### Step 8: Dublin Core Meta Tags

```php
<?php
/**
 * Dublin Core metadata for academic/library indexing
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $dublinCore = '';
    
    $dcTags = [
        'DC.title' => !empty($vars['pagetitle']) ? $vars['pagetitle'] : getCompanyName(),
        'DC.creator' => getCompanyName(),
        'DC.subject' => getDefaultKeywords(),
        'DC.description' => getDefaultDescription(),
        'DC.publisher' => getCompanyName(),
        'DC.contributor' => '',
        'DC.date' => date('Y-m-d'),
        'DC.type' => 'Service',
        'DC.format' => 'text/html',
        'DC.identifier' => getCurrentFullUrl(),
        'DC.source' => getSystemUrl(),
        'DC.language' => 'en',
        'DC.relation' => '',
        'DC.coverage' => '',
        'DC.rights' => 'All rights reserved'
    ];
    
    foreach ($dcTags as $name => $content) {
        if (!empty($content)) {
            $dublinCore .= '<meta name="' . $name . '" content="' . htmlspecialchars($content) . '">' . "\n";
        }
    }
    
    return $dublinCore;
});
```

## Best Practices
- Keep titles under 60 characters
- Keep descriptions under 160 characters
- Use unique meta tags per page
- Include primary keywords naturally
- Use structured data alongside meta tags
- Test social sharing previews
- Update meta tags when content changes
- Monitor indexing in search consoles
- Avoid duplicate meta descriptions
- Use proper Open Graph images (1200x630px)
