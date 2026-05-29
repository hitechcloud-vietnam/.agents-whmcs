---
name: whmcs-image-optimization
description: Image optimization for WHMCS
category: Performance & Monitoring
version: 1.0.0
---

# WHMCS Image Optimization Skill

## Overview
This skill provides patterns for optimizing images in WHMCS.

## Implementation Patterns

### Image Optimizer
```php
<?php
/**
 * WHMCS Image Optimization
 * Optimizes images for web
 */

namespace WHMCS\Module\Performance\Images;

class ImageOptimizer {
    /**
     * Optimize image
     */
    public function optimize(string $inputPath, string $outputPath, array $options = []): array {
        $quality = $options['quality'] ?? 85;
        $maxWidth = $options['max_width'] ?? 1920;
        $maxHeight = $options['max_height'] ?? 1080;

        $image = imagecreatefromstring(file_get_contents($inputPath));
        $width = imagesx($image);
        $height = imagesy($image);

        // Resize if needed
        if ($width > $maxWidth || $height > $maxHeight) {
            $ratio = min($maxWidth / $width, $maxHeight / $height);
            $newWidth = (int) ($width * $ratio);
            $newHeight = (int) ($height * $ratio);

            $resized = imagecreatetruecolor($newWidth, $newHeight);
            imagecopyresampled($resized, $image, 0, 0, 0, 0, $newWidth, $newHeight, $width, $height);
            imagedestroy($image);
            $image = $resized;
        }

        // Save as optimized JPEG
        imagejpeg($image, $outputPath, $quality);
        imagedestroy($image);

        return [
            'original_size' => filesize($inputPath),
            'optimized_size' => filesize($outputPath),
            'savings_percent' => round((1 - filesize($outputPath) / filesize($inputPath)) * 100, 2)
        ];
    }

    /**
     * Generate responsive images
     */
    public function generateResponsive(string $inputPath, array $widths): array {
        $outputPaths = [];

        foreach ($widths as $width) {
            $outputPath = preg_replace('/\.(jpg|png)$/', "_{$width}.$1", $inputPath);
            $this->optimize($inputPath, $outputPath, ['max_width' => $width]);
            $outputPaths[$width] = $outputPath;
        }

        return $outputPaths;
    }
}
```

## Best Practices

1. **Modern Formats**: Use WebP/AVIF when supported
2. **Responsive Images**: Generate multiple sizes
3. **Lazy Loading**: Implement lazy loading
4. **CDN Integration**: Serve via CDN
5. **Quality Balance**: Find optimal quality/size balance

## Related Skills

- whmcs-cdn-integration
- whmcs-lazy-loading
- whmcs-resource-optimization
- whmcs-caching-strategies