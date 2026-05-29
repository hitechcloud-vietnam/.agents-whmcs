# WHMCS Twitter Cards Configuration Workflow

## Purpose
Configure Twitter Card meta tags in WHMCS for optimal Twitter/X sharing.

## Prerequisites
- WHMCS installation
- Template access
- Twitter Developer account (optional for cards validation)

## Step-by-Step Process

### Step 1: Create Twitter Cards Hook

**Create hooks/twitter_cards.php:**
```php
<?php
/**
 * WHMCS Twitter Card Meta Tags
 * Optimized for Twitter/X sharing
 */

use WHMCS\Config\Setting;

function getTwitterConfig() {
    return [
        'site' => '@' . (Setting::getValue('TwitterUsername') ?: 'yourtwitter'),
        'creator' => '@' . (Setting::getValue('TwitterCreator') ?: 'yourtwitter'),
        'site_url' => rtrim(Setting::getValue('SystemURL'), '/'),
        'company_name' => Setting::getValue('CompanyName') ?: 'Your Company',
        'default_image' => Setting::getValue('SystemURL') . '/assets/img/twitter-default.jpg'
    ];
}
```

### Step 2: Core Twitter Card Tags

```php
<?php
/**
 * Twitter Card implementation
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $config = getTwitterConfig();
    $twitter = '';
    
    // Get page configuration
    $pageConfig = getPageTwitterConfig($vars, $config);
    
    // Card type (summary, summary_large_image, app, player)
    $cardType = $pageConfig['card'] ?? 'summary_large_image';
    $twitter .= '<meta name="twitter:card" content="' . $cardType . '">' . "\n";
    
    // Site username
    $twitter .= '<meta name="twitter:site" content="' . htmlspecialchars($config['site']) . '">' . "\n";
    
    // Creator (if different from site)
    if (!empty($pageConfig['creator'])) {
        $twitter .= '<meta name="twitter:creator" content="' . htmlspecialchars($pageConfig['creator']) . '">' . "\n";
    }
    
    // Title (max 70 characters)
    $title = substr($pageConfig['title'], 0, 70);
    $twitter .= '<meta name="twitter:title" content="' . htmlspecialchars($title) . '">' . "\n";
    
    // Description (max 200 characters)
    $description = substr($pageConfig['description'], 0, 200);
    $twitter .= '<meta name="twitter:description" content="' . htmlspecialchars($description) . '">' . "\n";
    
    // Image URL
    $imageUrl = $pageConfig['image'] ?? $config['default_image'];
    $twitter .= '<meta name="twitter:image" content="' . htmlspecialchars($imageUrl) . '">' . "\n";
    
    // Image alt text
    $twitter .= '<meta name="twitter:image:alt" content="' . htmlspecialchars($title) . '">' . "\n";
    
    // App links (for mobile app promotion)
    if (isset($pageConfig['app'])) {
        $app = $pageConfig['app'];
        if (isset($app['iphone'])) {
            $twitter .= '<meta name="twitter:app:iphone:name" content="' . htmlspecialchars($app['iphone']['name'] ?? '') . '">' . "\n";
            $twitter .= '<meta name="twitter:app:iphone:id" content="' . htmlspecialchars($app['iphone']['id'] ?? '') . '">' . "\n";
            $twitter .= '<meta name="twitter:app:iphone:url" content="' . htmlspecialchars($app['iphone']['url'] ?? '') . '">' . "\n";
        }
        if (isset($app['ipad'])) {
            $twitter .= '<meta name="twitter:app:ipad:name" content="' . htmlspecialchars($app['ipad']['name'] ?? '') . '">' . "\n";
            $twitter .= '<meta name="twitter:app:ipad:id" content="' . htmlspecialchars($app['ipad']['id'] ?? '') . '">' . "\n";
            $twitter .= '<meta name="twitter:app:ipad:url" content="' . htmlspecialchars($app['ipad']['url'] ?? '') . '">' . "\n";
        }
        if (isset($app['googleplay'])) {
            $twitter .= '<meta name="twitter:app:googleplay:name" content="' . htmlspecialchars($app['googleplay']['name'] ?? '') . '">' . "\n";
            $twitter .= '<meta name="twitter:app:googleplay:id" content="' . htmlspecialchars($app['googleplay']['id'] ?? '') . '">' . "\n";
            $twitter .= '<meta name="twitter:app:googleplay:url" content="' . htmlspecialchars($app['googleplay']['url'] ?? '') . '">' . "\n";
        }
    }
    
    return $twitter;
});
```

### Step 3: Page-Specific Twitter Configuration

```php
<?php
/**
 * Get Twitter Card configuration per page type
 */
function getPageTwitterConfig($vars, $config) {
    $filename = $vars['filename'] ?? '';
    
    switch ($filename) {
        case '':
        case 'index':
            return [
                'card' => 'summary_large_image',
                'title' => $config['company_name'] . ' - Premium Web Hosting',
                'description' => 'Reliable web hosting, VPS, and cloud solutions. 99.9% uptime guarantee.',
                'image' => $config['site_url'] . '/images/twitter-homepage.jpg'
            ];
            
        case 'announcements':
            if (isset($vars['announcement'])) {
                return [
                    'card' => 'summary_large_image',
                    'title' => $vars['announcement']['title'] ?? 'News',
                    'description' => substr(strip_tags($vars['announcement']['message'] ?? ''), 0, 200),
                    'image' => $config['site_url'] . '/images/twitter-announcement.jpg'
                ];
            }
            break;
            
        case 'knowledgebase':
            if (isset($vars['kbarticle'])) {
                return [
                    'card' => 'summary',
                    'title' => $vars['kbarticle']['title'] ?? 'Help Article',
                    'description' => substr(strip_tags($vars['kbarticle']['article'] ?? ''), 0, 200)
                ];
            }
            return [
                'card' => 'summary',
                'title' => 'Knowledge Base - ' . $config['company_name'],
                'description' => 'Find helpful guides and documentation for our services.'
            ];
            
        case 'domainchecker':
            return [
                'card' => 'summary_large_image',
                'title' => 'Domain Name Search - ' . $config['company_name'],
                'description' => 'Find and register your perfect domain name.',
                'image' => $config['site_url'] . '/images/twitter-domains.jpg'
            ];
            
        case 'affiliates':
            return [
                'card' => 'summary_large_image',
                'title' => 'Earn Money with Our Affiliate Program',
                'description' => 'Join our affiliate program and earn commission for every customer.',
                'image' => $config['site_url'] . '/images/twitter-affiliates.jpg'
            ];
            
        case 'contact':
            return [
                'card' => 'summary',
                'title' => 'Contact Us - ' . $config['company_name'],
                'description' => 'Get in touch with our support team. Available 24/7.'
            ];
            
        case 'cart':
            return [
                'card' => 'summary',
                'title' => 'Complete Your Order - ' . $config['company_name'],
                'description' => 'Review your cart and complete your purchase securely.'
            ];
    }
    
    // Product pages
    if (isset($vars['productinfo']) && is_array($vars['productinfo'])) {
        $product = $vars['productinfo'];
        return [
            'card' => 'summary_large_image',
            'title' => $product['name'] . ' - Order Now',
            'description' => substr(strip_tags($product['description'] ?? ''), 0, 200),
            'image' => !empty($product['image']) ? $config['site_url'] . '/' . $product['image'] : $config['default_image']
        ];
    }
    
    // Default
    return [
        'card' => 'summary',
        'title' => $config['company_name'],
        'description' => 'Professional hosting services and solutions.'
    ];
}
```

### Step 4: Twitter Card Validation

```php
<?php
/**
 * Twitter Card validation utility
 */
function validateTwitterCard($config) {
    $errors = [];
    
    // Check card type
    $validCards = ['summary', 'summary_large_image', 'app', 'player'];
    if (!in_array($config['card'], $validCards)) {
        $errors[] = 'Invalid card type';
    }
    
    // Check title length
    if (strlen($config['title']) > 70) {
        $errors[] = 'Title exceeds 70 characters';
    }
    
    // Check description length
    if (strlen($config['description']) > 200) {
        $errors[] = 'Description exceeds 200 characters';
    }
    
    // Check image dimensions
    if (!empty($config['image'])) {
        $imageInfo = @getimagesize($config['image']);
        if ($imageInfo) {
            $width = $imageInfo[0];
            $height = $imageInfo[1];
            
            if ($config['card'] === 'summary_large_image') {
                if ($width < 300 || $height < 157) {
                    $errors[] = 'Image too small for summary_large_image card (min 300x157)';
                }
            } else {
                if ($width < 120 || $height < 120) {
                    $errors[] = 'Image too small for summary card (min 120x120)';
                }
            }
        }
    }
    
    return $errors;
}
```

### Step 5: Player Card (Video Content)

```php
<?php
/**
 * Twitter Player Card for video content
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $twitter = '';
    
    // Check for video content
    if (isset($vars['video']) && is_array($vars['video'])) {
        $video = $vars['video'];
        
        // Player card requires HTTPS
        $protocol = 'https';
        $videoUrl = str_replace('http://', 'https://', $video['url']);
        
        $twitter .= '<meta name="twitter:card" content="player">' . "\n";
        $twitter .= '<meta name="twitter:player" content="' . htmlspecialchars($video['player_url']) . '">' . "\n";
        $twitter .= '<meta name="twitter:player:width" content="' . ($video['width'] ?? 1280) . '">' . "\n";
        $twitter .= '<meta name="twitter:player:height" content="' . ($video['height'] ?? 720) . '">' . "\n";
        $twitter .= '<meta name="twitter:player:stream" content="' . htmlspecialchars($videoUrl) . '">' . "\n";
        $twitter .= '<meta name="twitter:player:stream:content_type" content="' . ($video['type'] ?? 'video/mp4') . '">' . "\n";
    }
    
    return ['twitter_player' => $twitter];
});
```

### Step 6: App Card (Mobile App Promotion)

```php
<?php
/**
 * Twitter App Card for mobile app promotion
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $twitter = '';
    
    // If displaying app download page
    if ($vars['filename'] === 'apps' || isset($vars['mobile_app'])) {
        $twitter .= '<meta name="twitter:card" content="app">' . "\n";
        $twitter .= '<meta name="twitter:app:name:iphone" content="My App">' . "\n";
        $twitter .= '<meta name="twitter:app:id:iphone" content="123456789">' . "\n";
        $twitter .= '<meta name="twitter:app:name:ipad" content="My App">' . "\n";
        $twitter .= '<meta name="twitter:app:id:ipad" content="123456789">' . "\n";
        $twitter .= '<meta name="twitter:app:name:googleplay" content="My App">' . "\n";
        $twitter .= '<meta name="twitter:app:id:googleplay" content="com.example.myapp">' . "\n";
    }
    
    return ['twitter_app' => $twitter];
});
```

### Step 7: Card Testing Hook

```php
<?php
/**
 * Debug logging for Twitter cards
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    // Log when pages with Twitter cards are accessed
    $twitterDebug = $_GET['twitter_debug'] ?? null;
    
    if ($twitterDebug === 'true') {
        $config = getTwitterConfig();
        $pageConfig = getPageTwitterConfig($vars, $config);
        $errors = validateTwitterCard($pageConfig);
        
        logActivity('Twitter Card Debug: ' . print_r($pageConfig, true));
        if (!empty($errors)) {
            logActivity('Twitter Card Errors: ' . implode(', ', $errors));
        }
    }
    
    return '';
});
```

### Step 8: Image Optimization for Twitter

```css
/* 
 * Twitter Card Image Guidelines:
 * 
 * Summary Card:
 * - Minimum: 120x120px
 * - Recommended: 1200x628px (1.91:1 ratio)
 * 
 * Summary Large Image Card:
 * - Minimum: 300x157px
 * - Recommended: 1200x628px (1.91:1 ratio)
 * 
 * Always use HTTPS and JPG/PNG/WebP format
 */
```

### Step 9: Twitter Validator Integration

```php
<?php
/**
 * Helper to generate Twitter Card preview data
 */
function getTwitterPreviewData($vars) {
    $config = getTwitterConfig();
    $pageConfig = getPageTwitterConfig($vars, $config);
    
    return [
        'card_type' => $pageConfig['card'],
        'title' => substr($pageConfig['title'], 0, 70),
        'description' => substr($pageConfig['description'], 0, 200),
        'image_url' => $pageConfig['image'] ?? $config['default_image'],
        'site_handle' => $config['site'],
        'validation_url' => 'https://cards-dev.twitter.com/validator'
    ];
}
```

## Best Practices
- Use summary_large_image for most pages
- Keep titles under 55 characters
- Keep descriptions under 125 characters
- Use images with 2:1 aspect ratio
- Minimum image size: 300x157px
- Always use HTTPS URLs
- Test with Twitter Card Validator
- Include @mention in cards
- Use consistent branding
- Consider mobile-first design
