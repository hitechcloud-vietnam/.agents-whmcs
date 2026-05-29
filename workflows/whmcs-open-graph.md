# WHMCS Open Graph Setup Workflow

## Purpose
Configure Open Graph meta tags in WHMCS for optimal social media sharing.

## Prerequisites
- WHMCS installation
- Template access
- Social media presence
- Basic SEO knowledge

## Step-by-Step Process

### Step 1: Create Open Graph Hook

**Create hooks/open_graph.php:**
```php
<?php
/**
 * WHMCS Open Graph Meta Tags
 * Optimized for Facebook, LinkedIn, WhatsApp, and more
 */

use WHMCS\Config\Setting;

function getOGConfig() {
    return [
        'site_name' => Setting::getValue('CompanyName') ?: 'Your Company',
        'site_url' => rtrim(Setting::getValue('SystemURL'), '/'),
        'default_image' => Setting::getValue('SystemURL') . '/assets/img/og-default.jpg',
        'locale' => 'en_US',
        'fb_app_id' => Setting::getValue('FacebookAppId'),
        'twitter_handle' => Setting::getValue('TwitterUsername')
    ];
}
```

### Step 2: Core Open Graph Tags

```php
<?php
/**
 * Core Open Graph implementation
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $config = getOGConfig();
    $og = '';
    
    // Get page-specific configuration
    $pageOG = getPageOGConfig($vars, $config);
    
    // Required OG tags
    $og .= '<meta property="og:title" content="' . htmlspecialchars($pageOG['title']) . '">' . "\n";
    $og .= '<meta property="og:description" content="' . htmlspecialchars($pageOG['description']) . '">' . "\n";
    $og .= '<meta property="og:type" content="' . ($pageOG['type'] ?? 'website') . '">' . "\n";
    $og .= '<meta property="og:url" content="' . htmlspecialchars(getCurrentUrl()) . '">' . "\n";
    $og .= '<meta property="og:site_name" content="' . htmlspecialchars($config['site_name']) . '">' . "\n";
    $og .= '<meta property="og:locale" content="' . $config['locale'] . '">' . "\n";
    
    // Image (required for rich media)
    $imageUrl = $pageOG['image'] ?? $config['default_image'];
    $og .= '<meta property="og:image" content="' . htmlspecialchars($imageUrl) . '">' . "\n";
    $og .= '<meta property="og:image:secure_url" content="' . htmlspecialchars(str_replace('http://', 'https://', $imageUrl)) . '">' . "\n";
    $og .= '<meta property="og:image:type" content="image/jpeg">' . "\n";
    $og .= '<meta property="og:image:width" content="1200">' . "\n";
    $og .= '<meta property="og:image:height" content="630">' . "\n";
    $og .= '<meta property="og:image:alt" content="' . htmlspecialchars($pageOG['title']) . '">' . "\n";
    
    // Optional: Facebook App ID
    if (!empty($config['fb_app_id'])) {
        $og .= '<meta property="fb:app_id" content="' . htmlspecialchars($config['fb_app_id']) . '">' . "\n";
    }
    
    // Optional: Article-specific tags
    if (isset($pageOG['article'])) {
        $article = $pageOG['article'];
        if (!empty($article['published_time'])) {
            $og .= '<meta property="article:published_time" content="' . $article['published_time'] . '">' . "\n";
        }
        if (!empty($article['modified_time'])) {
            $og .= '<meta property="article:modified_time" content="' . $article['modified_time'] . '">' . "\n";
        }
        if (!empty($article['author'])) {
            $og .= '<meta property="article:author" content="' . htmlspecialchars($article['author']) . '">' . "\n";
        }
        if (!empty($article['section'])) {
            $og .= '<meta property="article:section" content="' . htmlspecialchars($article['section']) . '">' . "\n";
        }
        if (!empty($article['tags'])) {
            foreach ($article['tags'] as $tag) {
                $og .= '<meta property="article:tag" content="' . htmlspecialchars($tag) . '">' . "\n";
            }
        }
    }
    
    return $og;
});

function getCurrentUrl() {
    $protocol = (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off') ? 'https' : 'http';
    return $protocol . '://' . ($_SERVER['HTTP_HOST'] ?? '') . ($_SERVER['REQUEST_URI'] ?? '/');
}
```

### Step 3: Page-Specific OG Configuration

```php
<?php
/**
 * Get Open Graph configuration per page type
 */
function getPageOGConfig($vars, $config) {
    $default = [
        'title' => $config['site_name'] . ' - Professional Hosting Services',
        'description' => 'Reliable web hosting, VPS, and cloud solutions with 99.9% uptime.',
        'type' => 'website',
        'image' => $config['default_image']
    ];
    
    $filename = $vars['filename'] ?? '';
    
    switch ($filename) {
        case '':
        case 'index':
            return [
                'title' => $config['site_name'] . ' - Premium Web Hosting & Cloud Solutions',
                'description' => 'Get started with reliable web hosting, VPS, cloud servers, and domain registration. 99.9% uptime guarantee.',
                'type' => 'website',
                'image' => $config['site_url'] . '/images/og-homepage.jpg'
            ];
            
        case 'announcement':
        case 'announcements':
            if (isset($vars['announcement'])) {
                return [
                    'title' => $vars['announcement']['title'] ?? 'Announcement',
                    'description' => $vars['announcement']['summary'] ?? '',
                    'type' => 'article',
                    'image' => $config['site_url'] . '/images/og-announcement.jpg',
                    'article' => [
                        'published_time' => ($vars['announcement']['date'] ?? date('Y-m-d')) . 'T00:00:00+00:00',
                        'section' => 'Announcements',
                        'author' => $config['site_name']
                    ]
                ];
            }
            break;
            
        case 'knowledgebase':
            if (isset($vars['kbarticle'])) {
                return [
                    'title' => $vars['kbarticle']['title'] ?? 'Help Article',
                    'description' => substr(strip_tags($vars['kbarticle']['article'] ?? ''), 0, 160),
                    'type' => 'article',
                    'image' => $config['site_url'] . '/images/og-knowledgebase.jpg'
                ];
            }
            return [
                'title' => 'Knowledge Base - Help & Documentation | ' . $config['site_name'],
                'description' => 'Find helpful guides and documentation for our hosting services.',
                'type' => 'website'
            ];
            
        case 'clientarea':
            if (isset($vars['service'])) {
                return [
                    'title' => 'My Service: ' . ($vars['service']['domain'] ?? 'Hosting'),
                    'description' => 'Manage your hosting service in the client area.',
                    'type' => 'profile',
                    'image' => $config['site_url'] . '/images/og-service.jpg'
                ];
            }
            break;
            
        case 'cart':
            return [
                'title' => 'Complete Your Order | ' . $config['site_name'],
                'description' => 'Review your cart and complete your purchase securely.',
                'type' => 'website'
            ];
            
        case 'domainchecker':
            return [
                'title' => 'Domain Name Search | ' . $config['site_name'],
                'description' => 'Find and register your perfect domain name. Great prices and free DNS.',
                'type' => 'website',
                'image' => $config['site_url'] . '/images/og-domains.jpg'
            ];
            
        case 'affiliates':
            return [
                'title' => 'Earn Money with Our Affiliate Program | ' . $config['site_name'],
                'description' => 'Join our affiliate program and earn commission for every customer you refer.',
                'type' => 'website',
                'image' => $config['site_url'] . '/images/og-affiliates.jpg'
            ];
    }
    
    // Check for product configuration
    if (isset($vars['productinfo']) && is_array($vars['productinfo'])) {
        $product = $vars['productinfo'];
        return [
            'title' => $product['name'] . ' - Order Now | ' . $config['site_name'],
            'description' => substr(strip_tags($product['description'] ?? ''), 0, 160),
            'type' => 'product',
            'image' => !empty($product['image']) ? $config['site_url'] . '/' . $product['image'] : $config['default_image'],
            'product' => [
                'price' => $product['pricing']['monthly']['price'] ?? '0.00',
                'currency' => 'USD'
            ]
        ];
    }
    
    return $default;
}
```

### Step 4: Product OG Tags

```php
<?php
/**
 * Open Graph for product pages
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $og = '';
    
    if (isset($vars['productinfo']) && is_array($vars['productinfo'])) {
        $product = $vars['productinfo'];
        $config = getOGConfig();
        
        // Product group
        if (!empty($product['gid'])) {
            $og .= '<meta property="og:product:product:retailer_item_id" content="' . $product['id'] . '">' . "\n";
        }
        
        // Availability
        $og .= '<meta property="og:product:availability" content="instock">' . "\n";
        
        // Condition
        $og .= '<meta property="og:product:condition" content="new">' . "\n";
    }
    
    return ['product_og' => $og];
});
```

### Step 5: Profile OG Tags (Client Area)

```php
<?php
/**
 * Open Graph for user profile pages
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $og = '';
    
    if ($vars['loggedin'] && isset($vars['client'])) {
        $client = $vars['client'];
        
        // Profile metadata
        $og .= '<meta property="og:profile:first_name" content="' . htmlspecialchars($client['firstname'] ?? '') . '">' . "\n";
        $og .= '<meta property="og:profile:last_name" content="' . htmlspecialchars($client['lastname'] ?? '') . '">' . "\n";
        $og .= '<meta property="og:profile:username" content="' . htmlspecialchars($client['email'] ?? '') . '">' . "\n";
    }
    
    return ['profile_og' => $og];
});
```

### Step 6: Video OG Tags

```php
<?php
/**
 * Open Graph video metadata for tutorial pages
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $og = '';
    
    // If page has associated video
    if (isset($vars['video_url'])) {
        $videoUrl = $vars['video_url'];
        $videoType = $vars['video_type'] ?? 'video/mp4';
        $videoWidth = $vars['video_width'] ?? 1280;
        $videoHeight = $vars['video_height'] ?? 720;
        
        $og .= '<meta property="og:video" content="' . htmlspecialchars($videoUrl) . '">' . "\n";
        $og .= '<meta property="og:video:type" content="' . htmlspecialchars($videoType) . '">' . "\n";
        $og .= '<meta property="og:video:width" content="' . $videoWidth . '">' . "\n";
        $og .= '<meta property="og:video:height" content="' . $videoHeight . '">' . "\n";
        $og .= '<meta property="og:video:secure_url" content="' . htmlspecialchars(str_replace('http://', 'https://', $videoUrl)) . '">' . "\n";
        
        // Optional: Video thumbnail
        if (isset($vars['video_image'])) {
            $og .= '<meta property="og:image" content="' . htmlspecialchars($vars['video_image']) . '">' . "\n";
        }
    }
    
    return ['video_og' => $og];
});
```

### Step 7: Music OG Tags

```php
<?php
/**
 * Open Graph music metadata (for music-related services)
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $og = '';
    
    if (isset($vars['music_track'])) {
        $track = $vars['music_track'];
        
        $og .= '<meta property="og:type" content="music.song">' . "\n";
        $og .= '<meta property="music:duration" content="' . ($track['duration'] ?? 0) . '">' . "\n";
        $og .= '<meta property="music:album" content="' . htmlspecialchars($track['album'] ?? '') . '">' . "\n";
        
        if (isset($track['musician'])) {
            $og .= '<meta property="music:musician" content="' . htmlspecialchars($track['musician']) . '">' . "\n";
        }
    }
    
    return ['music_og' => $og];
});
```

### Step 8: Facebook Debug Tool Integration

```php
<?php
/**
 * Utility to log OG debug info
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    // When sharing a product, log for debugging
    if ($vars['productId']) {
        logActivity('OG Debug - Product ID: ' . $vars['productId'] . ' - Shared at: ' . date('Y-m-d H:i:s'));
    }
});
```

### Step 9: Default OG Image Guidelines

```php
<?php
/**
 * OG Image validation helper
 */
function validateOGImage($imagePath) {
    $requiredWidth = 1200;
    $requiredHeight = 630;
    $minWidth = 600;
    $minHeight = 314;
    
    if (!file_exists($imagePath)) {
        return ['valid' => false, 'error' => 'Image file not found'];
    }
    
    $imageInfo = @getimagesize($imagePath);
    if (!$imageInfo) {
        return ['valid' => false, 'error' => 'Invalid image file'];
    }
    
    $width = $imageInfo[0];
    $height = $imageInfo[1];
    
    // Check minimum dimensions
    if ($width < $minWidth || $height < $minHeight) {
        return [
            'valid' => false,
            'error' => "Image too small. Minimum: {$minWidth}x{$minHeight}, Current: {$width}x{$height}"
        ];
    }
    
    // Check aspect ratio (should be roughly 1.91:1)
    $aspectRatio = $width / $height;
    $expectedRatio = 1.91;
    $tolerance = 0.1;
    
    if (abs($aspectRatio - $expectedRatio) > $tolerance) {
        return [
            'valid' => true,
            'warning' => "Image aspect ratio is {$aspectRatio}, recommended is 1.91:1"
        ];
    }
    
    return ['valid' => true, 'dimensions' => ['width' => $width, 'height' => $height]];
}
```

## Best Practices
- Always include og:image (required for rich previews)
- Use 1200x630px images for optimal display
- Maintain 1.91:1 aspect ratio for images
- Use HTTPS for all OG URLs and images
- Include og:locale for proper language
- Test with Facebook Debugger tool
- Don't use relative URLs
- Keep descriptions between 40-80 characters
- Use unique images per page type
- Include alt text for accessibility
