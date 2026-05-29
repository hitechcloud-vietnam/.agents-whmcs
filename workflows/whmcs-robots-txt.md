# WHMCS Robots.txt Configuration Workflow

## Purpose
Configure robots.txt file for optimal search engine crawling and indexing.

## Prerequisites
- WHMCS installation
- FTP/cPanel file access
- Basic SEO knowledge

## Step-by-Step Process

### Step 1: Create Robots.txt File

**Create /whmcs/robots.txt:**
```
# WHMCS Robots.txt Configuration
# Version: 1.0
# Last Updated: 2026-01-15

# robots.txt for WHMCS
```

### Step 2: Basic Configuration

```
# WHMCS Robots.txt
# =========================================

# Crawl-delay directive (optional)
Crawl-delay: 5

# Sitemap location
Sitemap: https://yourdomain.com/sitemap.xml

# Allow all crawlers
User-agent: *
Allow: /

# =========================================
# DISALLOW SECTIONS
# =========================================
```

### Step 3: Admin Area Restrictions

```
# Block admin directory
Disallow: /admin/
Disallow: /administrator/

# Block configuration files
Disallow: /configuration.php
Disallow: /config/
Disallow: /inc/
Disallow: /includes/
```

### Step 4: System Directories

```
# Block WHMCS system directories
Disallow: /vendor/
Disallow: /node_modules/
Disallow: /.git/
Disallow: /.svn/
Disallow: /cache/
Disallow: /logs/
Disallow: /templates_c/
```

### Step 5: API and Technical Paths

```
# Block API endpoints
Disallow: /api/
Disallow: /api.php

# Block download files
Disallow: /download/

# Block cron files
Disallow: /cron/
Disallow: /whois.php
```

### Step 6: Client-Specific Restrictions

```
# Block client area sensitive actions
Disallow: /clientarea.php?action=details
Disallow: /clientarea.php?action=security
Disallow: /clientarea.php?action=contacts
Disallow: /clientarea.php?action=password
Disallow: /clientarea.php?action=twofa

# Block billing sensitive pages
Disallow: /viewinvoice.php?*
Disallow: /offlinepayment.php
```

### Step 7: Session and Query Parameters

```
# Block sessions
Disallow: /*session*

# Block tracking parameters
Disallow: /*utm_*
Disallow: /*source=*
Disallow: /*campaign=*

# Allow specific query strings
Allow: /cart.php?a=*
Allow: /cart.php?gid=*
Allow: /domainchecker.php?*
Allow: /announcements.php?id=*
```

### Step 8: Crawler-Specific Rules

```
# Googlebot
User-agent: Googlebot
Allow: /
Disallow: /admin/
Disallow: /configuration.php
Crawl-delay: 1

# Googlebot-Image
User-agent: Googlebot-Image
Allow: /images/
Allow: /assets/img/
Disallow: /

# Bingbot
User-agent: Bingbot
Allow: /
Disallow: /admin/
Disallow: /configuration.php
Crawl-delay: 5

# Slurp (Yahoo)
User-agent: Slurp
Allow: /
Disallow: /admin/

# DuckDuckBot
User-agent: DuckDuckBot
Allow: /

# Baidu
User-agent: Baiduspider
Allow: /
Disallow: /admin/
Disallow: /api/
```

### Step 9: Dynamic PHP Hook Generation

**Create hooks/robots_txt.php:**
```php
<?php
/**
 * Dynamic robots.txt generation
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    // Only apply if requesting robots.txt
    if (strpos($_SERVER['REQUEST_URI'], 'robots.txt') !== false) {
        header('Content-Type: text/plain');
        
        $baseUrl = rtrim(\WHMCS\Config\Setting::getValue('SystemURL'), '/');
        $disallowedPaths = getDisallowedPaths();
        
        $robots = generateRobotsTxt($baseUrl, $disallowedPaths);
        
        // For caching in production, uncomment:
        // header('Cache-Control: max-age=86400');
        
        die($robots);
    }
});

function getDisallowedPaths() {
    return [
        '/admin/',
        '/configuration.php',
        '/config/',
        '/includes/',
        '/vendor/',
        '/api/',
        '/download/',
        '/cache/',
        '/logs/',
        '/templates_c/',
        '/.git/',
        '/clientarea.php?action=details',
        '/clientarea.php?action=security'
    ];
}

function generateRobotsTxt($baseUrl, $disallowedPaths) {
    $output = "# WHMCS Robots.txt\n";
    $output .= "# Generated: " . date('Y-m-d H:i:s') . "\n\n";
    
    $output .= "User-agent: *\n";
    $output .= "Allow: /\n\n";
    
    // Add disallow rules
    foreach ($disallowedPaths as $path) {
        $output .= "Disallow: " . $path . "\n";
    }
    
    $output .= "\n";
    $output .= "Sitemap: " . $baseUrl . "/sitemap.xml\n";
    
    return $output;
}
```

### Step 10: Security Considerations

```
# Block sensitive files
Disallow: /*.php$
Allow: /index.php
Allow: /cart.php
Allow: /domainchecker.php
Allow: /announcements.php
Allow: /submitticket.php
Allow: /register.php
Allow: /login.php
Allow: /contact.php
Allow: /knowledgebase.php

# Block config and credentials
Disallow: /*config*
Disallow: /*credential*
Disallow: /*secret*
Disallow: /*password*

# Block backup files
Disallow: /*.bak$
Disallow: /*.backup$
Disallow: /*.sql$
Disallow: /*.gz$

# Block version control
Disallow: /*.git*
Disallow: /*.svn*
Disallow: /*.hg*
```

### Step 11: Performance Directives

```
# Rate limiting for aggressive crawlers
User-agent: MJ12bot
Crawl-delay: 10

User-agent: AhrefsBot
Crawl-delay: 10

User-agent: SemrushBot
Crawl-delay: 10

User-agent:rogerbot
Crawl-delay: 10

User-agent: CCBot
Crawl-delay: 10
```

### Step 12: Complete Robots.txt Example

```
# WHMCS Robots.txt - Complete Configuration
# ============================================

# Crawl Delay
Crawl-delay: 5

# Sitemap
Sitemap: https://yourdomain.com/sitemap.xml

# ============================================
# USER AGENTS
# ============================================

User-agent: *
# Allow all by default
Allow: /

# Block admin and system
Disallow: /admin/
Disallow: /administrator/
Disallow: /configuration.php
Disallow: /config/
Disallow: /includes/
Disallow: /vendor/
Disallow: /api/
Disallow: /download/
Disallow: /cache/
Disallow: /logs/
Disallow: /templates_c/

# Block sensitive client areas
Disallow: /clientarea.php?action=details
Disallow: /clientarea.php?action=security
Disallow: /clientarea.php?action=contacts
Disallow: /clientarea.php?action=password
Disallow: /clientarea.php?action=twofa
Disallow: /clientarea.php?action=addfunds

# Block billing pages
Disallow: /viewinvoice.php
Disallow: /offlinepayment.php

# Block tracking parameters
Disallow: /*utm_*
Disallow: /*session*
Disallow: /*PHPSESSID*

# Block backup and temp files
Disallow: /*.bak
Disallow: /*.backup
Disallow: /*.sql
Disallow: /*.gz

# ============================================
# SPECIFIC CRAWLERS
# ============================================

User-agent: Googlebot
Allow: /
Disallow: /admin/
Crawl-delay: 1

User-agent: Googlebot-Image
Allow: /images/
Allow: /assets/img/
Allow: /products/
Disallow: /

User-agent: Googlebot-News
Allow: /announcements/
Disallow: /

User-agent: Googlebot-Video
Allow: /videos/
Disallow: /

User-agent: Bingbot
Allow: /
Disallow: /admin/
Crawl-delay: 5

User-agent: Slurp
Disallow: /

User-agent: DuckDuckBot
Allow: /

User-agent: Yandex
Allow: /
Disallow: /admin/

# ============================================
# AGGRESSIVE BOT RATE LIMITING
# ============================================

User-agent: MJ12bot
Disallow: /

User-agent: AhrefsBot
Disallow: /

User-agent: SemrushBot
Disallow: /

# ============================================
# END
# ============================================
```

### Step 13: Testing Robots.txt

```php
<?php
/**
 * Validate robots.txt syntax
 */
function validateRobotsTxt($content) {
    $errors = [];
    $lines = explode("\n", $content);
    
    foreach ($lines as $lineNum => $line) {
        $line = trim($line);
        
        // Skip comments and empty lines
        if (empty($line) || $line[0] === '#') {
            continue;
        }
        
        // Validate format
        if (!preg_match('/^(User-agent|Allow|Disallow|Crawl-delay|Sitemap):\s*/i', $line)) {
            $errors[] = "Line " . ($lineNum + 1) . ": Invalid directive";
        }
        
        // Check for wildcards in wrong places
        if (preg_match('/Disallow:.*\*\*/', $line)) {
            $errors[] = "Line " . ($lineNum + 1) . ": Double wildcard not recommended";
        }
    }
    
    return $errors;
}
```

### Step 14: WordPress-Style Robots.txt

If WHMCS is installed alongside WordPress in subdirectory:

```
# Main site robots.txt
User-agent: *
Allow: /

# WordPress in /blog/
Disallow: /blog/wp-admin/
Allow: /blog/wp-admin/admin-ajax.php

# WHMCS in /billing/
Disallow: /billing/admin/
Disallow: /billing/configuration.php
Allow: /billing/

# Common sitemaps
Sitemap: https://yourdomain.com/blog/sitemap.xml
Sitemap: https://yourdomain.com/billing/sitemap.xml
```

## Best Practices
- Always include Sitemap directive
- Test with Google Search Console
- Use specific disallow rules
- Set appropriate crawl delays
- Block sensitive directories
- Allow essential pages
- Update regularly
- Use consistent formatting
- Consider crawler-specific rules
- Monitor for accidental blocks
