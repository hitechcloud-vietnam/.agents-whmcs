# Composite Pattern in WHMCS

The Composite Pattern composes objects into tree structures to represent part-whole hierarchies. In WHMCS, this pattern is useful for building hierarchical structures like product bundles, service packages, permission systems, and menu structures.

## Overview

Composite pattern lets clients treat individual objects and compositions of objects uniformly:
- Create tree structures of objects
- Work with complex structures as single objects
- Uniform interface for leaves and composites
- Simplifies client code
- Makes adding new component types easy

## Core Structure

### Component Interface

```php
<?php
// includes/Composite/ComponentInterface.php

namespace CustomModule\Composite;

interface ComponentInterface
{
    public function getName(): string;
    public function getPrice(): float;
    public function getChildren(): array;
    public function isLeaf(): bool;
    public function accept(VisitorInterface $visitor): mixed;
}
```

## Real-World WHMCS Examples

### Product Bundle Composite

```php
<?php
// includes/Composite/ProductComponent.php

namespace CustomModule\Composite;

abstract class ProductComponent implements ComponentInterface
{
    protected string $name;
    protected string $sku;
    protected array $metadata = [];

    public function getName(): string
    {
        return $this->name;
    }

    public function getSku(): string
    {
        return $this->sku;
    }

    public function getMetadata(string $key, $default = null)
    {
        return $this->metadata[$key] ?? $default;
    }

    protected function setMetadata(string $key, $value): void
    {
        $this->metadata[$key] = $value;
    }
}
```

```php
<?php
// includes/Composite/ProductLeaf.php

namespace CustomModule\Composite;

class ProductLeaf extends ProductComponent
{
    protected float $price;
    protected float $setupFee;
    protected int $stockQuantity = -1; // -1 = unlimited

    public function __construct(string $name, string $sku, float $price)
    {
        $this->name = $name;
        $this->sku = $sku;
        $this->price = $price;
    }

    public function getPrice(): float
    {
        return $this->price;
    }

    public function setSetupFee(float $fee): self
    {
        $this->setupFee = $fee;
        return $this;
    }

    public function getSetupFee(): float
    {
        return $this->setupFee ?? 0;
    }

    public function getChildren(): array
    {
        return [];
    }

    public function isLeaf(): bool
    {
        return true;
    }

    public function accept(VisitorInterface $visitor): mixed
    {
        return $visitor->visitLeaf($this);
    }

    public function hasStock(int $quantity = 1): bool
    {
        if ($this->stockQuantity < 0) {
            return true;
        }
        return $this->stockQuantity >= $quantity;
    }

    public function reserveStock(int $quantity): bool
    {
        if (!$this->hasStock($quantity)) {
            return false;
        }
        $this->stockQuantity -= $quantity;
        return true;
    }
}
```

```php
<?php
// includes/Composite/ProductComposite.php

namespace CustomModule\Composite;

class ProductComposite extends ProductComponent
{
    protected array $children = [];
    protected bool $allowPartialSelection = true;
    protected ?float $bundleDiscount = null;

    public function __construct(string $name, string $sku)
    {
        $this->name = $name;
        $this->sku = $sku;
    }

    public function add(ProductComponent $component): self
    {
        $this->children[$component->getSku()] = $component;
        return $this;
    }

    public function remove(string $sku): self
    {
        unset($this->children[$sku]);
        return $this;
    }

    public function getChild(string $sku): ?ProductComponent
    {
        return $this->children[$sku] ?? null;
    }

    public function getChildren(): array
    {
        return array_values($this->children);
    }

    public function isLeaf(): bool
    {
        return false;
    }

    public function getPrice(): float
    {
        $total = 0;
        foreach ($this->children as $child) {
            $total += $child->getPrice();
        }

        // Apply bundle discount if set
        if ($this->bundleDiscount !== null) {
            return $total * (1 - $this->bundleDiscount);
        }

        return $total;
    }

    public function getTotalPrice(): float
    {
        return $this->getPrice();
    }

    public function getSetupFees(): float
    {
        $total = 0;
        foreach ($this->children as $child) {
            if ($child instanceof ProductLeaf) {
                $total += $child->getSetupFee();
            } elseif ($child instanceof ProductComposite) {
                $total += $child->getSetupFees();
            }
        }
        return $total;
    }

    public function setBundleDiscount(float $discountPercent): self
    {
        $this->bundleDiscount = $discountPercent / 100;
        return $this;
    }

    public function setAllowPartialSelection(bool $allow): self
    {
        $this->allowPartialSelection = $allow;
        return $this;
    }

    public function isAllowPartialSelection(): bool
    {
        return $this->allowPartialSelection;
    }

    public function getAllLeafProducts(): array
    {
        $leaves = [];
        foreach ($this->children as $child) {
            if ($child instanceof ProductLeaf) {
                $leaves[] = $child;
            } elseif ($child instanceof ProductComposite) {
                $leaves = array_merge($leaves, $child->getAllLeafProducts());
            }
        }
        return $leaves;
    }

    public function accept(VisitorInterface $visitor): mixed
    {
        return $visitor->visitComposite($this);
    }
}
```

### Service Configuration Composite

```php
<?php
// includes/Composite/ServiceConfiguration.php

namespace CustomModule\Composite;

class ServiceConfiguration implements ComponentInterface
{
    protected string $name;
    protected array $children = [];

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function getPrice(): float
    {
        $total = 0;
        foreach ($this->children as $child) {
            $total += $child->getPrice();
        }
        return $total;
    }

    public function getChildren(): array
    {
        return array_values($this->children);
    }

    public function isLeaf(): bool
    {
        return empty($this->children);
    }

    public function addConfig(ConfigOption $config): self
    {
        $this->children[$config->getKey()] = $config;
        return $this;
    }

    public function removeConfig(string $key): self
    {
        unset($this->children[$key]);
        return $this;
    }

    public function getConfig(string $key): ?ConfigOption
    {
        return $this->children[$key] ?? null;
    }

    public function getAllConfigs(): array
    {
        return $this->getChildren();
    }

    public function accept(VisitorInterface $visitor): mixed
    {
        return $visitor->visitServiceConfiguration($this);
    }
}
```

```php
<?php
// includes/Composite/ConfigOption.php

namespace CustomModule\Composite;

class ConfigOption implements ComponentInterface
{
    protected string $name;
    protected string $key;
    protected float $price;
    protected string $type; // 'radio', 'dropdown', 'checkbox', 'quantity'
    protected array $options = [];
    protected bool $selected = false;
    protected int $quantity = 1;

    public function __construct(string $name, string $key, float $price, string $type = 'checkbox')
    {
        $this->name = $name;
        $this->key = $key;
        $this->price = $price;
        $this->type = $type;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function getKey(): string
    {
        return $this->key;
    }

    public function getPrice(): float
    {
        if ($this->type === 'quantity') {
            return $this->price * $this->quantity;
        }
        return $this->selected ? $this->price : 0;
    }

    public function getChildren(): array
    {
        return [];
    }

    public function isLeaf(): bool
    {
        return true;
    }

    public function isSelected(): bool
    {
        return $this->selected;
    }

    public function setSelected(bool $selected): self
    {
        $this->selected = $selected;
        return $this;
    }

    public function setQuantity(int $quantity): self
    {
        $this->quantity = max(1, $quantity);
        return $this;
    }

    public function getQuantity(): int
    {
        return $this->quantity;
    }

    public function addOption(string $label, $value): self
    {
        $this->options[$value] = $label;
        return $this;
    }

    public function getOptions(): array
    {
        return $this->options;
    }

    public function accept(VisitorInterface $visitor): mixed
    {
        return $visitor->visitConfigOption($this);
    }
}
```

### Visitor Pattern Implementation

```php
<?php
// includes/Composite/VisitorInterface.php

namespace CustomModule\Composite;

interface VisitorInterface
{
    public function visitLeaf(ProductLeaf $leaf): mixed;
    public function visitComposite(ProductComposite $composite): mixed;
    public function visitConfigOption(ConfigOption $config): mixed;
    public function visitServiceConfiguration(ServiceConfiguration $config): mixed;
}
```

```php
<?php
// includes/Composite/PricingVisitor.php

namespace CustomModule\Composite;

class PricingVisitor implements VisitorInterface
{
    protected array $prices = [];

    public function visitLeaf(ProductLeaf $leaf): array
    {
        $this->prices[$leaf->getSku()] = [
            'name' => $leaf->getName(),
            'price' => $leaf->getPrice(),
            'setup_fee' => $leaf->getSetupFee()
        ];
        return $this->prices;
    }

    public function visitComposite(ProductComposite $composite): array
    {
        foreach ($composite->getChildren() as $child) {
            $child->accept($this);
        }
        return $this->prices;
    }

    public function visitConfigOption(ConfigOption $config): array
    {
        $this->prices[$config->getKey()] = [
            'name' => $config->getName(),
            'price' => $config->getPrice(),
            'type' => $config->isSelected() ? 'included' : 'optional'
        ];
        return $this->prices;
    }

    public function visitServiceConfiguration(ServiceConfiguration $config): array
    {
        foreach ($config->getChildren() as $child) {
            $child->accept($this);
        }
        return $this->prices;
    }

    public function getTotalPrice(): float
    {
        return array_sum(array_column($this->prices, 'price'));
    }

    public function getTotalSetupFees(): float
    {
        return array_sum(array_column($this->prices, 'setup_fee'));
    }
}
```

```php
<?php
// includes/Composite/ValidationVisitor.php

namespace CustomModule\Composite;

class ValidationVisitor implements VisitorInterface
{
    protected array $errors = [];
    protected array $warnings = [];

    public function visitLeaf(ProductLeaf $leaf): void
    {
        if (!$leaf->hasStock()) {
            $this->errors[] = "Product '{$leaf->getName()}' is out of stock";
        }
    }

    public function visitComposite(ProductComposite $composite): void
    {
        $children = $composite->getChildren();

        if (empty($children)) {
            $this->warnings[] = "Bundle '{$composite->getName()}' has no products";
        }

        foreach ($children as $child) {
            $child->accept($this);
        }
    }

    public function visitConfigOption(ConfigOption $config): void
    {
        // Validate config options
    }

    public function visitServiceConfiguration(ServiceConfiguration $config): void
    {
        foreach ($config->getChildren() as $child) {
            $child->accept($this);
        }
    }

    public function getErrors(): array
    {
        return $this->errors;
    }

    public function getWarnings(): array
    {
        return $this->warnings;
    }

    public function isValid(): bool
    {
        return empty($this->errors);
    }
}
```

### Menu Structure Composite

```php
<?php
// includes/Composite/MenuComponent.php

namespace CustomModule\Composite;

class MenuComponent implements ComponentInterface
{
    protected string $name;
    protected string $route;
    protected string $icon;
    protected array $children = [];
    protected bool $visible = true;
    protected ?string $permission = null;
    protected int $sortOrder = 0;

    public function __construct(string $name, string $route = '')
    {
        $this->name = $name;
        $this->route = $route;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function getRoute(): string
    {
        return $this->route;
    }

    public function getIcon(): string
    {
        return $this->icon;
    }

    public function setIcon(string $icon): self
    {
        $this->icon = $icon;
        return $this;
    }

    public function getPrice(): float
    {
        return 0; // Not applicable for menu
    }

    public function getChildren(): array
    {
        return array_values($this->children);
    }

    public function isLeaf(): bool
    {
        return empty($this->children);
    }

    public function addChild(MenuComponent $child): self
    {
        $this->children[$child->getName()] = $child;
        return $this;
    }

    public function removeChild(string $name): self
    {
        unset($this->children[$name]);
        return $this;
    }

    public function setVisible(bool $visible): self
    {
        $this->visible = $visible;
        return $this;
    }

    public function isVisible(): bool
    {
        return $this->visible;
    }

    public function setPermission(?string $permission): self
    {
        $this->permission = $permission;
        return $this;
    }

    public function hasPermission(): bool
    {
        if ($this->permission === null) {
            return true;
        }
        return $this->checkPermission($this->permission);
    }

    protected function checkPermission(string $permission): bool
    {
        // Check against WHMCS permissions
        return has_permission($permission);
    }

    public function setSortOrder(int $order): self
    {
        $this->sortOrder = $order;
        return $this;
    }

    public function getSortOrder(): int
    {
        return $this->sortOrder;
    }

    public function accept(VisitorInterface $visitor): mixed
    {
        return $visitor->visitMenu($this);
    }

    public function toArray(): array
    {
        $result = [
            'name' => $this->name,
            'route' => $this->route,
            'icon' => $this->icon,
            'visible' => $this->visible,
            'permission' => $this->permission
        ];

        if (!empty($this->children)) {
            $result['children'] = array_map(
                fn($child) => $child->toArray(),
                $this->getChildren()
            );
        }

        return $result;
    }
}
```

### Usage Example

```php
<?php
// Creating a product bundle

$bundle = new ProductComposite('Enterprise Suite', 'ENT-001');
$bundle->setBundleDiscount(15); // 15% off when buying together

// Add individual products
$cloudStorage = new ProductLeaf('Cloud Storage 100GB', 'STO-100', 9.99);
$cloudStorage->setSetupFee(0);
$bundle->add($cloudStorage);

$emailService = new ProductLeaf('Business Email', 'EML-001', 4.99);
$emailService->setSetupFee(2.99);
$bundle->add($emailService);

// Add another composite (sub-bundle)
$securityBundle = new ProductComposite('Security Pack', 'SEC-001');
$securityBundle->add(new ProductLeaf('Antivirus', 'ANT-001', 5.99));
$securityBundle->add(new ProductLeaf('VPN Access', 'VPN-001', 7.99));
$bundle->add($securityBundle);

// Calculate total price
$pricingVisitor = new PricingVisitor();
$bundle->accept($pricingVisitor);

echo "Bundle Price: $" . number_format($pricingVisitor->getTotalPrice(), 2);
echo "Setup Fees: $" . number_format($pricingVisitor->getTotalSetupFees(), 2);
```

### Permission Hierarchy Composite

```php
<?php
// includes/Composite/PermissionGroup.php

namespace CustomModule\Composite;

class PermissionGroup implements ComponentInterface
{
    protected string $name;
    protected string $description;
    protected array $children = [];
    protected array $permissions = [];

    public function __construct(string $name, string $description = '')
    {
        $this->name = $name;
        $this->description = $description;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function getDescription(): string
    {
        return $this->description;
    }

    public function getPrice(): float
    {
        return 0;
    }

    public function getChildren(): array
    {
        return array_values($this->children);
    }

    public function isLeaf(): bool
    {
        return empty($this->children);
    }

    public function addGroup(PermissionGroup $group): self
    {
        $this->children[$group->getName()] = $group;
        return $this;
    }

    public function addPermission(string $permission): self
    {
        $this->permissions[] = $permission;
        return $this;
    }

    public function getAllPermissions(): array
    {
        $allPermissions = $this->permissions;

        foreach ($this->children as $child) {
            if ($child instanceof PermissionGroup) {
                $allPermissions = array_merge($allPermissions, $child->getAllPermissions());
            }
        }

        return array_unique($allPermissions);
    }

    public function hasPermission(string $permission): bool
    {
        return in_array($permission, $this->getAllPermissions());
    }

    public function accept(VisitorInterface $visitor): mixed
    {
        return $visitor->visitPermissionGroup($this);
    }
}
```

## Pros

- **Uniformity**: Treat individual objects and compositions the same
- **Simplicity**: Simplifies client code dealing with tree structures
- **Extensibility**: Easy to add new component types
- **Flexibility**: Build complex structures dynamically

## Cons

- **Overgeneralization**: May make design overly general
- **Complexity**: Can create complex hierarchies
- **Type Safety**: Less type-safe than explicit hierarchies

## Best Practices

1. Define clear component interfaces
2. Keep leaf and composite responsibilities separate
3. Implement visitor pattern for operations on complex trees
4. Consider caching computed values on composites
5. Validate tree structure to prevent cycles