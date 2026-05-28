# Factory Pattern in WHMCS

The Factory Pattern provides a way to create objects without specifying the exact class of object that will be created. In WHMCS, this pattern is essential for creating modules, services, and components that can be extended or customized.

## Overview

Factory patterns come in several forms:
- **Simple Factory**: Centralizes object creation in one class
- **Factory Method**: Subclasses decide which class to instantiate
- **Abstract Factory**: Creates families of related objects

## Core Structure

### Simple Factory

```php
<?php
// includes/Factory/ServiceFactory.php

namespace CustomModule\Factory;

class ServiceFactory
{
    protected array $bindings = [];

    /**
     * Register a service binding
     */
    public function bind(string $abstract, string $concrete): void
    {
        $this->bindings[$abstract] = $concrete;
    }

    /**
     * Create a service instance
     */
    public function make(string $abstract, array $parameters = []): object
    {
        if (!isset($this->bindings[$abstract])) {
            throw new \Exception("No binding found for: {$abstract}");
        }

        $concrete = $this->bindings[$abstract];

        if (!class_exists($concrete)) {
            throw new \Exception("Class not found: {$concrete}");
        }

        return $this->build($concrete, $parameters);
    }

    protected function build(string $concrete, array $parameters): object
    {
        $reflector = new \ReflectionClass($concrete);

        if (!$reflector->isInstantiable()) {
            throw new \Exception("Class is not instantiable: {$concrete}");
        }

        $constructor = $reflector->getConstructor();

        if (is_null($constructor)) {
            return new $concrete();
        }

        $dependencies = $this->resolveDependencies($constructor->getParameters(), $parameters);

        return $reflector->newInstanceArgs($dependencies);
    }

    protected function resolveDependencies(array $parameters, array $userParameters): array
    {
        $resolved = [];

        foreach ($parameters as $parameter) {
            $type = $parameter->getType();
            $name = $parameter->getName();

            if (isset($userParameters[$name])) {
                $resolved[] = $userParameters[$name];
            } elseif ($type && !$type->isBuiltin()) {
                $resolved[] = $this->make($type->getName());
            } elseif ($parameter->isDefaultValueAvailable()) {
                $resolved[] = $parameter->getDefaultValue();
            } else {
                throw new \Exception("Cannot resolve parameter: {$name}");
            }
        }

        return $resolved;
    }
}
```

## Real-World WHMCS Examples

### Module Factory

```php
<?php
// includes/Factory/ModuleFactory.php

namespace CustomModule\Factory;

use WHMCS\Module\Module;
use WHMCS\Database\Capsule;

class ModuleFactory
{
    protected array $modules = [];
    protected array $instances = [];

    /**
     * Register a module with its handler class
     */
    public function register(string $name, string $handlerClass, array $config = []): void
    {
        $this->modules[$name] = [
            'handler' => $handlerClass,
            'config' => $config,
            'created_at' => date('Y-m-d H:i:s')
        ];
    }

    /**
     * Get or create module instance (singleton)
     */
    public function make(string $name): Module
    {
        if (!isset($this->modules[$name])) {
            throw new \InvalidArgumentException("Module '{$name}' not registered");
        }

        if (!isset($this->instances[$name])) {
            $this->instances[$name] = $this->resolve($name);
        }

        return $this->instances[$name];
    }

    /**
     * Create a new instance (prototype pattern)
     */
    public function create(string $name): Module
    {
        if (!isset($this->modules[$name])) {
            throw new \InvalidArgumentException("Module '{$name}' not registered");
        }

        return $this->resolve($name);
    }

    /**
     * Get all registered module names
     */
    public function all(): array
    {
        return array_keys($this->modules);
    }

    /**
     * Check if module is registered
     */
    public function has(string $name): bool
    {
        return isset($this->modules[$name]);
    }

    protected function resolve(string $name): Module
    {
        $moduleConfig = $this->modules[$name];
        $handlerClass = $moduleConfig['handler'];

        // Get module configuration from database
        $dbConfig = $this->getModuleConfig($name);

        // Merge with registration config
        $config = array_merge($moduleConfig['config'], $dbConfig);

        return new $handlerClass($config);
    }

    protected function getModuleConfig(string $name): array
    {
        $config = Capsule::table('tbladdonmodules')
            ->where('module', $name)
            ->first();

        if (!$config) {
            return [];
        }

        // Decode any stored configuration
        return json_decode($config->settings ?? '{}', true) ?: [];
    }
}
```

### Payment Gateway Factory

```php
<?php
// includes/Factory/GatewayFactory.php

namespace CustomModule\Factory;

use WHMCS\Module\Gateway;
use CustomModule\Gateways\StripeGateway;
use CustomModule\Gateways\PayPalGateway;
use CustomModule\Gateways\SquareGateway;
use CustomModule\Interfaces\GatewayInterface;

class GatewayFactory
{
    protected static array $gateways = [];
    protected static array $instances = [];

    /**
     * Register gateway configuration
     */
    public static function register(string $name, array $config): void
    {
        self::$gateways[$name] = $config;
    }

    /**
     * Get gateway instance
     */
    public static function get(string $name): GatewayInterface
    {
        if (!isset(self::$gateways[$name])) {
            throw new \InvalidArgumentException("Gateway '{$name}' not found");
        }

        if (!isset(self::$instances[$name])) {
            self::$instances[$name] = self::resolve($name);
        }

        return self::$instances[$name];
    }

    /**
     * Get all available gateways
     */
    public static function all(): array
    {
        return self::$gateways;
    }

    /**
     * Check if gateway is enabled
     */
    public static function isEnabled(string $name): bool
    {
        return isset(self::$gateways[$name]) && (self::$gateways[$name]['enabled'] ?? false);
    }

    /**
     * Create gateway by type
     */
    public static function createByType(string $type, array $config): GatewayInterface
    {
        return match($type) {
            'stripe' => new StripeGateway($config),
            'paypal' => new PayPalGateway($config),
            'square' => new SquareGateway($config),
            default => throw new \InvalidArgumentException("Unknown gateway type: {$type}")
        };
    }

    protected static function resolve(string $name): GatewayInterface
    {
        $config = self::$gateways[$name];

        $gatewayClass = $config['class'] ?? self::getDefaultClass($name);
        $gatewayConfig = $config['config'] ?? [];

        return new $gatewayClass($gatewayConfig);
    }

    protected static function getDefaultClass(string $name): string
    {
        $defaults = [
            'stripe' => StripeGateway::class,
            'paypal' => PayPalGateway::class,
            'square' => SquareGateway::class
        ];

        return $defaults[$name] ?? Gateway::class;
    }
}

// Register gateways
GatewayFactory::register('stripe', [
    'class' => StripeGateway::class,
    'enabled' => true,
    'config' => [
        'api_key' => getenv('STRIPE_API_KEY'),
        'webhook_secret' => getenv('STRIPE_WEBHOOK_SECRET')
    ]
]);

GatewayFactory::register('paypal', [
    'class' => PayPalGateway::class,
    'enabled' => true,
    'config' => [
        'client_id' => getenv('PAYPAL_CLIENT_ID'),
        'client_secret' => getenv('PAYPAL_CLIENT_SECRET'),
        'mode' => 'sandbox'
    ]
]);
```

### Service Factory with Dependency Injection

```php
<?php
// includes/Factory/ServiceFactory.php

namespace CustomModule\Factory;

class ServiceFactory
{
    protected array $bindings = [];
    protected array $instances = [];
    protected array $tags = [];

    /**
     * Bind interface to implementation
     */
    public function bind(string $abstract, string $concrete, array $tags = []): self
    {
        $this->bindings[$abstract] = $concrete;

        foreach ($tags as $tag) {
            $this->tags[$tag][] = $abstract;
        }

        return $this;
    }

    /**
     * Register as singleton
     */
    public function singleton(string $abstract, string $concrete): self
    {
        $this->instances[$abstract] = null; // Will be populated on first make()
        return $this->bind($abstract, $concrete);
    }

    /**
     * Resolve service by tag
     */
    public function tagged(string $tag): array
    {
        if (!isset($this->tags[$tag])) {
            return [];
        }

        return array_map(
            fn($abstract) => $this->make($abstract),
            $this->tags[$tag]
        );
    }

    /**
     * Make service instance
     */
    public function make(string $abstract): object
    {
        if (isset($this->instances[$abstract]) && $this->instances[$abstract] !== null) {
            return $this->instances[$abstract];
        }

        if (!isset($this->bindings[$abstract])) {
            // Try to instantiate directly
            if (class_exists($abstract)) {
                return $this->build($abstract);
            }
            throw new \Exception("No binding for: {$abstract}");
        }

        $concrete = $this->bindings[$abstract];
        $instance = $this->build($concrete);

        // Store singleton if registered
        if (isset($this->instances[$abstract])) {
            $this->instances[$abstract] = $instance;
        }

        return $instance;
    }

    protected function build(string $concrete): object
    {
        $reflector = new \ReflectionClass($concrete);

        $constructor = $reflector->getConstructor();

        if ($constructor === null) {
            return new $concrete();
        }

        $dependencies = $this->resolveDependencies($constructor->getParameters());

        return $reflector->newInstanceArgs($dependencies);
    }

    protected function resolveDependencies(array $parameters): array
    {
        $dependencies = [];

        foreach ($parameters as $parameter) {
            $type = $parameter->getType();

            if ($type && !$type->isBuiltin() && $type->getName()) {
                $dependencies[] = $this->make($type->getName());
            } elseif ($parameter->isDefaultValueAvailable()) {
                $dependencies[] = $parameter->getDefaultValue();
            } else {
                $dependencies[] = null;
            }
        }

        return $dependencies;
    }

    /**
     * Reset all instances (for testing)
     */
    public function reset(): void
    {
        $this->instances = [];
    }
}
```

### Usage in WHMCS Module

```php
<?php
// module.php

use CustomModule\Factory\ModuleFactory;
use CustomModule\Factory\GatewayFactory;

// Initialize factory
$factory = new ModuleFactory();

// Register modules
$factory->register('inventory', InventoryModule::class, [
    'version' => '1.0.0',
    'author' => 'Custom Vendor'
]);

$factory->register('shipping', ShippingModule::class, [
    'version' => '1.0.0'
]);

// Get module instance
$inventory = $factory->make('inventory');

// Use gateway factory
$gateway = GatewayFactory::get('stripe');

// Process payment
$result = $gateway->charge(1000, 'USD', 'cus_xxx');
```

### Factory for Creating Reports

```php
<?php
// includes/Factory/ReportFactory.php

namespace CustomModule\Factory;

use CustomModule\Reports\ClientReport;
use CustomModule\Reports\InvoiceReport;
use CustomModule\Reports\ServiceReport;
use CustomModule\Reports\RevenueReport;

class ReportFactory
{
    protected static array $reportTypes = [
        'clients' => ClientReport::class,
        'invoices' => InvoiceReport::class,
        'services' => ServiceReport::class,
        'revenue' => RevenueReport::class
    ];

    /**
     * Create report by type
     */
    public static function create(string $type, array $options = []): ReportInterface
    {
        if (!isset(self::$reportTypes[$type])) {
            throw new \InvalidArgumentException("Report type '{$type}' not supported");
        }

        $reportClass = self::$reportTypes[$type];
        return new $reportClass($options);
    }

    /**
     * Create multiple reports
     */
    public static function createBatch(array $types, array $options = []): array
    {
        return array_map(
            fn($type) => self::create($type, $options),
            $types
        );
    }

    /**
     * Register custom report type
     */
    public static function register(string $type, string $reportClass): void
    {
        if (!is_subclass_of($reportClass, ReportInterface::class)) {
            throw new \InvalidArgumentException("Report class must implement ReportInterface");
        }

        self::$reportTypes[$type] = $reportClass;
    }
}

// Usage
$clientReport = ReportFactory::create('clients', [
    'date_from' => '2024-01-01',
    'date_to' => '2024-12-31',
    'group_by' => 'country'
]);
```

## Pros

- **Decoupling**: Client code doesn't need to know concrete class names
- **Flexibility**: Easy to swap implementations
- **Testability**: Easy to mock factories in tests
- **Centralization**: Object creation logic in one place
- **Extensibility**: Add new types without modifying existing code

## Cons

- **Complexity**: Additional layer for simple object creation
- **Abstraction**: Can hide important details from developers
- **Maintenance**: Factory must be updated when adding new types

## Best Practices

1. Use factories for complex object creation
2. Keep factory methods focused on single responsibility
3. Use constructor injection for dependencies
4. Consider using interfaces for better testability
5. Document expected parameters and return types