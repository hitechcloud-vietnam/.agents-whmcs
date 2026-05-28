# Strategy Pattern in WHMCS

The Strategy Pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. In WHMCS, this pattern is useful for handling different pricing models, tax calculations, shipping methods, and discount strategies.

## Overview

Strategy pattern allows you to:
- Define a family of algorithms
- Encapsulate each algorithm in its own class
- Make algorithms interchangeable
- Select algorithm at runtime
- Avoid complex conditionals

## Core Structure

### Strategy Interface

```php
<?php
// includes/Strategy/PricingStrategyInterface.php

namespace CustomModule\Strategy;

interface PricingStrategyInterface
{
    /**
     * Calculate price based on strategy
     */
    public function calculate(array $context): PriceResult;

    /**
     * Get strategy identifier
     */
    public function getName(): string;

    /**
     * Check if strategy is applicable
     */
    public function isApplicable(array $context): bool;
}
```

### Concrete Strategy

```php
<?php
// includes/Strategy/VolumeDiscountStrategy.php

namespace CustomModule\Strategy;

class VolumeDiscountStrategy implements PricingStrategyInterface
{
    protected array $tiers = [];
    protected float $baseDiscount = 0;

    public function __construct(array $tiers = [])
    {
        $this->tiers = $tiers;
    }

    public function calculate(array $context): PriceResult
    {
        $quantity = $context['quantity'] ?? 1;
        $unitPrice = $context['unit_price'] ?? 0;

        $discount = $this->getDiscountForQuantity($quantity);
        $totalPrice = $unitPrice * $quantity;
        $discountAmount = $totalPrice * $discount;
        $finalPrice = $totalPrice - $discountAmount;

        return new PriceResult(
            originalPrice: $totalPrice,
            finalPrice: $finalPrice,
            discount: $discountAmount,
            discountPercent: $discount * 100
        );
    }

    public function getName(): string
    {
        return 'volume_discount';
    }

    public function isApplicable(array $context): bool
    {
        $quantity = $context['quantity'] ?? 0;
        return $quantity >= ($this->tiers[0]['min_quantity'] ?? 1);
    }

    protected function getDiscountForQuantity(int $quantity): float
    {
        foreach ($this->tiers as $tier) {
            if ($quantity >= $tier['min_quantity']) {
                $this->baseDiscount = $tier['discount'];
            }
        }
        return $this->baseDiscount;
    }
}
```

## Real-World WHMCS Examples

### Tax Calculation Strategies

```php
<?php
// includes/Strategy/TaxStrategyInterface.php

namespace CustomModule\Strategy;

interface TaxStrategyInterface
{
    public function calculate(float $amount, array $context): TaxResult;
    public function getTaxRate(array $context): float;
    public function getTaxType(): string;
}
```

```php
<?php
// includes/Strategy/USSalesTaxStrategy.php

namespace CustomModule\Strategy;

use WHMCS\Database\Capsule;

class USSalesTaxStrategy implements TaxStrategyInterface
{
    protected array $stateRates = [];

    public function __construct()
    {
        $this->stateRates = $this->loadStateRates();
    }

    public function calculate(float $amount, array $context): TaxResult
    {
        $state = $context['state'] ?? '';
        $zipCode = $context['zip'] ?? '';
        $isBusiness = $context['is_business'] ?? false;

        $stateRate = $this->getStateRate($state);
        $localRate = $this->getLocalRate($zipCode, $state);

        // Business exemption in some states
        if ($isBusiness && $this->isBusinessExempt($state)) {
            return new TaxResult(
                taxAmount: 0,
                taxRate: 0,
                taxType: 'business_exempt'
            );
        }

        $totalRate = $stateRate + $localRate;
        $taxAmount = $amount * $totalRate;

        return new TaxResult(
            taxAmount: $taxAmount,
            taxRate: $totalRate,
            taxType: 'sales_tax',
            breakdown: [
                'state_rate' => $stateRate,
                'local_rate' => $localRate,
                'state' => $state
            ]
        );
    }

    public function getTaxRate(array $context): float
    {
        $state = $context['state'] ?? '';
        $zipCode = $context['zip'] ?? '';

        return $this->getStateRate($state) + $this->getLocalRate($zipCode, $state);
    }

    public function getTaxType(): string
    {
        return 'us_sales_tax';
    }

    protected function loadStateRates(): array
    {
        return [
            'CA' => 0.0725,
            'NY' => 0.08,
            'TX' => 0.0625,
            'FL' => 0.06,
            'WA' => 0.065,
            'OR' => 0.0,
            'MT' => 0.0,
            'NH' => 0.0,
            'DE' => 0.0,
            'AK' => 0.0
        ];
    }

    protected function getStateRate(string $state): float
    {
        return $this->stateRates[strtoupper($state)] ?? 0;
    }

    protected function getLocalRate(string $zipCode, string $state): float
    {
        // Fetch local tax rate based on zip
        $localRate = Capsule::table('mod_tax_local_rates')
            ->where('zip_prefix', substr($zipCode, 0, 3))
            ->where('state', $state)
            ->first();

        return $localRate->rate ?? 0;
    }

    protected function isBusinessExempt(string $state): bool
    {
        return in_array(strtoupper($state), ['TX', 'FL']);
    }
}
```

```php
<?php
// includes/Strategy/VATStrategy.php

namespace CustomModule\Strategy;

class VATStrategy implements TaxStrategyInterface
{
    protected float $standardRate;
    protected float $reducedRate;
    protected array $exemptCategories = [];

    public function __construct(float $standardRate = 0.20, float $reducedRate = 0.05)
    {
        $this->standardRate = $standardRate;
        $this->reducedRate = $reducedRate;
    }

    public function calculate(float $amount, array $context): TaxResult
    {
        $country = $context['country'] ?? 'GB';
        $category = $context['product_category'] ?? 'standard';
        $vatNumber = $context['vat_number'] ?? '';

        // B2B within EU - verify VAT number
        if (!empty($vatNumber) && $this->validateVATNumber($vatNumber)) {
            return new TaxResult(
                taxAmount: 0,
                taxRate: 0,
                taxType: 'vat_exempt_b2b',
                vatNumber: $vatNumber
            );
        }

        $rate = $this->getRateForCategory($category);
        $taxAmount = $amount * $rate;

        return new TaxResult(
            taxAmount: $taxAmount,
            taxRate: $rate,
            taxType: 'vat',
            country: $country
        );
    }

    public function getTaxRate(array $context): float
    {
        $category = $context['product_category'] ?? 'standard';
        return $this->getRateForCategory($category);
    }

    public function getTaxType(): string
    {
        return 'vat';
    }

    public function addExemptCategory(string $category): void
    {
        $this->exemptCategories[] = $category;
    }

    protected function getRateForCategory(string $category): float
    {
        if (in_array($category, $this->exemptCategories)) {
            return 0;
        }

        return match($category) {
            'reduced' => $this->reducedRate,
            'zero' => 0,
            default => $this->standardRate
        };
    }

    protected function validateVATNumber(string $vatNumber): bool
    {
        // Validate VAT number format and check with VIES
        return preg_match('/^[A-Z]{2}[0-9A-Z]{8,12}$/', $vatNumber) === 1;
    }
}
```

### Tax Strategy Context

```php
<?php
// includes/Strategy/TaxStrategyContext.php

namespace CustomModule\Strategy;

class TaxStrategyContext
{
    protected TaxStrategyInterface $strategy;

    public function __construct(TaxStrategyInterface $strategy)
    {
        $this->strategy = $strategy;
    }

    public function setStrategy(TaxStrategyInterface $strategy): void
    {
        $this->strategy = $strategy;
    }

    public function calculate(float $amount, array $context): TaxResult
    {
        return $this->strategy->calculate($amount, $context);
    }

    public function getStrategyName(): string
    {
        return $this->strategy->getTaxType();
    }
}
```

```php
<?php
// includes/Strategy/TaxStrategyFactory.php

namespace CustomModule\Strategy;

class TaxStrategyFactory
{
    public static function create(string $country): TaxStrategyInterface
    {
        return match(strtoupper($country)) {
            'US', 'CA' => new USSalesTaxStrategy(),
            'GB', 'DE', 'FR', 'IT', 'ES', 'NL', 'BE' => new VATStrategy(),
            'JP' => new JapanConsumptionTaxStrategy(),
            'AU' => new AustralianGSTStrategy(),
            default => new NoTaxStrategy()
        };
    }
}
```

### Shipping Calculation Strategies

```php
<?php
// includes/Strategy/ShippingStrategyInterface.php

namespace CustomModule\Strategy;

interface ShippingStrategyInterface
{
    public function calculate(ShippingRequest $request): ShippingResult;
    public function getName(): string;
    public function getEstimatedDays(ShippingRequest $request): int;
}
```

```php
<?php
// includes/Strategy/FlatRateShippingStrategy.php

namespace CustomModule\Strategy;

class FlatRateShippingStrategy implements ShippingStrategyInterface
{
    protected float $rate;
    protected array $zones = [];

    public function __construct(float $rate = 9.99, array $zones = [])
    {
        $this->rate = $rate;
        $this->zones = $zones;
    }

    public function calculate(ShippingRequest $request): ShippingResult
    {
        $zone = $this->determineZone($request->getCountry(), $request->getZipCode());
        $rate = $this->getRateForZone($zone);

        // Free shipping threshold
        if ($request->getSubtotal() >= $this->freeShippingThreshold) {
            $rate = 0;
        }

        return new ShippingResult(
            cost: $rate,
            carrier: 'FlatRate',
            service: 'Standard',
            estimatedDays: $this->getEstimatedDays($request)
        );
    }

    public function getName(): string
    {
        return 'flat_rate';
    }

    public function getEstimatedDays(ShippingRequest $request): int
    {
        return 5;
    }

    protected function determineZone(string $country, string $zip): string
    {
        return $country === 'US' ? 'domestic' : 'international';
    }

    protected function getRateForZone(string $zone): float
    {
        return $this->zones[$zone] ?? $this->rate;
    }
}
```

```php
<?php
// includes/Strategy/WeightBasedShippingStrategy.php

namespace CustomModule\Strategy;

class WeightBasedShippingStrategy implements ShippingStrategyInterface
{
    protected array $rateTables = [];

    public function __construct(array $rateTables = [])
    {
        $this->rateTables = $rateTables;
    }

    public function calculate(ShippingRequest $request): ShippingResult
    {
        $weight = $request->getTotalWeight();
        $country = $request->getCountry();
        $zone = $this->getZoneForCountry($country);

        $baseRate = $this->getBaseRateForZone($zone);
        $weightSurcharge = $this->calculateWeightSurcharge($weight, $zone);

        $totalCost = $baseRate + $weightSurcharge;

        return new ShippingResult(
            cost: $totalCost,
            carrier: 'Standard',
            service: 'WeightBased',
            estimatedDays: $this->getEstimatedDays($request)
        );
    }

    public function getName(): string
    {
        return 'weight_based';
    }

    public function getEstimatedDays(ShippingRequest $request): int
    {
        $zone = $this->getZoneForCountry($request->getCountry());
        return match($zone) {
            'domestic' => 3,
            'north_america' => 5,
            'europe' => 7,
            'asia' => 10,
            default => 14
        };
    }

    protected function getZoneForCountry(string $country): string
    {
        if ($country === 'US') return 'domestic';
        if (in_array($country, ['CA', 'MX'])) return 'north_america';
        if (in_array($country, ['GB', 'DE', 'FR', 'IT', 'ES', 'NL'])) return 'europe';
        if (in_array($country, ['JP', 'CN', 'KR', 'AU'])) return 'asia';
        return 'other';
    }

    protected function getBaseRateForZone(string $zone): float
    {
        return $this->rateTables[$zone]['base_rate'] ?? 10.00;
    }

    protected function calculateWeightSurcharge(float $weight, string $zone): float
    {
        $table = $this->rateTables[$zone] ?? [];
        $ratePerLb = $table['rate_per_lb'] ?? 0.50;

        return $weight * $ratePerLb;
    }
}
```

### Strategy Selection in Module

```php
<?php
// includes/Services/OrderProcessor.php

namespace CustomModule\Services;

use CustomModule\Strategy\TaxStrategyFactory;
use CustomModule\Strategy\ShippingStrategyFactory;
use CustomModule\Strategy\PricingStrategyFactory;

class OrderProcessor
{
    public function processOrder(array $orderData): array
    {
        // Determine tax strategy
        $taxStrategy = TaxStrategyFactory::create($orderData['country']);
        $taxContext = new TaxStrategyContext($taxStrategy);

        // Calculate tax
        $subtotal = $this->calculateSubtotal($orderData);
        $taxResult = $taxContext->calculate($subtotal, [
            'country' => $orderData['country'],
            'state' => $orderData['state'] ?? '',
            'zip' => $orderData['zip'] ?? '',
            'vat_number' => $orderData['vat_number'] ?? ''
        ]);

        // Determine shipping strategy
        $shippingStrategy = $this->getShippingStrategy($orderData);
        $shippingRequest = new ShippingRequest(
            items: $orderData['items'],
            country: $orderData['country'],
            zip: $orderData['zip']
        );
        $shippingResult = $shippingStrategy->calculate($shippingRequest);

        // Apply pricing strategy (discounts)
        $pricingStrategy = PricingStrategyFactory::get(
            $orderData['pricing_type'] ?? 'standard'
        );
        $priceResult = $pricingStrategy->calculate([
            'quantity' => count($orderData['items']),
            'unit_price' => $subtotal,
            'customer_tier' => $orderData['customer_tier'] ?? 'standard'
        ]);

        $total = $priceResult->finalPrice + $taxResult->taxAmount + $shippingResult->cost;

        return [
            'subtotal' => $priceResult->finalPrice,
            'discount' => $priceResult->discount,
            'tax' => $taxResult->taxAmount,
            'shipping' => $shippingResult->cost,
            'total' => $total,
            'currency' => $orderData['currency'] ?? 'USD'
        ];
    }

    protected function getShippingStrategy(array $orderData): ShippingStrategyInterface
    {
        return match($orderData['shipping_method'] ?? 'flat') {
            'flat' => new FlatRateShippingStrategy(),
            'weight' => new WeightBasedShippingStrategy(),
            'express' => new ExpressShippingStrategy(),
            default => new FlatRateShippingStrategy()
        };
    }

    protected function calculateSubtotal(array $orderData): float
    {
        return array_reduce($orderData['items'], function($sum, $item) {
            return $sum + ($item['price'] * $item['quantity']);
        }, 0);
    }
}
```

## Pros

- **Flexibility**: Swap algorithms at runtime
- **Single Responsibility**: Each strategy handles one algorithm
- **Testability**: Test strategies in isolation
- **Open/Closed**: Add new strategies without modifying existing code
- **Decoupling**: Consumer doesn't know about algorithm details

## Cons

- **Complexity**: More classes to maintain
- **Overhead**: Strategy objects add memory usage
- **Client Awareness**: Client must understand strategy differences
- **Overkill**: Too many strategies can be confusing

## Best Practices

1. Use strategy when you have multiple ways to accomplish a task
2. Keep strategy interfaces stable
3. Consider using factory pattern for strategy selection
4. Document the trade-offs between strategies
5. Use composition for strategies that need shared state