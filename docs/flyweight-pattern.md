# Flyweight Pattern in WHMCS

The Flyweight Pattern minimizes memory usage by sharing as much data as possible between similar objects. In WHMCS, this pattern is excellent for handling large numbers of similar objects like product configurations, service instances, or cached data.

## Overview

Flyweight pattern reduces memory by sharing common parts of object state:
- Share intrinsic state (unchanging) between objects
- Separate extrinsic state (context-dependent) from objects
- Use factory to manage shared objects
- Store shared data in a central cache
- Reduces memory footprint significantly

## Core Structure

### Flyweight Interface

```php
<?php
// includes/Flyweight/FlyweightInterface.php

namespace CustomModule\Flyweight;

interface FlyweightInterface
{
    public function operation(sharedContext $context): void;
    public function getIntrinsicState(): array;
}
```

## Real-World WHMCS Examples

### Product Configuration Flyweight

```php
<?php
// includes/Flyweight/ProductConfigFlyweight.php

namespace CustomModule\Flyweight;

/**
 * Shared product configuration - intrinsic state
 */
class ProductConfigFlyweight
{
    protected string $productId;
    protected string $productName;
    protected array $configOptions; // Shared across all instances
    protected float $basePrice;
    protected array $permissions;

    private static array $cache = [];

    public function __construct(string $productId, array $data)
    {
        $this->productId = $productId;
        $this->productName = $data['name'] ?? '';
        $this->configOptions = $data['config_options'] ?? [];
        $this->basePrice = $data['base_price'] ?? 0;
        $this->permissions = $data['permissions'] ?? [];
    }

    public static function get(string $productId): ?self
    {
        if (isset(self::$cache[$productId])) {
            return self::$cache[$productId];
        }

        // Load from database
        $data = Capsule::table('tblproducts')
            ->where('id', $productId)
            ->first();

        if (!$data) {
            return null;
        }

        $flyweight = new self($productId, [
            'name' => $data->name,
            'base_price' => $data->monthly,
            'config_options' => json_decode($data->configoptions, true) ?? [],
            'permissions' => $this->loadPermissions($productId)
        ]);

        self::$cache[$productId] = $flyweight;

        return $flyweight;
    }

    public static function preload(array $productIds): void
    {
        $products = Capsule::table('tblproducts')
            ->whereIn('id', $productIds)
            ->get();

        foreach ($products as $product) {
            $flyweight = new self((string)$product->id, [
                'name' => $product->name,
                'base_price' => $product->monthly,
                'config_options' => json_decode($product->configoptions, true) ?? [],
                'permissions' => []
            ]);

            self::$cache[(string)$product->id] = $flyweight;
        }
    }

    public static function clearCache(): void
    {
        self::$cache = [];
    }

    public function getProductId(): string
    {
        return $this->productId;
    }

    public function getProductName(): string
    {
        return $this->productName;
    }

    public function getBasePrice(): float
    {
        return $this->basePrice;
    }

    public function getConfigOptions(): array
    {
        return $this->configOptions;
    }

    public function hasPermission(string $permission): bool
    {
        return in_array($permission, $this->permissions);
    }

    // Flyweight operation - takes extrinsic state
    public function calculatePrice(array $extrinsicState): float
    {
        $price = $this->basePrice;

        // Apply billing cycle adjustment (extrinsic)
        if (isset($extrinsicState['billing_cycle'])) {
            $price = $this->getPriceForCycle($extrinsicState['billing_cycle']);
        }

        // Apply selected config options (extrinsic)
        if (isset($extrinsicState['selected_options'])) {
            foreach ($extrinsicState['selected_options'] as $optionId) {
                $price += $this->getOptionPrice($optionId);
            }
        }

        // Apply quantity discount (extrinsic)
        if (isset($extrinsicState['quantity'])) {
            $price *= $this->getQuantityMultiplier($extrinsicState['quantity']);
        }

        return $price;
    }

    protected function getPriceForCycle(string $cycle): float
    {
        $prices = [
            'monthly' => $this->basePrice,
            'quarterly' => $this->basePrice * 3 * 0.95,
            'semiannually' => $this->basePrice * 6 * 0.90,
            'annually' => $this->basePrice * 12 * 0.85,
            'biennially' => $this->basePrice * 24 * 0.80
        ];

        return $prices[$cycle] ?? $this->basePrice;
    }

    protected function getOptionPrice(string $optionId): float
    {
        foreach ($this->configOptions as $option) {
            if ($option['id'] === $optionId) {
                return $option['price'] ?? 0;
            }
        }
        return 0;
    }

    protected function getQuantityMultiplier(int $quantity): float
    {
        if ($quantity >= 100) return 0.70;
        if ($quantity >= 50) return 0.80;
        if ($quantity >= 20) return 0.90;
        return 1.0;
    }

    protected function loadPermissions(string $productId): array
    {
        return Capsule::table('tblproduct_permissions')
            ->where('product_id', $productId)
            ->pluck('permission')
            ->toArray();
    }
}
```

### Service Instance Flyweight

```php
<?php
// includes/Flyweight/ServiceInstanceFactory.php

namespace CustomModule\Flyweight;

/**
 * Manages service instance flyweights
 */
class ServiceInstanceFactory
{
    protected static array $flyweights = [];
    protected static array $instanceData = [];

    /**
     * Get or create a flyweight for service template
     */
    public static function getServiceTemplate(string $templateId): ServiceTemplateFlyweight
    {
        if (!isset(self::$flyweights[$templateId])) {
            $template = Capsule::table('tblproduct_groups')
                ->where('id', $templateId)
                ->first();

            if (!$template) {
                throw new \InvalidArgumentException("Template not found: {$templateId}");
            }

            self::$flyweights[$templateId] = new ServiceTemplateFlyweight(
                $templateId,
                [
                    'name' => $template->name,
                    'description' => $template->description,
                    'modules' => $this->loadModules($templateId),
                    'settings' => $this->loadSettings($templateId)
                ]
            );
        }

        return self::$flyweights[$templateId];
    }

    /**
     * Create service instance with extrinsic data
     */
    public static function createServiceInstance(
        string $templateId,
        int $clientId,
        string $domain,
        array $extrinsicData
    ): ServiceInstance {
        $template = self::getServiceTemplate($templateId);

        $instanceId = uniqid('svc_');

        // Store extrinsic data separately
        self::$instanceData[$instanceId] = [
            'client_id' => $clientId,
            'domain' => $domain,
            'custom_config' => $extrinsicData['custom_config'] ?? [],
            'notes' => $extrinsicData['notes'] ?? '',
            'ip_address' => $extrinsicData['ip_address'] ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ];

        return new ServiceInstance($instanceId, $template, self::$instanceData[$instanceId]);
    }

    public static function getInstanceData(string $instanceId): ?array
    {
        return self::$instanceData[$instanceId] ?? null;
    }

    public static function clearCache(): void
    {
        self::$flyweights = [];
    }

    protected static function loadModules(string $templateId): array
    {
        return Capsule::table('tblproduct_group_modules')
            ->where('group_id', $templateId)
            ->get()
            ->toArray();
    }

    protected static function loadSettings(string $templateId): array
    {
        return Capsule::table('tblproduct_settings')
            ->where('group_id', $templateId)
            ->pluck('value', 'key')
            ->toArray();
    }
}
```

```php
<?php
// includes/Flyweight/ServiceTemplateFlyweight.php

namespace CustomModule\Flyweight;

class ServiceTemplateFlyweight
{
    protected string $templateId;
    protected string $name;
    protected string $description;
    protected array $modules;
    protected array $settings;

    public function __construct(string $templateId, array $data)
    {
        $this->templateId = $templateId;
        $this->name = $data['name'];
        $this->description = $data['description'];
        $this->modules = $data['modules'];
        $this->settings = $data['settings'];
    }

    public function getTemplateId(): string
    {
        return $this->templateId;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function getSettings(): array
    {
        return $this->settings;
    }

    public function getSetting(string $key, $default = null)
    {
        return $this->settings[$key] ?? $default;
    }

    public function hasModule(string $moduleType): bool
    {
        foreach ($this->modules as $module) {
            if ($module->type === $moduleType) {
                return true;
            }
        }
        return false;
    }

    public function getModules(): array
    {
        return $this->modules;
    }
}
```

```php
<?php
// includes/Flyweight/ServiceInstance.php

namespace CustomModule\Flyweight;

/**
 * Service instance - holds extrinsic state
 */
class ServiceInstance
{
    protected string $instanceId;
    protected ServiceTemplateFlyweight $template;
    protected array $extrinsicData;

    public function __construct(string $instanceId, ServiceTemplateFlyweight $template, array $extrinsicData)
    {
        $this->instanceId = $instanceId;
        $this->template = $template;
        $this->extrinsicData = $extrinsicData;
    }

    public function getInstanceId(): string
    {
        return $this->instanceId;
    }

    public function getTemplate(): ServiceTemplateFlyweight
    {
        return $this->template;
    }

    public function getClientId(): int
    {
        return $this->extrinsicData['client_id'];
    }

    public function getDomain(): string
    {
        return $this->extrinsicData['domain'];
    }

    public function getCustomConfig(): array
    {
        return $this->extrinsicData['custom_config'];
    }

    public function getIpAddress(): ?string
    {
        return $this->extrinsicData['ip_address'];
    }

    public function setIpAddress(string $ip): void
    {
        $this->extrinsicData['ip_address'] = $ip;
    }

    public function toArray(): array
    {
        return array_merge(
            [
                'instance_id' => $this->instanceId,
                'template_id' => $this->template->getTemplateId(),
                'template_name' => $this->template->getName()
            ],
            $this->extrinsicData
        );
    }
}
```

### Cached Configuration Flyweight

```php
<?php
// includes/Flyweight/ConfigCacheFlyweight.php

namespace CustomModule\Flyweight;

class ConfigCacheFlyweight
{
    protected array $configs = [];
    protected int $maxCacheSize = 1000;
    protected int $ttlSeconds = 3600;

    private static array $sharedInstances = [];

    public static function getInstance(string $configType): self
    {
        if (!isset(self::$sharedInstances[$configType])) {
            self::$sharedInstances[$configType] = new self($configType);
        }

        return self::$sharedInstances[$configType];
    }

    public function get(string $key): mixed
    {
        if (!isset($this->configs[$key])) {
            return null;
        }

        $entry = $this->configs[$key];

        // Check TTL
        if (time() > $entry['expires_at']) {
            unset($this->configs[$key]);
            return null;
        }

        return $entry['value'];
    }

    public function set(string $key, $value, ?int $ttl = null): void
    {
        // Enforce max cache size
        if (count($this->configs) >= $this->maxCacheSize) {
            $this->evictOldest();
        }

        $this->configs[$key] = [
            'value' => $value,
            'created_at' => time(),
            'expires_at' => time() + ($ttl ?? $this->ttlSeconds)
        ];
    }

    public function has(string $key): bool
    {
        return $this->get($key) !== null;
    }

    public function delete(string $key): void
    {
        unset($this->configs[$key]);
    }

    public function clear(): void
    {
        $this->configs = [];
    }

    protected function evictOldest(): void
    {
        $oldestKey = null;
        $oldestTime = PHP_INT_MAX;

        foreach ($this->configs as $key => $entry) {
            if ($entry['created_at'] < $oldestTime) {
                $oldestTime = $entry['created_at'];
                $oldestKey = $key;
            }
        }

        if ($oldestKey !== null) {
            unset($this->configs[$oldestKey]);
        }
    }

    public function getStats(): array
    {
        return [
            'size' => count($this->configs),
            'max_size' => $this->maxCacheSize,
            'ttl' => $this->ttlSeconds
        ];
    }
}
```

### Usage Example

```php
<?php
// Flyweight pattern for product pricing

// Instead of loading all product data for each instance...
// Heavy: new Product(123, $clientData, $allConfigOptions, ...)

// Use flyweight to share common data
$template = ProductConfigFlyweight::get('123');

$price1 = $template->calculatePrice([
    'billing_cycle' => 'annually',
    'selected_options' => ['opt1', 'opt2'],
    'quantity' => 5
]);

$price2 = $template->calculatePrice([
    'billing_cycle' => 'monthly',
    'selected_options' => ['opt1'],
    'quantity' => 1
]);

// Memory efficient - only one ProductConfigFlyweight instance
// but two different price calculations based on extrinsic state

// Batch preload for better performance
ProductConfigFlyweight::preload(['123', '456', '789', '101']);

foreach ($clientProducts as $productId) {
    $template = ProductConfigFlyweight::get($productId);
    // ...
}
```

### Email Template Flyweight

```php
<?php
// includes/Flyweight/EmailTemplateFlyweight.php

namespace CustomModule\Flyweight;

class EmailTemplateFlyweight
{
    protected string $templateId;
    protected string $subject;
    protected string $body;
    protected array $variables; // Shared template variables
    protected array $attachments;

    private static array $cache = [];

    public static function get(string $templateId): ?self
    {
        if (isset(self::$cache[$templateId])) {
            return self::$cache[$templateId];
        }

        $template = Capsule::table('tblemailtemplates')
            ->where('id', $templateId)
            ->first();

        if (!$template) {
            return null;
        }

        $flyweight = new self($templateId, [
            'subject' => $template->subject,
            'body' => $template->message,
            'variables' => $this->extractVariables($template->message),
            'attachments' => $this->loadAttachments($templateId)
        ]);

        self::$cache[$templateId] = $flyweight;

        return $flyweight;
    }

    public static function preloadByType(string $type): void
    {
        $templates = Capsule::table('tblemailtemplates')
            ->where('type', $type)
            ->get();

        foreach ($templates as $template) {
            self::get((string)$template->id);
        }
    }

    public function render(array $variables): EmailContent
    {
        $subject = $this->subject;
        $body = $this->body;

        // Replace variables (extrinsic - changes per email)
        foreach ($variables as $key => $value) {
            $subject = str_replace('{' . $key . '}', $value, $subject);
            $body = str_replace('{' . $key . '}', $value, $body);
        }

        return new EmailContent($subject, $body, $this->attachments);
    }

    protected function extractVariables(string $template): array
    {
        preg_match_all('/\{([A-Za-z0-9_]+)\}/', $template, $matches);
        return array_unique($matches[1]);
    }

    protected function loadAttachments(string $templateId): array
    {
        return Capsule::table('tblemail_template_attachments')
            ->where('template_id', $templateId)
            ->pluck('file_path')
            ->toArray();
    }
}
```

## Pros

- **Memory Efficiency**: Dramatically reduces memory usage
- **Performance**: Faster object creation with cached instances
- **Centralization**: Shared state in one place
- **Consistency**: All clients get same shared data

## Cons

- **Complexity**: More complex implementation
- **Trade-off**: CPU vs memory trade-off
- **Synchronization**: Thread safety considerations
- **Debugging**: Harder to track shared state changes

## Best Practices

1. Identify shared intrinsic state early
2. Separate intrinsic from extrinsic state clearly
3. Use factory to manage flyweight creation
4. Implement proper cache invalidation
5. Consider LRU eviction for bounded caches
6. Document which state is shared vs per-instance