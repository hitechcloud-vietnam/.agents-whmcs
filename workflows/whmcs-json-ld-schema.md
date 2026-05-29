# WHMCS JSON-LD Schema Workflow

## Purpose
Implement JSON-LD structured data in WHMCS for enhanced search engine visibility.

## Prerequisites
- WHMCS installation
- Template access
- Understanding of Schema.org
- Basic JSON knowledge

## Step-by-Step Process

### Step 1: Create JSON-LD Hook File

**Create hooks/json_ld_schema.php:**
```php
<?php
/**
 * WHMCS JSON-LD Structured Data
 * Schema.org implementation for enhanced SEO
 */

use WHMCS\Config\Setting;

function getCompanyData() {
    return [
        '@context' => 'https://schema.org',
        '@type' => 'Organization',
        'name' => Setting::getValue('CompanyName') ?: 'Your Company',
        'url' => rtrim(Setting::getValue('SystemURL'), '/'),
        'logo' => [
            '@type' => 'ImageObject',
            'url' => Setting::getValue('SystemURL') . '/assets/img/logo.png',
            'width' => 200,
            'height' => 60
        ],
        'contactPoint' => [
            '@type' => 'ContactPoint',
            'telephone' => Setting::getValue('PhoneNumber') ?: '+1-555-555-5555',
            'contactType' => 'customer service',
            'availableLanguage' => 'English',
            'areaServed' => 'Worldwide'
        ],
        'sameAs' => getSocialLinks()
    ];
}

function getSocialLinks() {
    $links = [];
    
    $socials = [
        'FacebookURL',
        'TwitterURL', 
        'LinkedInURL',
        'YouTubeURL',
        'InstagramURL'
    ];
    
    foreach ($socials as $social) {
        $url = Setting::getValue($social);
        if ($url) {
            $links[] = $url;
        }
    }
    
    return $links;
}
```

### Step 2: Core Schema Implementation

```php
<?php
/**
 * Add JSON-LD structured data to head
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $schemas = [];
    
    // Always include Organization schema
    $schemas[] = getCompanyData();
    
    // Always include WebSite schema with search action
    $schemas[] = [
        '@context' => 'https://schema.org',
        '@type' => 'WebSite',
        'name' => getCompanyData()['name'],
        'url' => getCompanyData()['url'],
        'potentialAction' => [
            '@type' => 'SearchAction',
            'target' => [
                '@type' => 'EntryPoint',
                'urlTemplate' => getCompanyData()['url'] . '/search?q={search_term_string}'
            ],
            'query-input' => 'required name=search_term_string'
        ]
    ];
    
    // Generate JSON-LD scripts
    $jsonLd = '';
    foreach ($schemas as $schema) {
        $jsonLd .= '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES | JSON_PRETTY_PRINT) . '</script>' . "\n";
    }
    
    return $jsonLd;
});
```

### Step 3: Breadcrumb Schema

```php
<?php
/**
 * BreadcrumbList schema for navigation
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $items = [];
    
    // Home
    $items[] = [
        '@type' => 'ListItem',
        'position' => 1,
        'name' => 'Home',
        'item' => getCompanyData()['url']
    ];
    
    $filename = $vars['filename'] ?? '';
    $position = 2;
    
    // Add page-specific breadcrumbs
    switch ($filename) {
        case 'announcements':
            $items[] = [
                '@type' => 'ListItem',
                'position' => $position,
                'name' => 'Announcements',
                'item' => getCompanyData()['url'] . '/announcements'
            ];
            
            if (isset($vars['announcement'])) {
                $position++;
                $items[] = [
                    '@type' => 'ListItem',
                    'position' => $position,
                    'name' => $vars['announcement']['title'],
                    'item' => getCurrentUrl()
                ];
            }
            break;
            
        case 'knowledgebase':
            $items[] = [
                '@type' => 'ListItem',
                'position' => $position,
                'name' => 'Knowledge Base',
                'item' => getCompanyData()['url'] . '/knowledgebase'
            ];
            
            if (isset($vars['kbarticle'])) {
                $position++;
                $items[] = [
                    '@type' => 'ListItem',
                    'position' => $position,
                    'name' => $vars['kbarticle']['title'],
                    'item' => getCurrentUrl()
                ];
            }
            break;
            
        case 'supporttickets':
            $items[] = [
                '@type' => 'ListItem',
                'position' => $position,
                'name' => 'Support Tickets',
                'item' => getCompanyData()['url'] . '/supporttickets'
            ];
            break;
            
        case 'clientarea':
            $items[] = [
                '@type' => 'ListItem',
                'position' => $position,
                'name' => 'My Account',
                'item' => getCompanyData()['url'] . '/clientarea'
            ];
            
            if (isset($vars['service'])) {
                $position++;
                $items[] = [
                    '@type' => 'ListItem',
                    'position' => $position,
                    'name' => $vars['service']['domain'],
                    'item' => getCurrentUrl()
                ];
            }
            break;
    }
    
    $breadcrumbSchema = [
        '@context' => 'https://schema.org',
        '@type' => 'BreadcrumbList',
        'itemListElement' => $items
    ];
    
    return ['breadcrumb_schema' => '<script type="application/ld+json">' . json_encode($breadcrumbSchema) . '</script>'];
});

function getCurrentUrl() {
    $protocol = (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off') ? 'https' : 'http';
    return $protocol . '://' . ($_SERVER['HTTP_HOST'] ?? '') . ($_SERVER['REQUEST_URI'] ?? '/');
}
```

### Step 4: Product Schema

```php
<?php
/**
 * Product schema for product pages
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $productSchema = '';
    
    if (isset($vars['productinfo']) && is_array($vars['productinfo'])) {
        $product = $vars['productinfo'];
        
        // Determine availability
        $availability = 'https://schema.org/InStock';
        if (!empty($product['stock'])) {
            $availability = $product['stock'] > 0 
                ? 'https://schema.org/InStock' 
                : 'https://schema.org/OutOfStock';
        }
        
        $schema = [
            '@context' => 'https://schema.org',
            '@type' => 'Product',
            'name' => $product['name'],
            'description' => strip_tags($product['description'] ?? ''),
            'image' => !empty($product['image']) 
                ? getCompanyData()['url'] . '/' . $product['image'] 
                : getCompanyData()['url'] . '/images/product-default.jpg',
            'sku' => $product['sku'] ?? 'PROD-' . $product['id'],
            'brand' => [
                '@type' => 'Brand',
                'name' => getCompanyData()['name']
            ],
            'offers' => [
                '@type' => 'Offer',
                'url' => getCurrentUrl(),
                'priceCurrency' => 'USD',
                'price' => $product['pricing']['monthly']['price'] ?? '0.00',
                'priceValidUntil' => date('Y-12-31'),
                'availability' => $availability,
                'seller' => [
                    '@type' => 'Organization',
                    'name' => getCompanyData()['name']
                ]
            ]
        ];
        
        // Add rating if available
        if (!empty($product['rating'])) {
            $schema['aggregateRating'] = [
                '@type' => 'AggregateRating',
                'ratingValue' => $product['rating'],
                'reviewCount' => $product['reviewCount'] ?? 1
            ];
        }
        
        $productSchema = '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
    }
    
    return ['product_schema' => $productSchema];
});
```

### Step 5: FAQ Schema

```php
<?php
/**
 * FAQ schema for knowledgebase pages
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $faqSchema = '';
    
    // For knowledgebase category or article pages
    if (isset($vars['kbarticles']) && is_array($vars['kbarticles'])) {
        $faqs = [];
        
        foreach (array_slice($vars['kbarticles'], 0, 10) as $article) {
            $faqs[] = [
                '@type' => 'Question',
                'name' => $article['title'],
                'acceptedAnswer' => [
                    '@type' => 'Answer',
                    'text' => strip_tags($article['article'] ?? $article['preview'] ?? '')
                ]
            ];
        }
        
        if (!empty($faqs)) {
            $schema = [
                '@context' => 'https://schema.org',
                '@type' => 'FAQPage',
                'mainEntity' => $faqs
            ];
            
            $faqSchema = '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
        }
    }
    
    return ['faq_schema' => $faqSchema];
});
```

### Step 6: Local Business Schema

```php
<?php
/**
 * LocalBusiness schema
 */
function getLocalBusinessSchema() {
    $schema = [
        '@context' => 'https://schema.org',
        '@type' => 'LocalBusiness',
        '@id' => getCompanyData()['url'] . '/#business',
        'name' => getCompanyData()['name'],
        'url' => getCompanyData()['url'],
        'logo' => getCompanyData()['logo']['url'],
        'image' => getCompanyData()['logo']['url'],
        'description' => 'Professional web hosting and cloud services provider.',
        'telephone' => Setting::getValue('PhoneNumber') ?: '+1-555-555-5555',
        'email' => Setting::getValue('Email') ?: 'support@example.com',
        'address' => [
            '@type' => 'PostalAddress',
            'streetAddress' => Setting::getValue('Address1') ?: '123 Main Street',
            'addressLocality' => Setting::getValue('City') ?: 'City',
            'addressRegion' => Setting::getValue('State') ?: 'State',
            'postalCode' => Setting::getValue('Postcode') ?: '12345',
            'addressCountry' => Setting::getValue('Country') ?: 'US'
        ],
        'geo' => [
            '@type' => 'GeoCoordinates',
            'latitude' => Setting::getValue('Latitude') ?: '40.7128',
            'longitude' => Setting::getValue('Longitude') ?: '-74.0060'
        ],
        'openingHoursSpecification' => [
            '@type' => 'OpeningHoursSpecification',
            'dayOfWeek' => ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday'],
            'opens' => '00:00',
            'closes' => '23:59'
        ],
        'sameAs' => getSocialLinks()
    ];
    
    return $schema;
}
```

### Step 7: Article/News Schema

```php
<?php
/**
 * Article schema for announcements
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $articleSchema = '';
    
    if (isset($vars['announcement']) && is_array($vars['announcement'])) {
        $announcement = $vars['announcement'];
        
        $schema = [
            '@context' => 'https://schema.org',
            '@type' => 'Article',
            'headline' => $announcement['title'],
            'description' => $announcement['summary'] ?? '',
            'datePublished' => ($announcement['date'] ?? date('Y-m-d')) . 'T00:00:00+00:00',
            'dateModified' => ($announcement['modified'] ?? $announcement['date'] ?? date('Y-m-d')) . 'T00:00:00+00:00',
            'author' => [
                '@type' => 'Organization',
                'name' => getCompanyData()['name']
            ],
            'publisher' => [
                '@type' => 'Organization',
                'name' => getCompanyData()['name'],
                'logo' => getCompanyData()['logo']
            ],
            'mainEntityOfPage' => [
                '@type' => 'WebPage',
                '@id' => getCurrentUrl()
            ],
            'image' => [
                '@type' => 'ImageObject',
                'url' => getCompanyData()['url'] . '/images/og-announcement.jpg'
            ]
        ];
        
        $articleSchema = '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
    }
    
    return ['article_schema' => $articleSchema];
});
```

### Step 8: Service Schema

```php
<?php
/**
 * Service schema for hosting services
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $serviceSchema = '';
    
    if (isset($vars['productinfo']) && is_array($vars['productinfo'])) {
        $product = $vars['productinfo'];
        
        $schema = [
            '@context' => 'https://schema.org',
            '@type' => 'Service',
            'name' => $product['name'],
            'description' => strip_tags($product['description'] ?? ''),
            'provider' => [
                '@type' => 'Organization',
                'name' => getCompanyData()['name']
            ],
            'areaServed' => 'Worldwide',
            'hasOfferCatalog' => [
                '@type' => 'OfferCatalog',
                'name' => $product['group'] ?? 'Hosting Services',
                'itemListElement' => [
                    '@type' => 'Offer',
                    'itemOffered' => [
                        '@type' => 'Service',
                        'name' => $product['name']
                    ],
                    'price' => $product['pricing']['monthly']['price'] ?? '0.00',
                    'priceCurrency' => 'USD'
                ]
            ]
        ];
        
        $serviceSchema = '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
    }
    
    return ['service_schema' => $serviceSchema];
});
```

### Step 9: Review/Rating Schema

```php
<?php
/**
 * Review schema for testimonials
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $reviewSchema = '';
    
    // Testimonials or reviews section
    if (isset($vars['reviews']) && is_array($vars['reviews'])) {
        $reviews = [];
        
        foreach ($vars['reviews'] as $review) {
            $reviews[] = [
                '@type' => 'Review',
                'reviewRating' => [
                    '@type' => 'Rating',
                    'ratingValue' => $review['rating'] ?? 5,
                    'bestRating' => 5,
                    'worstRating' => 1
                ],
                'author' => [
                    '@type' => 'Person',
                    'name' => $review['author'] ?? 'Verified Customer'
                ],
                'reviewBody' => $review['content'] ?? '',
                'datePublished' => $review['date'] ?? date('Y-m-d')
            ];
        }
        
        $schema = [
            '@context' => 'https://schema.org',
            '@type' => 'Product',
            'name' => getCompanyData()['name'] . ' Services',
            'aggregateRating' => [
                '@type' => 'AggregateRating',
                'ratingValue' => array_sum(array_column($vars['reviews'], 'rating')) / count($vars['reviews']),
                'reviewCount' => count($vars['reviews'])
            ],
            'review' => $reviews
        ];
        
        $reviewSchema = '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
    }
    
    return ['review_schema' => $reviewSchema];
});
```

### Step 10: Video Schema

```php
<?php
/**
 * Video schema for tutorial videos
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $videoSchema = '';
    
    if (isset($vars['video']) && is_array($vars['video'])) {
        $video = $vars['video'];
        
        $schema = [
            '@context' => 'https://schema.org',
            '@type' => 'VideoObject',
            'name' => $video['title'],
            'description' => $video['description'] ?? '',
            'thumbnailUrl' => [$video['thumbnail'] ?? ''],
            'uploadDate' => $video['date'] ?? date('Y-m-d'),
            'duration' => $video['duration'] ?? 'PT5M',
            'contentUrl' => $video['url'],
            'embedUrl' => $video['embed_url'] ?? '',
            'publisher' => [
                '@type' => 'Organization',
                'name' => getCompanyData()['name'],
                'logo' => getCompanyData()['logo']
            ]
        ];
        
        $videoSchema = '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
    }
    
    return ['video_schema' => $videoSchema];
});
```

## Best Practices
- Use JSON-LD format (preferred by Google)
- Place scripts in document head when possible
- Use complete, valid JSON
- Include all required properties for each type
- Use the most specific schema types
- Test with Google's Rich Results Test
- Validate with Schema.org validator
- Update schemas when content changes
- Don't duplicate schema in multiple formats
- Include only relevant schemas per page
