# WHMCS XML Sitemap Workflow

## Purpose
Generate and maintain XML sitemaps for WHMCS to improve search engine indexing.

## Prerequisites
- WHMCS installation
- Cron access
- FTP/cPanel file access

## Step-by-Step Process

### Step 1: Create Sitemap Generator

**Create hooks/sitemap_generator.php:**
```php
<?php
/**
 * WHMCS XML Sitemap Generator
 * Generates comprehensive XML sitemap for SEO
 */

use WHMCS\Database\Capsule;
use WHMCS\Config\Setting;

function generateSitemap() {
    $baseUrl = rtrim(Setting::getValue('SystemURL'), '/');
    $today = date('Y-m-d');
    
    $urls = [];
    
    // Priority levels
    $priority = [
        'home' => 1.0,
        'page' => 0.8,
        'category' => 0.7,
        'product' => 0.9,
        'article' => 0.6,
        'static' => 0.5
    ];
    
    $changefreq = [
        'home' => 'daily',
        'page' => 'weekly',
        'category' => 'weekly',
        'product' => 'weekly',
        'article' => 'monthly',
        'static' => 'monthly'
    ];
    
    // Home page
    $urls[] = [
        'loc' => $baseUrl . '/',
        'priority' => $priority['home'],
        'changefreq' => $changefreq['home'],
        'lastmod' => $today
    ];
    
    // Static pages
    $staticPages = [
        ['url' => '/announcements/', 'priority' => 0.7, 'changefreq' => 'weekly'],
        ['url' => '/knowledgebase/', 'priority' => 0.7, 'changefreq' => 'weekly'],
        ['url' => '/serverstatus/', 'priority' => 0.6, 'changefreq' => 'daily'],
        ['url' => '/contact/', 'priority' => 0.5, 'changefreq' => 'monthly'],
        ['url' => '/affiliates/', 'priority' => 0.6, 'changefreq' => 'monthly'],
        ['url' => '/register/', 'priority' => 0.4, 'changefreq' => 'yearly'],
        ['url' => '/login/', 'priority' => 0.4, 'changefreq' => 'yearly'],
        ['url' => '/password-reset/', 'priority' => 0.3, 'changefreq' => 'yearly'],
    ];
    
    foreach ($staticPages as $page) {
        $urls[] = [
            'loc' => $baseUrl . $page['url'],
            'priority' => $page['priority'],
            'changefreq' => $page['changefreq'],
            'lastmod' => $today
        ];
    }
    
    // Product categories
    $categories = Capsule::table('tblproductgroups')
        ->where('hidden', 0)
        ->get(['id', 'name', 'slug', 'updated_at']);
    
    foreach ($categories as $cat) {
        $slug = $cat->slug ?? slugify($cat->name);
        $lastmod = $cat->updated_at ? date('Y-m-d', strtotime($cat->updated_at)) : $today;
        
        $urls[] = [
            'loc' => $baseUrl . '/store/' . $slug,
            'priority' => $priority['category'],
            'changefreq' => $changefreq['category'],
            'lastmod' => $lastmod
        ];
    }
    
    // Products
    $products = Capsule::table('tblproducts')
        ->join('tblproductgroups', 'tblproducts.gid', '=', 'tblproductgroups.id')
        ->where('tblproducts.hidden', 0)
        ->where('tblproducts.showorder', 1)
        ->where('tblproductgroups.hidden', 0)
        ->get([
            'tblproducts.id', 
            'tblproducts.name', 
            'tblproducts.slug',
            'tblproducts.updated_at'
        ]);
    
    foreach ($products as $product) {
        $slug = $product->slug ?? slugify($product->name);
        $lastmod = $product->updated_at ? date('Y-m-d', strtotime($product->updated_at)) : $today;
        
        $urls[] = [
            'loc' => $baseUrl . '/store/' . $slug . '-' . $product->id,
            'priority' => $priority['product'],
            'changefreq' => $changefreq['product'],
            'lastmod' => $lastmod,
            'images' => getProductImages($product->id)
        ];
    }
    
    // Announcements
    $announcements = Capsule::table('tblannouncements')
        ->where('published', 1)
        ->orderBy('created_at', 'desc')
        ->limit(100)
        ->get(['id', 'title', 'created_at', 'updated_at']);
    
    foreach ($announcements as $ann) {
        $slug = slugify($ann->title);
        $lastmod = $ann->updated_at ? date('Y-m-d', strtotime($ann->updated_at)) : date('Y-m-d', strtotime($ann->created_at));
        
        $urls[] = [
            'loc' => $baseUrl . '/announcements/' . $ann->id . '/' . $slug,
            'priority' => $priority['article'],
            'changefreq' => $changefreq['article'],
            'lastmod' => $lastmod
        ];
    }
    
    // Knowledgebase categories
    $kbCategories = Capsule::table('tblknowledgebaselcategories')
        ->where('hidden', 0)
        ->get(['id', 'name']);
    
    foreach ($kbCategories as $cat) {
        $slug = slugify($cat->name);
        $urls[] = [
            'loc' => $baseUrl . '/knowledgebase/' . $cat->id . '/' . $slug,
            'priority' => 0.6,
            'changefreq' => 'weekly',
            'lastmod' => $today
        ];
    }
    
    // Knowledgebase articles
    $kbArticles = Capsule::table('tblknowledgebase')
        ->where('published', 1)
        ->where('noeditor', 0)
        ->get(['id', 'title', 'category', 'created', 'modified']);
    
    foreach ($kbArticles as $article) {
        $slug = slugify($article->title);
        $lastmod = $article->modified ? date('Y-m-d', strtotime($article->modified)) : date('Y-m-d', strtotime($article->created));
        
        $urls[] = [
            'loc' => $baseUrl . '/knowledgebase/' . $article->id . '/' . $slug,
            'priority' => $priority['article'],
            'changefreq' => $changefreq['article'],
            'lastmod' => $lastmod
        ];
    }
    
    // Network issues (if applicable)
    $networkIssues = Capsule::table('tblnetworkissues')
        ->where('status', '!=', 'resolved')
        ->get(['id', 'title']);
    
    foreach ($networkIssues as $issue) {
        $urls[] = [
            'loc' => $baseUrl . '/serverstatus/#issue-' . $issue->id,
            'priority' => 0.5,
            'changefreq' => 'daily',
            'lastmod' => $today
        ];
    }
    
    return $urls;
}

function getProductImages($productId) {
    $images = [];
    
    // Get product images from database or filesystem
    $productImages = Capsule::table('tblproductfiles')
        ->where('productid', $productId)
        ->where('type', 'thumbnail')
        ->get(['filename']);
    
    foreach ($productImages as $img) {
        $images[] = [
            'loc' => \WHMCS\Config\Setting::getValue('SystemURL') . '/products/' . $img->filename,
            'title' => 'Product Image'
        ];
    }
    
    return $images;
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

### Step 2: Generate XML Output

```php
<?php
/**
 * Generate XML sitemap file
 */
function generateSitemapXml($urls) {
    $baseUrl = rtrim(\WHMCS\Config\Setting::getValue('SystemURL'), '/');
    
    $xml = '<?xml version="1.0" encoding="UTF-8"?>' . "\n";
    $xml .= '<?xml-stylesheet type="text/xsl" href="' . $baseUrl . '/sitemap.xsl"?>' . "\n";
    $xml .= '<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"' . "\n";
    $xml .= '        xmlns:image="http://www.google.com/schemas/sitemap-image/1.1">' . "\n";
    
    foreach ($urls as $url) {
        $xml .= '    <url>' . "\n";
        $xml .= '        <loc>' . htmlspecialchars($url['loc']) . '</loc>' . "\n";
        $xml .= '        <lastmod>' . $url['lastmod'] . '</lastmod>' . "\n";
        $xml .= '        <changefreq>' . $url['changefreq'] . '</changefreq>' . "\n";
        $xml .= '        <priority>' . $url['priority'] . '</priority>' . "\n";
        
        // Add image URLs if present
        if (!empty($url['images'])) {
            foreach ($url['images'] as $image) {
                $xml .= '        <image:image>' . "\n";
                $xml .= '            <image:loc>' . htmlspecialchars($image['loc']) . '</image:loc>' . "\n";
                if (!empty($image['title'])) {
                    $xml .= '            <image:title>' . htmlspecialchars($image['title']) . '</image:title>' . "\n";
                }
                $xml .= '        </image:image>' . "\n";
            }
        }
        
        $xml .= '    </url>' . "\n";
    }
    
    $xml .= '</urlset>';
    
    return $xml;
}
```

### Step 3: Save Sitemap File

```php
<?php
/**
 * Save sitemap to file
 */
function saveSitemap() {
    $urls = generateSitemap();
    $xml = generateSitemapXml($urls);
    
    $sitemapPath = ROOTDIR . '/sitemap.xml';
    $result = file_put_contents($sitemapPath, $xml);
    
    if ($result !== false) {
        logActivity('Sitemap generated successfully: ' . count($urls) . ' URLs');
        return ['success' => true, 'url_count' => count($urls), 'file_size' => $result];
    } else {
        logActivity('Failed to generate sitemap');
        return ['success' => false, 'error' => 'Failed to write sitemap file'];
    }
}
```

### Step 4: Set Up Cron Job

```php
<?php
/**
 * Hook for daily sitemap regeneration
 */
add_hook('DailyCronJob', 1, function($vars) {
    $result = saveSitemap();
    
    if ($result['success']) {
        logActivity("Daily sitemap update: {$result['url_count']} URLs, {$result['file_size']} bytes");
    }
    
    return $result;
});
```

### Step 5: Create Sitemap Index

```php
<?php
/**
 * Generate sitemap index file (for large sites)
 */
function generateSitemapIndex($sitemaps) {
    $xml = '<?xml version="1.0" encoding="UTF-8"?>' . "\n";
    $xml .= '<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">' . "\n";
    
    foreach ($sitemaps as $sitemap) {
        $xml .= '    <sitemap>' . "\n";
        $xml .= '        <loc>' . htmlspecialchars($sitemap['loc']) . '</loc>' . "\n";
        $xml .= '        <lastmod>' . $sitemap['lastmod'] . '</lastmod>' . "\n";
        $xml .= '    </sitemap>' . "\n";
    }
    
    $xml .= '</sitemapindex>';
    
    return $xml;
}

/**
 * Generate multiple sitemaps for large sites
 */
function generateMultipleSitemaps() {
    $baseUrl = rtrim(\WHMCS\Config\Setting::getValue('SystemURL'), '/');
    $maxUrlsPerSitemap = 1000;
    
    $urls = generateSitemap();
    $chunks = array_chunk($urls, $maxUrlsPerSitemap);
    
    $sitemaps = [];
    
    foreach ($chunks as $index => $chunk) {
        $filename = 'sitemap-' . ($index + 1) . '.xml';
        $xml = generateSitemapXml($chunk);
        
        $filepath = ROOTDIR . '/' . $filename;
        file_put_contents($filepath, $xml);
        
        $sitemaps[] = [
            'loc' => $baseUrl . '/' . $filename,
            'lastmod' => date('Y-m-d')
        ];
    }
    
    // Generate sitemap index
    $indexXml = generateSitemapIndex($sitemaps);
    file_put_contents(ROOTDIR . '/sitemap.xml', $indexXml);
    
    return $sitemaps;
}
```

### Step 6: Create XSL Stylesheet

**Create /whmcs/sitemap.xsl:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="2.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:sitemap="http://www.sitemaps.org/schemas/sitemap/0.9"
    xmlns:image="http://www.google.com/schemas/sitemap-image/1.1">
    
    <xsl:output method="html" version="1.0" encoding="UTF-8" indent="yes"/>
    
    <xsl:template match="/">
        <html>
            <head>
                <title>XML Sitemap - <xsl:value-of select="sitemap:urlset/sitemap:url[1]/sitemap:loc"/></title>
                <meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
                <style>
                    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; max-width: 1200px; margin: 0 auto; padding: 20px; }
                    h1 { color: #1a1a1a; }
                    table { width: 100%; border-collapse: collapse; }
                    th, td { padding: 10px; text-align: left; border-bottom: 1px solid #eee; }
                    th { background-color: #f8f9fa; }
                    tr:hover { background-color: #f8f9fa; }
                    a { color: #007bff; text-decoration: none; }
                    a:hover { text-decoration: underline; }
                    .priority-high { color: #28a745; }
                    .priority-medium { color: #ffc107; }
                    .priority-low { color: #6c757d; }
                </style>
            </head>
            <body>
                <h1>XML Sitemap</h1>
                <p>This XML sitemap contains <xsl:value-of select="count(sitemap:urlset/sitemap:url)"/> URLs.</p>
                <table>
                    <thead>
                        <tr>
                            <th>URL</th>
                            <th>Priority</th>
                            <th>Change Frequency</th>
                            <th>Last Modified</th>
                        </tr>
                    </thead>
                    <tbody>
                        <xsl:for-each select="sitemap:urlset/sitemap:url">
                            <tr>
                                <td>
                                    <a href="{sitemap:loc}">
                                        <xsl:value-of select="sitemap:loc"/>
                                    </a>
                                </td>
                                <td>
                                    <xsl:attribute name="class">
                                        <xsl:choose>
                                            <xsl:when test="sitemap:priority >= 0.8">priority-high</xsl:when>
                                            <xsl:when test="sitemap:priority >= 0.5">priority-medium</xsl:when>
                                            <xsl:otherwise>priority-low</xsl:otherwise>
                                        </xsl:choose>
                                    </xsl:attribute>
                                    <xsl:value-of select="sitemap:priority"/>
                                </td>
                                <td><xsl:value-of select="sitemap:changefreq"/></td>
                                <td><xsl:value-of select="sitemap:lastmod"/></td>
                            </tr>
                        </xsl:for-each>
                    </tbody>
                </table>
            </body>
        </html>
    </xsl:template>
</xsl:stylesheet>
```

### Step 7: Add Robots.txt Entry

```php
<?php
/**
 * Ensure robots.txt includes sitemap reference
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    // This would typically be handled by the web server or a static robots.txt
    // But we can add a note here
    return '';
});
```

Add to /whmcs/robots.txt:
```
Sitemap: https://yourdomain.com/sitemap.xml
```

### Step 8: Video Sitemap (Optional)

```php
<?php
/**
 * Generate video sitemap
 */
function generateVideoSitemap($videos) {
    $xml = '<?xml version="1.0" encoding="UTF-8"?>' . "\n";
    $xml .= '<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
                  xmlns:video="http://www.google.com/schemas/sitemap-video/1.1">' . "\n";
    
    foreach ($videos as $video) {
        $xml .= '    <url>' . "\n";
        $xml .= '        <loc>' . htmlspecialchars($video['url']) . '</loc>' . "\n";
        $xml .= '        <video:video>' . "\n";
        $xml .= '            <video:title>' . htmlspecialchars($video['title']) . '</video:title>' . "\n";
        $xml .= '            <video:description>' . htmlspecialchars($video['description']) . '</video:description>' . "\n";
        $xml .= '            <video:content_loc>' . htmlspecialchars($video['video_url']) . '</video:content_loc>' . "\n";
        $xml .= '            <video:thumbnail_loc>' . htmlspecialchars($video['thumbnail']) . '</video:thumbnail_loc>' . "\n";
        $xml .= '            <video:duration>' . $video['duration'] . '</video:duration>' . "\n";
        $xml .= '            <video:publication_date>' . $video['published'] . '</video:publication_date>' . "\n";
        $xml .= '        </video:video>' . "\n";
        $xml .= '    </url>' . "\n";
    }
    
    $xml .= '</urlset>';
    
    return $xml;
}
```

### Step 9: News Sitemap (Optional)

```php
<?php
/**
 * Generate Google News sitemap
 */
function generateNewsSitemap($articles) {
    $xml = '<?xml version="1.0" encoding="UTF-8"?>' . "\n";
    $xml .= '<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
                  xmlns:news="http://www.google.com/schemas/sitemap-news/0.9">' . "\n";
    
    foreach ($articles as $article) {
        $xml .= '    <url>' . "\n";
        $xml .= '        <loc>' . htmlspecialchars($article['url']) . '</loc>' . "\n";
        $xml .= '        <news:news>' . "\n";
        $xml .= '            <news:publication>' . "\n";
        $xml .= '                <news:name>' . htmlspecialchars($article['publication_name']) . '</news:name>' . "\n";
        $xml .= '                <news:language>en</news:language>' . "\n";
        $xml .= '            </news:publication>' . "\n";
        $xml .= '            <news:publication_date>' . $article['date'] . '</news:publication_date>' . "\n";
        $xml .= '            <news:title>' . htmlspecialchars($article['title']) . '</news:title>' . "\n";
        $xml .= '        </news:news>' . "\n";
        $xml .= '    </url>' . "\n";
    }
    
    $xml .= '</urlset>';
    
    return $xml;
}
```

## Best Practices
- Update sitemap regularly (daily or weekly)
- Limit URLs per sitemap to 50,000
- Limit sitemap file size to 50MB
- Use compression for large sitemaps
- Submit sitemap to search engines
- Monitor indexing in Search Console
- Include only canonical URLs
- Set appropriate priorities
- Use correct change frequencies
- Consider using sitemap index for large sites
