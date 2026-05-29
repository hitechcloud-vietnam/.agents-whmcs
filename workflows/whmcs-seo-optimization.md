# WHMCS SEO Optimization Workflow

## Purpose
Optimize WHMCS for search engines to improve organic visibility and rankings.

## Prerequisites
- WHMCS installation
- Admin access
- Basic SEO knowledge

## Step-by-Step Process

### Step 1: Configure Basic SEO Settings

**WHMCS Admin Settings:**
```
1. Navigate to: Setup > General Settings > Security
2. Enable Search Engine Friendly URLs
3. Enable Disable SEOFriendly URLs for API

Navigate to: Setup > General Settings > SEO
4. Set Default Title: {companyName} - {pageTitle}
5. Set Default Keywords
6. Configure Meta Description Template
```

### Step 2: Enable SEO-Friendly URLs

**Via configuration.php:**
```php
// Add or modify in /whmcs/configuration.php
$seo_friendly_urls = true;

// Or use htaccess rules for Apache:
/*
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^([^/]+)/?$ index.php?rp=$1 [L,QSA]
*/
```

### Step 3: Create SEO Hook File

**Create hooks/seo_optimization.php:**
```php
<?php
/**
 * WHMCS SEO Optimization Hooks
 */

// Set canonical URLs
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $protocol = isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on' ? 'https' : 'http';
    $host = $_SERVER['HTTP_HOST'] ?? '';
    $uri = $_SERVER['REQUEST_URI'] ?? '';
    $canonical = $protocol . '://' . $host . $uri;
    
    // Remove trailing slash except for root
    $canonical = rtrim($canonical, '/');
    if ($canonical === '') {
        $canonical = $protocol . '://' . $host;
    }
    
    $seo = '<link rel="canonical" href="' . htmlspecialchars($canonical) . '">' . "\n";
    
    // Prevent indexing of duplicate pages
    if (isset($_GET['nocache']) || isset($_GET['session'])) {
        $seo .= '<meta name="robots" content="noindex, nofollow">' . "\n";
    }
    
    return $seo;
});

// Add structured data
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $structuredData = [];
    
    // Organization Schema
    $structuredData[] = [
        '@context' => 'https://schema.org',
        '@type' => 'Organization',
        'name' => \WHMCS\Config\Setting::getValue('CompanyName'),
        'url' => \WHMCS\Config\Setting::getValue('SystemURL'),
        'logo' => \WHMCS\Config\Setting::getValue('SystemURL') . '/assets/img/logo.png',
        'sameAs' => getSocialProfiles()
    ];
    
    // WebSite Schema
    $structuredData[] = [
        '@context' => 'https://schema.org',
        '@type' => 'WebSite',
        'name' => \WHMCS\Config\Setting::getValue('CompanyName'),
        'url' => \WHMCS\Config\Setting::getValue('SystemURL'),
        'potentialAction' => [
            '@type' => 'SearchAction',
            'target' => \WHMCS\Config\Setting::getValue('SystemURL') . '/search?q={search_term_string}',
            'query-input' => 'required name=search_term_string'
        ]
    ];
    
    $jsonLd = '';
    foreach ($structuredData as $data) {
        $jsonLd .= '<script type="application/ld+json">' . json_encode($data, JSON_UNESCAPED_SLASHES) . '</script>' . "\n";
    }
    
    return ['structured_data' => $jsonLd];
});

function getSocialProfiles() {
    $profiles = [];
    
    if ($facebook = \WHMCS\Config\Setting::getValue('FacebookURL')) {
        $profiles[] = $facebook;
    }
    if ($twitter = \WHMCS\Config\Setting::getValue('TwitterURL')) {
        $profiles[] = $twitter;
    }
    if ($linkedin = \WHMCS\Config\Setting::getValue('LinkedInURL')) {
        $profiles[] = $linkedin;
    }
    
    return $profiles;
}
```

### Step 4: Page-Specific Meta Tags

```php
<?php
/**
 * Dynamic meta tags per page
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $meta = '';
    $title = '';
    $description = '';
    $ogImage = '';
    
    switch ($vars['filename'] ?? '') {
        case 'index':
            $title = 'Premium Web Hosting & Cloud Solutions';
            $description = 'Reliable web hosting, VPS, and cloud services. 99.9% uptime guarantee with 24/7 support.';
            $ogImage = '/images/og-homepage.jpg';
            break;
            
        case 'cart':
            $title = 'Shopping Cart - Review Your Order';
            $description = 'Review your cart and complete your order securely.';
            break;
            
        case ' register':
        case 'login':
            $title = 'Client Portal - ' . \WHMCS\Config\Setting::getValue('CompanyName');
            $description = 'Access your client portal to manage services, invoices, and support.';
            break;
            
        case 'contact':
            $title = 'Contact Us - Get Support';
            $description = 'Contact our support team for assistance with your services.';
            break;
            
        default:
            // Use WHMCS defaults
            if (!empty($vars['pagetitle'])) {
                $title = $vars['pagetitle'] . ' - ' . \WHMCS\Config\Setting::getValue('CompanyName');
            }
            break;
    }
    
    // Generate meta tags
    if ($title) {
        $meta .= '<title>' . htmlspecialchars($title) . '</title>' . "\n";
    }
    
    if ($description) {
        $meta .= '<meta name="description" content="' . htmlspecialchars($description) . '">' . "\n";
    }
    
    // Open Graph tags
    $meta .= '<meta property="og:title" content="' . htmlspecialchars($title ?: getDefaultTitle()) . '">' . "\n";
    $meta .= '<meta property="og:description" content="' . htmlspecialchars($description ?: getDefaultDescription()) . '">' . "\n";
    $meta .= '<meta property="og:type" content="website">' . "\n";
    $meta .= '<meta property="og:url" content="' . getCurrentUrl() . '">' . "\n";
    $meta .= '<meta property="og:site_name" content="' . htmlspecialchars(\WHMCS\Config\Setting::getValue('CompanyName')) . '">' . "\n";
    
    if ($ogImage) {
        $meta .= '<meta property="og:image" content="' . rtrim(\WHMCS\Config\Setting::getValue('SystemURL'), '/') . $ogImage . '">' . "\n";
        $meta .= '<meta property="og:image:width" content="1200">' . "\n";
        $meta .= '<meta property="og:image:height" content="630">' . "\n";
    }
    
    // Twitter Card tags
    $meta .= '<meta name="twitter:card" content="summary_large_image">' . "\n";
    $meta .= '<meta name="twitter:title" content="' . htmlspecialchars($title ?: getDefaultTitle()) . '">' . "\n";
    $meta .= '<meta name="twitter:description" content="' . htmlspecialchars($description ?: getDefaultDescription()) . '">' . "\n";
    
    return $meta;
});

function getCurrentUrl() {
    $protocol = isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on' ? 'https' : 'http';
    return $protocol . '://' . ($_SERVER['HTTP_HOST'] ?? '') . ($_SERVER['REQUEST_URI'] ?? '');
}
```

### Step 5: XML Sitemap Generation

```php
<?php
/**
 * Generate XML sitemap for WHMCS
 */
add_hook('DailyCronJob', 1, function($vars) {
    generateSitemap();
});

function generateSitemap() {
    $baseUrl = rtrim(\WHMCS\Config\Setting::getValue('SystemURL'), '/');
    $priority = 0.8;
    $changefreq = 'weekly';
    
    $urls = [];
    
    // Static pages
    $staticPages = [
        ['loc' => '/', 'priority' => 1.0, 'changefreq' => 'daily'],
        ['loc' => '/announcements/', 'priority' => 0.7, 'changefreq' => 'weekly'],
        ['loc' => '/knowledgebase/', 'priority' => 0.7, 'changefreq' => 'weekly'],
        ['loc' => '/serverstatus/', 'priority' => 0.6, 'changefreq' => 'daily'],
        ['loc' => '/contact/', 'priority' => 0.5, 'changefreq' => 'monthly'],
        ['loc' => '/register/', 'priority' => 0.6, 'changefreq' => 'yearly'],
        ['loc' => '/login/', 'priority' => 0.6, 'changefreq' => 'yearly'],
    ];
    
    foreach ($staticPages as $page) {
        $urls[] = [
            'loc' => $baseUrl . $page['loc'],
            'priority' => $page['priority'],
            'changefreq' => $page['changefreq'],
            'lastmod' => date('Y-m-d')
        ];
    }
    
    // Product pages
    $products = \WHMCS\Database\Capsule::table('tblproducts')
        ->where('hidden', 0)
        ->where('showorder', 1)
        ->get(['id', 'name']);
    
    foreach ($products as $product) {
        $slug = slugify($product->name);
        $urls[] = [
            'loc' => $baseUrl . '/store/' . $slug,
            'priority' => 0.8,
            'changefreq' => 'weekly',
            'lastmod' => date('Y-m-d')
        ];
    }
    
    // Knowledgebase articles
    $articles = \WHMCS\Database\Capsule::table('tblknowledgebase')
        ->where('published', 1)
        ->get(['id', 'title', 'modified']);
    
    foreach ($articles as $article) {
        $slug = slugify($article->title);
        $urls[] = [
            'loc' => $baseUrl . '/knowledgebase/' . $article->id . '/' . $slug,
            'priority' => 0.5,
            'changefreq' => 'monthly',
            'lastmod' => date('Y-m-d', strtotime($article->modified ?? 'now'))
        ];
    }
    
    // Generate XML
    $xml = '<?xml version="1.0" encoding="UTF-8"?>' . "\n";
    $xml .= '<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">' . "\n";
    
    foreach ($urls as $url) {
        $xml .= '  <url>' . "\n";
        $xml .= '    <loc>' . htmlspecialchars($url['loc']) . '</loc>' . "\n";
        $xml .= '    <lastmod>' . $url['lastmod'] . '</lastmod>' . "\n";
        $xml .= '    <changefreq>' . $url['changefreq'] . '</changefreq>' . "\n";
        $xml .= '    <priority>' . $url['priority'] . '</priority>' . "\n";
        $xml .= '  </url>' . "\n";
    }
    
    $xml .= '</urlset>';
    
    // Save sitemap
    $sitemapPath = ROOTDIR . '/sitemap.xml';
    file_put_contents($sitemapPath, $xml);
    
    logActivity('Sitemap generated: ' . count($urls) . ' URLs');
    
    return $sitemapPath;
}

function slugify($text) {
    $text = preg_replace('~[^\pL\d]+~u', '-', $text);
    $text = iconv('utf-8', 'us-ascii//TRANSLIT', $text);
    $text = preg_replace('~[^-\w]+~', '', $text);
    $text = trim($text, '-');
    $text = preg_replace('~-+~', '-', $text);
    return strtolower($text);
}
```

### Step 6: Robots.txt Configuration

**Create /whmcs/robots.txt:**
```
User-agent: *
Allow: /

# Disallow admin and configuration areas
Disallow: /admin/
Disallow: /configuration.php
Disallow: /includes/
Disallow: /vendor/
Disallow: /api/

# Disallow duplicate content
Disallow: /*?*
Allow: /*?q=*

# Disallow sensitive pages
Disallow: /clientarea.php?action=details
Disallow: /clientarea.php?action=security

# Sitemap location
Sitemap: https://yourdomain.com/sitemap.xml

# Crawl delay (if needed)
Crawl-delay: 10
```

### Step 7: Performance SEO

```css
/* Preconnect to external resources */
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

<!-- DNS prefetch -->
<link rel="dns-prefetch" href="https://www.google-analytics.com">
```

### Step 8: Breadcrumb Schema

```php
<?php
/**
 * Add breadcrumb structured data
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $breadcrumbs = [];
    
    $breadcrumbs[] = [
        '@type' => 'ListItem',
        'position' => 1,
        'name' => 'Home',
        'item' => rtrim(\WHMCS\Config\Setting::getValue('SystemURL'), '/')
    ];
    
    // Add page-specific breadcrumbs based on filename
    $currentUrl = getCurrentUrl();
    
    if (isset($vars['breadcrumb'])) {
        foreach ($vars['breadcrumb'] as $index => $crumb) {
            $breadcrumbs[] = [
                '@type' => 'ListItem',
                'position' => $index + 2,
                'name' => $crumb['label'],
                'item' => $crumb['url'] ?? $currentUrl
            ];
        }
    }
    
    $schema = [
        '@context' => 'https://schema.org',
        '@type' => 'BreadcrumbList',
        'itemListElement' => $breadcrumbs
    ];
    
    return ['breadcrumb_schema' => '<script type="application/ld+json">' . json_encode($schema) . '</script>'];
});
```

### Step 9: Page Speed Optimization

```php
<?php
/**
 * Critical CSS and performance optimization
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $optimization = '';
    
    // Preload critical resources
    $optimization .= '<link rel="preload" href="/templates/' . $vars['template'] . '/fonts/main.woff2" as="font" type="font/woff2" crossorigin>' . "\n";
    
    // Defer non-critical JavaScript
    $optimization .= '<script defer src="/templates/' . $vars['template'] . '/js/analytics.js"></script>' . "\n";
    
    return $optimization;
});
```

### Step 10: SSL and Security Headers

**Via .htaccess or server config:**
```
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "SAMEORIGIN"
Header always set X-XSS-Protection "1; mode=block"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
```

## Best Practices
- Use unique, descriptive title tags (under 60 characters)
- Write meta descriptions under 160 characters
- Use semantic HTML structure
- Implement structured data for rich snippets
- Create and submit XML sitemap
- Optimize for Core Web Vitals
- Use SSL/HTTPS
- Minimize duplicate content
- Set canonical URLs
- Monitor indexing with Google Search Console
