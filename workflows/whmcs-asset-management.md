# WHMCS Asset Management Workflow

## Purpose
Properly manage static assets (CSS, JS, images, fonts) in WHMCS.

## Prerequisites
- WHMCS installation
- FTP/cPanel file access
- Basic asset management knowledge

## Step-by-Step Process

### Step 1: Understand Asset Directory Structure
```
/whmcs/
├── assets/                    # Core WHMCS assets
│   ├── css/
│   ├── js/
│   ├── img/
│   └── fonts/
├── templates/               # Template assets
│   └── your_template/
│       ├── css/
│       ├── js/
│       ├── img/
│       └── fonts/
└── downloads/               # Downloadable assets
```

### Step 2: Register Custom Assets

**Create Hook File:**
```php
<?php
/**
 * Register custom assets via hook
 */
use WHMCS\View\Asset;

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $assets = '';
    
    // Custom CSS
    $assets .= '<link rel="stylesheet" href="' . 
               Asset::url('/css/custom.min.css') . '">';
    
    // Custom JS
    $assets .= '<script src="' . 
               Asset::url('/js/custom.min.js') . '"></script>';
    
    return $assets;
});
```

### Step 3: Asset Versioning

**Versioned Asset Path:**
```php
<?php
/**
 * Asset versioning for cache busting
 */
define('ASSET_VERSION', '1.0.5');

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $version = defined('ASSET_VERSION') ? ASSET_VERSION : time();
    
    return '<link rel="stylesheet" href="/css/custom.css?v=' . $version . '">';
});
```

### Step 4: Image Optimization

**Compression Script:**
```php
<?php
/**
 * Optimize images before upload
 */
function optimizeImage($source, $destination, $quality = 80) {
    $info = getimagesize($source);
    
    switch ($info['mime']) {
        case 'image/jpeg':
            $image = imagecreatefromjpeg($source);
            imagejpeg($image, $destination, $quality);
            break;
        case 'image/png':
            $image = imagecreatefrompng($source);
            imagepng($image, $destination, 9);
            break;
        case 'image/webp':
            $image = imagecreatefromwebp($source);
            imagewebp($image, $destination, $quality);
            break;
    }
    
    imagedestroy($image);
}
```

### Step 5: CDN Integration

**CDN Asset Hook:**
```php
<?php
/**
 * Serve assets from CDN
 */
define('CDN_URL', 'https://cdn.yourdomain.com');

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $cdnUrl = defined('CDN_URL') ? CDN_URL : '';
    
    if (empty($cdnUrl)) return '';
    
    $assets = '';
    
    // Replace local assets with CDN URLs
    $templatePath = '/templates/' . $vars['template'];
    
    $assets .= '<link rel="stylesheet" href="' . $cdnUrl . $templatePath . '/css/style.css?v=2">';
    $assets .= '<script src="' . $cdnUrl . $templatePath . '/js/app.js?v=2"></script>';
    
    return $assets;
});
```

### Step 6: Lazy Loading Images

**Implementation Hook:**
```php
<?php
/**
 * Add lazy loading to images
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    return [
        'lazy_loading_enabled' => true
    ];
});
```

**Template Code:**
```smarty
{foreach from=$products item=product}
    <img src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw=="
         data-src="{$product.image}"
         alt="{$product.name}"
         class="lazy">
{/foreach}

<script>
document.addEventListener("DOMContentLoaded", function() {
    const lazyImages = document.querySelectorAll('.lazy');
    
    const observer = new IntersectionObserver(function(entries) {
        entries.forEach(entry => {
            if (entry.isIntersecting) {
                const img = entry.target;
                img.src = img.dataset.src;
                img.classList.remove('lazy');
                observer.unobserve(img);
            }
        });
    });
    
    lazyImages.forEach(img => observer.observe(img));
});
</script>
```

### Step 7: Font Management

**Custom Fonts Setup:**
```php
<?php
/**
 * Register custom fonts
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $fontUrl = '/templates/' . $vars['template'] . '/fonts/';
    
    return '<style>
        @font-face {
            font-family: "CustomFont";
            src: url("' . $fontUrl . 'CustomFont-Regular.woff2") format("woff2"),
                 url("' . $fontUrl . 'CustomFont-Regular.woff") format("woff");
            font-weight: 400;
            font-style: normal;
        }
        
        @font-face {
            font-family: "CustomFont";
            src: url("' . $fontUrl . 'CustomFont-Bold.woff2") format("woff2"),
                 url("' . $fontUrl . 'CustomFont-Bold.woff") format("woff");
            font-weight: 700;
            font-style: normal;
        }
    </style>';
});
```

### Step 8: Asset Manifest

**manifest.json:**
```json
{
    "custom.css": {
        "file": "css/custom.min.css",
        "hash": "a1b2c3d4",
        "size": 15234
    },
    "custom.js": {
        "file": "js/custom.min.js",
        "hash": "e5f6g7h8",
        "size": 8765
    }
}
```

**Load from Manifest:**
```php
<?php
/**
 * Load assets from manifest
 */
function loadAssetsFromManifest($template, $manifestPath) {
    if (!file_exists($manifestPath)) return [];
    
    $manifest = json_decode(file_get_contents($manifestPath), true);
    $assets = [];
    
    foreach ($manifest as $name => $data) {
        $assets[$name] = [
            'url' => '/templates/' . $template . '/' . $data['file'] . '?v=' . $data['hash'],
            'size' => $data['size']
        ];
    }
    
    return $assets;
}
```

### Step 9: Asset Pipeline

**Combine and Minify:**
```php
<?php
/**
 * Asset pipeline - combine CSS files
 */
function combineAssets($files, $outputFile, $type = 'css') {
    $combined = '';
    
    foreach ($files as $file) {
        if (file_exists($file)) {
            $combined .= file_get_contents($file) . "\n";
        }
    }
    
    // Minify based on type
    if ($type === 'css') {
        $combined = minifyCSS($combined);
    } elseif ($type === 'js') {
        $combined = minifyJS($combined);
    }
    
    file_put_contents($outputFile, $combined);
}
```

### Step 10: Cache Management

**Clear Asset Cache:**
```php
<?php
/**
 * Asset cache management
 */
function clearAssetCache($template = null) {
    $cacheDir = ROOTDIR . '/templates_c/';
    
    if ($template) {
        $pattern = $cacheDir . '*' . $template . '*';
    } else {
        $pattern = $cacheDir . '*';
    }
    
    foreach (glob($pattern) as $file) {
        if (is_file($file)) {
            unlink($file);
        }
    }
}

// Hook to clear cache when assets update
add_hook('AssetUpdated', 1, function($vars) {
    clearAssetCache($vars['template'] ?? null);
});
```

## Best Practices
- Always use relative URLs for local assets
- Implement cache busting with version hashes
- Use WebP format for images when possible
- Minify CSS and JavaScript for production
- Lazy load images below the fold
- Serve fonts with font-display: swap
- Use preconnect for external resources
- Monitor asset loading performance
- Organize assets by type (css, js, img, fonts)
- Use consistent naming conventions
