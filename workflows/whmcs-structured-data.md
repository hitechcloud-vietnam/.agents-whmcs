# WHMCS Structured Data Workflow

## Purpose
Implement comprehensive structured data markup in WHMCS for rich search results.

## Prerequisites
- WHMCS installation
- Template access
- Schema.org knowledge
- JSON-LD understanding

## Step-by-Step Process

### Step 1: Create Structured Data Manager

**Create hooks/structured_data.php:**
```php
<?php
/**
 * WHMCS Structured Data Manager
 * Centralized structured data implementation
 */

use WHMCS\Config\Setting;

class StructuredDataManager {
    private $schemas = [];
    private $baseUrl;
    private $companyName;
    
    public function __construct() {
        $this->baseUrl = rtrim(Setting::getValue('SystemURL'), '/');
        $this->companyName = Setting::getValue('CompanyName') ?: 'Your Company';
    }
    
    public function addSchema($type, $data) {
        $this->schemas[$type] = $data;
    }
    
    public function getSchemas() {
        return $this->schemas;
    }
    
    public function renderScripts() {
        $output = '';
        foreach ($this->schemas as $schema) {
            $output .= '<script type="application/ld+json">' . 
                       json_encode($schema, JSON_UNESCAPED_SLASHES | JSON_PRETTY_PRINT) . 
                       '</script>' . "\n";
        }
        return $output;
    }
}

$sdManager = new StructuredDataManager();
```

### Step 2: Organization Schema

```php
<?php
/**
 * Organization structured data
 */
function addOrganizationSchema($manager) {
    $socialLinks = [];
    $socialFields = ['FacebookURL', 'TwitterURL', 'LinkedInURL', 'YouTubeURL', 'InstagramURL'];
    
    foreach ($socialFields as $field) {
        $url = Setting::getValue($field);
        if ($url) {
            $socialLinks[] = $url;
        }
    }
    
    $schema = [
        '@context' => 'https://schema.org',
        '@type' => ['Organization', 'Corporation'],
        '@id' => getCompanyUrl() . '/#organization',
        'name' => getCompanyName(),
        'url' => getCompanyUrl(),
        'logo' => [
            '@type' => 'ImageObject',
            'url' => getCompanyUrl() . '/assets/img/logo.png',
            'width' => 200,
            'height' => 60
        ],
        'image' => getCompanyUrl() . '/assets/img/logo.png',
        'description' => 'Professional web hosting and cloud solutions provider offering reliable hosting, VPS, domains, and SSL certificates.',
        ' foundingDate' => '2020',
        'foundingLocation' => [
            '@type' => 'Place',
            'name' => 'United States'
        ],
        'contactPoint' => [
            '@type' => 'ContactPoint',
            'telephone' => getCompanyPhone(),
            'contactType' => 'customer service',
            'availableLanguage' => ['English'],
            'areaServed' => 'Worldwide',
            'hoursAvailable' => [
                '@type' => 'OpeningHoursSpecification',
                'dayOfWeek' => ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'],
                'opens' => '00:00',
                'closes' => '23:59'
            ]
        ],
        'address' => [
            '@type' => 'PostalAddress',
            'streetAddress' => Setting::getValue('Address1') ?: '123 Main Street',
            'addressLocality' => Setting::getValue('City') ?: 'City',
            'addressRegion' => Setting::getValue('State') ?: 'State',
            'postalCode' => Setting::getValue('Postcode') ?: '12345',
            'addressCountry' => [
                '@type' => 'Country',
                'name' => Setting::getValue('Country') ?: 'US'
            ]
        ],
        'sameAs' => $socialLinks,
        'knowsAbout' => ['Web Hosting', 'Cloud Computing', 'Domain Registration', 'SSL Certificates'],
        'areaServed' => [
            '@type' => 'Place',
            'name' => 'Worldwide'
        ],
        'priceRange' => '$',
        'hasOfferCatalog' => [
            '@type' => 'OfferCatalog',
            'name' => 'Hosting Services',
            'itemListElement' => [
                ['@type' => 'Offer', 'name' => 'Web Hosting'],
                ['@type' => 'Offer', 'name' => 'VPS Hosting'],
                ['@type' => 'Offer', 'name' => 'Dedicated Servers'],
                ['@type' => 'Offer', 'name' => 'Domain Registration'],
                ['@type' => 'Offer', 'name' => 'SSL Certificates']
            ]
        ]
    ];
    
    $manager->addSchema('organization', $schema);
}
```

### Step 3: WebSite Schema with Search

```php
<?php
/**
 * WebSite schema with search action
 */
function addWebsiteSchema($manager) {
    $schema = [
        '@context' => 'https://schema.org',
        '@type' => 'WebSite',
        '@id' => getCompanyUrl() . '/#website',
        'name' => getCompanyName(),
        'url' => getCompanyUrl(),
        'about' => [
            '@type' => 'Thing',
            'name' => 'Web Hosting Services',
            'description' => 'Professional hosting solutions for businesses of all sizes'
        ],
        'publisher' => [
            '@id' => getCompanyUrl() . '/#organization'
        ],
        'potentialAction' => [
            '@type' => 'SearchAction',
            'target' => [
                '@type' => 'EntryPoint',
                'urlTemplate' => getCompanyUrl() . '/search?q={search_term_string}'
            ],
            'query-input' => 'required name=search_term_string',
            'description' => 'Search for hosting services, domains, and support articles'
        ],
        'inLanguage' => 'en-US',
        'isAccessibleForFree' => 'Yes',
        'isPartOf' => [
            '@id' => getCompanyUrl() . '/#website'
        ]
    ];
    
    $manager->addSchema('website', $schema);
}
```

### Step 4: Software Application Schema

```php
<?php
/**
 * Software application schema for client area
 */
function addSoftwareSchema($manager) {
    $schema = [
        '@context' => 'https://schema.org',
        '@type' => 'SoftwareApplication',
        'name' => getCompanyName() . ' Client Portal',
        'operatingSystem' => 'Web Browser',
        'applicationCategory' => 'BusinessApplication',
        'url' => getCompanyUrl() . '/clientarea',
        'offers' => [
            '@type' => 'Offer',
            'price' => '0',
            'priceCurrency' => 'USD',
            'priceValidUntil' => date('Y-12-31'),
            'availability' => 'https://schema.org/InStock'
        ],
        'description' => 'Manage your hosting services, domains, invoices, and support tickets.',
        'screenshot' => getCompanyUrl() . '/images/client-area-screenshot.jpg',
        'aggregateRating' => [
            '@type' => 'AggregateRating',
            'ratingValue' => '4.8',
            'ratingCount' => '500',
            'bestRating' => '5',
            'worstRating' => '1'
        ],
        'featureList' => [
            'Service Management',
            'Invoice Viewing',
            'Ticket Support',
            'Domain Management',
            'Knowledge Base Access'
        ]
    ];
    
    $manager->addSchema('software', $schema);
}
```

### Step 5: Financial Service Schema

```php
<?php
/**
 * Financial service schema for billing
 */
function addFinancialServiceSchema($manager) {
    $schema = [
        '@context' => 'https://schema.org',
        '@type' => 'FinancialService',
        'name' => getCompanyName() . ' Billing Services',
        'url' => getCompanyUrl() . '/clientarea/billing',
        'logo' => getCompanyUrl() . '/assets/img/logo.png',
        'description' => 'Secure billing and payment processing for hosting services.',
        'provider' => [
            '@id' => getCompanyUrl() . '/#organization'
        ],
        'hasOfferCatalog' => [
            '@type' => 'OfferCatalog',
            'name' => 'Payment Methods',
            'itemListElement' => [
                ['@type' => 'Offer', 'name' => 'Credit Card'],
                ['@type' => 'Offer', 'name' => 'PayPal'],
                ['@type' => 'Offer', 'name' => 'Bank Transfer'],
                ['@type' => 'Offer', 'name' => 'Cryptocurrency']
            ]
        ],
        'feesAndCommissionsSpecification' => 'No hidden fees. Prices as listed.',
        'termsOfService' => getCompanyUrl() . '/terms'
    ];
    
    $manager->addSchema('financial_service', $schema);
}
```

### Step 6: Event Schema (Maintenance Windows)

```php
<?php
/**
 * Event schema for maintenance notices
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $eventSchema = '';
    
    // Check for active maintenance notices
    if (isset($vars['maintenance_events']) && is_array($vars['maintenance_events'])) {
        $events = [];
        
        foreach ($vars['maintenance_events'] as $event) {
            $events[] = [
                '@type' => 'Event',
                'name' => $event['title'],
                'description' => $event['description'],
                'startDate' => $event['start_date'] . 'T' . ($event['start_time'] ?? '00:00:00'),
                'endDate' => $event['end_date'] . 'T' . ($event['end_time'] ?? '23:59:59'),
                'eventStatus' => 'https://schema.org/EventScheduled',
                'eventAttendanceMode' => 'https://schema.org/OnlineEventAttendanceMode',
                'location' => [
                    '@type' => 'VirtualLocation',
                    'url' => getCompanyUrl()
                ],
                'organizer' => [
                    '@type' => 'Organization',
                    'name' => getCompanyName(),
                    'url' => getCompanyUrl()
                ],
                'about' => [
                    '@type' => 'Thing',
                    'name' => $event['affected_services'] ?? 'Server Maintenance'
                ]
            ];
        }
        
        $schema = [
            '@context' => 'https://schema.org',
            '@graph' => $events
        ];
        
        $eventSchema = '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
    }
    
    return ['event_schema' => $eventSchema];
});
```

### Step 7: HowTo Schema (Knowledgebase)

```php
<?php
/**
 * HowTo schema for tutorial articles
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $howToSchema = '';
    
    if (isset($vars['kbarticle']) && isset($vars['kbarticle']['steps'])) {
        $steps = [];
        
        foreach ($vars['kbarticle']['steps'] as $index => $step) {
            $steps[] = [
                '@type' => 'HowToStep',
                'name' => $step['title'] ?? 'Step ' . ($index + 1),
                'text' => $step['description'] ?? '',
                'url' => getCurrentUrl() . '#step-' . ($index + 1)
            ];
        }
        
        $schema = [
            '@context' => 'https://schema.org',
            '@type' => 'HowTo',
            'name' => $vars['kbarticle']['title'],
            'description' => substr(strip_tags($vars['kbarticle']['article'] ?? ''), 0, 160),
            'totalTime' => $vars['kbarticle']['duration'] ?? 'PT15M',
            'tool' => [
                '@type' => 'HowToTool',
                'name' => 'Web Browser'
            ],
            'step' => $steps,
            'author' => [
                '@type' => 'Organization',
                'name' => getCompanyName()
            ],
            'datePublished' => $vars['kbarticle']['date'] ?? date('Y-m-d')
        ];
        
        $howToSchema = '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
    }
    
    return ['howto_schema' => $howToSchema];
});
```

### Step 8: Offer Schema (Pricing Pages)

```php
<?php
/**
 * Offer schema for pricing
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $offerSchema = '';
    
    if (isset($vars['pricing_plans']) && is_array($vars['pricing_plans'])) {
        $offers = [];
        
        foreach ($vars['pricing_plans'] as $plan) {
            $offers[] = [
                '@type' => 'Offer',
                'name' => $plan['name'],
                'description' => $plan['description'] ?? '',
                'url' => getCurrentUrl(),
                'price' => $plan['price'],
                'priceCurrency' => 'USD',
                'priceValidUntil' => date('Y-12-31'),
                'availability' => 'https://schema.org/InStock',
                'itemCondition' => 'https://schema.org/NewCondition',
                'seller' => [
                    '@type' => 'Organization',
                    'name' => getCompanyName()
                ],
                'hasMerchantReturnPolicy' => [
                    '@type' => 'MerchantReturnPolicy',
                    'name' => '30 Day Money Back Guarantee',
                    'returnPolicyCategory' => 'https://schema.org/MoneyBack',
                    'merchantDays' => 30
                ]
            ];
        }
        
        $schema = [
            '@context' => 'https://schema.org',
            '@type' => 'AggregateOffer',
            'lowPrice' => min(array_column($offers, 'price')),
            'highPrice' => max(array_column($offers, 'price')),
            'priceCurrency' => 'USD',
            'offerCount' => count($offers),
            'offers' => $offers
        ];
        
        $offerSchema = '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
    }
    
    return ['offer_schema' => $offerSchema];
});
```

### Step 9: Reservation Schema

```php
<?php
/**
 * Reservation schema for service orders
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $reservationSchema = '';
    
    if (isset($vars['order']) && isset($vars['order']['id'])) {
        $order = $vars['order'];
        
        $schema = [
            '@context' => 'https://schema.org',
            '@type' => 'Reservation',
            'reservationId' => $order['id'],
            'reservationStatus' => getReservationStatus($order['status']),
            'underName' => [
                '@type' => 'Person',
                'name' => $order['client_name']
            ],
            'broker' => [
                '@type' => 'Organization',
                'name' => getCompanyName(),
                'url' => getCompanyUrl()
            ],
            'reservationFor' => [
                '@type' => 'Service',
                'name' => $order['product_name'],
                'provider' => [
                    '@type' => 'Organization',
                    'name' => getCompanyName()
                ]
            ],
            'startDate' => $order['date'],
            'totalPrice' => [
                '@type' => 'PriceSpecification',
                'price' => $order['total'],
                'priceCurrency' => 'USD'
            ],
            'url' => getCompanyUrl() . '/clientarea/orders/' . $order['id']
        ];
        
        $reservationSchema = '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
    }
    
    return ['reservation_schema' => $reservationSchema];
});

function getReservationStatus($status) {
    $statusMap = [
        'pending' => 'https://schema.org/ReservationPending',
        'active' => 'https://schema.org/ReservationConfirmed',
        'completed' => 'https://schema.org/ReservationConfirmed',
        'cancelled' => 'https://schema.org/ReservationCancelled'
    ];
    return $statusMap[$status] ?? 'https://schema.org/ReservationStatus';
}
```

### Step 10: QAPage Schema (FAQ)

```php
<?php
/**
 * QAPage schema for FAQ sections
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $qaSchema = '';
    
    if (isset($vars['faq_items']) && is_array($vars['faq_items'])) {
        $questions = [];
        
        foreach ($vars['faq_items'] as $faq) {
            $questions[] = [
                '@type' => 'Question',
                'name' => $faq['question'],
                'acceptedAnswer' => [
                    '@type' => 'Answer',
                    'text' => $faq['answer'],
                    'author' => [
                        '@type' => 'Organization',
                        'name' => getCompanyName()
                    ]
                ],
                'upvoteCount' => $faq['helpful_count'] ?? 0,
                'dateCreated' => $faq['date'] ?? date('Y-m-d')
            ];
        }
        
        $schema = [
            '@context' => 'https://schema.org',
            '@type' => 'QAPage',
            'mainEntity' => $questions
        ];
        
        $qaSchema = '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
    }
    
    return ['qa_schema' => $qaSchema];
});
```

### Step 11: WebApplication Schema

```php
<?php
/**
 * WebApplication schema
 */
function addWebApplicationSchema($manager) {
    $schema = [
        '@context' => 'https://schema.org',
        '@type' => 'WebApplication',
        'name' => getCompanyName() . ' Control Panel',
        'url' => getCompanyUrl() . '/clientarea',
        'description' => 'Web hosting control panel for managing services, domains, and support.',
        'applicationCategory' => 'WebApplication',
        'operatingSystem' => 'Any',
        'browserRequirements' => 'Requires JavaScript and cookies',
        'screenshot' => getCompanyUrl() . '/images/control-panel.jpg',
        'softwareVersion' => WHMCS_VERSION,
        'aggregateRating' => [
            '@type' => 'AggregateRating',
            'ratingValue' => '4.5',
            'ratingCount' => '1000',
            'bestRating' => '5',
            'worstRating' => '1'
        ],
        'offers' => [
            '@type' => 'Offer',
            'price' => '0',
            'priceCurrency' => 'USD'
        ],
        'author' => [
            '@id' => getCompanyUrl() . '/#organization'
        ]
    ];
    
    $manager->addSchema('webapplication', $schema);
}
```

### Step 12: Collection Page Schema

```php
<?php
/**
 * CollectionPage schema for product listings
 */
add_hook('ClientAreaPagePreOutput', 1, function($vars) {
    $collectionSchema = '';
    
    if (isset($vars['products']) && is_array($vars['products'])) {
        $items = [];
        
        foreach ($vars['products'] as $product) {
            $items[] = [
                '@type' => 'ListItem',
                'position' => count($items) + 1,
                'url' => getCompanyUrl() . '/cart.php?a=add&pid=' . $product['id']
            ];
        }
        
        $schema = [
            '@context' => 'https://schema.org',
            '@type' => 'CollectionPage',
            'name' => $vars['pagetitle'] ?? 'Hosting Plans',
            'description' => 'Browse our selection of web hosting and cloud solutions.',
            'url' => getCurrentUrl(),
            'numberOfItems' => count($items),
            'itemListElement' => $items,
            'publisher' => [
                '@id' => getCompanyUrl() . '/#organization'
            ]
        ];
        
        $collectionSchema = '<script type="application/ld+json">' . json_encode($schema, JSON_UNESCAPED_SLASHES) . '</script>';
    }
    
    return ['collection_schema' => $collectionSchema];
});
```

## Best Practices
- Use multiple schema types where appropriate
- Implement @id for entity matching across schemas
- Test structured data with Google's tool
- Keep data up to date with content
- Use the most specific schema types
- Include required properties
- Nest related entities properly
- Consider using @graph for multiple types
- Validate against Schema.org validator
- Monitor for errors in Search Console
