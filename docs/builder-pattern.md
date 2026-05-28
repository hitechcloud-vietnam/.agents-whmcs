# Builder Pattern in WHMCS

The Builder Pattern separates the construction of a complex object from its representation. In WHMCS, this pattern is excellent for creating complex entities like invoices, orders, or reports that require multiple steps and optional parameters.

## Overview

Builder pattern allows you to:
- Construct complex objects step by step
- Support multiple representations of the same construction process
- Create different variations of an object
- Separate construction from representation
- Improve readability of complex object creation

## Core Structure

### Simple Builder

```php
<?php
// includes/Builder/BuilderInterface.php

namespace CustomModule\Builder;

interface BuilderInterface
{
    public function reset(): self;
    public function build(): object;
}
```

## Real-World WHMCS Examples

### Invoice Builder

```php
<?php
// includes/Builder/InvoiceBuilder.php

namespace CustomModule\Builder;

use WHMCS\Database\Capsule;
use Carbon\Carbon;

class InvoiceBuilder implements BuilderInterface
{
    protected ?int $clientId = null;
    protected string $status = 'Draft';
    protected ?string $dueDate = null;
    protected ?string $invoiceDate = null;
    protected ?string $paymentMethod = null;
    protected array $lineItems = [];
    protected array $customFields = [];
    protected ?int $parentInvoiceId = null;
    protected ?string $notes = null;
    protected ?string $taxZone = null;

    public function reset(): self
    {
        $this->clientId = null;
        $this->status = 'Draft';
        $this->dueDate = null;
        $this->invoiceDate = null;
        $this->paymentMethod = null;
        $this->lineItems = [];
        $this->customFields = [];
        $this->parentInvoiceId = null;
        $this->notes = null;
        $this->taxZone = null;

        return $this;
    }

    public function forClient(int $clientId): self
    {
        $this->clientId = $clientId;
        return $this;
    }

    public function withStatus(string $status): self
    {
        $this->status = $status;
        return $this;
    }

    public function dueInDays(int $days): self
    {
        $this->dueDate = Carbon::now()->addDays($days)->format('Y-m-d');
        return $this;
    }

    public function dueOn(string $date): self
    {
        $this->dueDate = Carbon::parse($date)->format('Y-m-d');
        return $this;
    }

    public function invoiceDate(string $date): self
    {
        $this->invoiceDate = Carbon::parse($date)->format('Y-m-d');
        return $this;
    }

    public function usingPaymentMethod(string $paymentMethod): self
    {
        $this->paymentMethod = $paymentMethod;
        return $this;
    }

    public function addItem(string $description, float $amount, int $quantity = 1, array $options = []): self
    {
        $this->lineItems[] = [
            'description' => $description,
            'amount' => $amount,
            'quantity' => $quantity,
            'taxed' => $options['taxed'] ?? true,
            'taxrate' => $options['taxrate'] ?? 0,
            'type' => $options['type'] ?? 'Item',
            'relid' => $options['relid'] ?? 0
        ];
        return $this;
    }

    public function addTaxedItem(string $description, float $amount, float $taxRate): self
    {
        return $this->addItem($description, $amount, 1, ['taxed' => true, 'taxrate' => $taxRate]);
    }

    public function addDiscountItem(string $description, float $amount): self
    {
        return $this->addItem($description, -abs($amount), 1, ['taxed' => false, 'type' => 'Discount']);
    }

    public function setNotes(string $notes): self
    {
        $this->notes = $notes;
        return $this;
    }

    public function setCustomField(string $name, $value): self
    {
        $this->customFields[$name] = $value;
        return $this;
    }

    public function setTaxZone(string $taxZone): self
    {
        $this->taxZone = $taxZone;
        return $this;
    }

    public function isRecurringFrom(int $parentInvoiceId): self
    {
        $this->parentInvoiceId = $parentInvoiceId;
        return $this;
    }

    public function build(): Invoice
    {
        // Validate required fields
        if ($this->clientId === null) {
            throw new \InvalidArgumentException('Client ID is required');
        }

        // Create the invoice
        $invoiceId = Capsule::table('tblinvoices')->insertGetId([
            'userid' => $this->clientId,
            'invoicenum' => $this->generateInvoiceNumber(),
            'date' => $this->invoiceDate ?? date('Y-m-d'),
            'duedate' => $this->dueDate ?? Carbon::now()->addDays(14)->format('Y-m-d'),
            'status' => $this->status,
            'paymentmethod' => $this->paymentMethod ?? 'invoice',
            'notes' => $this->notes,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        // Add line items
        foreach ($this->lineItems as $item) {
            Capsule::table('tblinvoiceitems')->insert([
                'invoiceid' => $invoiceId,
                'userid' => $this->clientId,
                'description' => $item['description'],
                'amount' => $item['amount'],
                'quantity' => $item['quantity'],
                'taxed' => $item['taxed'] ? 1 : 0,
                'taxrate' => $item['taxrate'],
                'type' => $item['type'],
                'relid' => $item['relid']
            ]);
        }

        // Set custom fields
        foreach ($this->customFields as $name => $value) {
            $this->setInvoiceCustomField($invoiceId, $name, $value);
        }

        // Handle recurring invoice link
        if ($this->parentInvoiceId) {
            Capsule::table('mod_recurring_links')->insert([
                'parent_invoice_id' => $this->parentInvoiceId,
                'child_invoice_id' => $invoiceId,
                'created_at' => date('Y-m-d H:i:s')
            ]);
        }

        // Calculate totals
        $this->calculateInvoiceTotals($invoiceId);

        return new Invoice($invoiceId);
    }

    protected function generateInvoiceNumber(): string
    {
        $prefix = Capsule::table('tblinvoiceconfig')->first()->prefix ?? 'INV';
        $lastNumber = Capsule::table('tblinvoiceconfig')->first()->last_number ?? 0;

        return $prefix . '-' . str_pad($lastNumber + 1, 6, '0', STR_PAD_LEFT);
    }

    protected function setInvoiceCustomField(int $invoiceId, string $name, $value): void
    {
        $field = Capsule::table('tblcustomfields')
            ->where('type', 'invoice')
            ->where('fieldname', $name)
            ->first();

        if ($field) {
            Capsule::table('tblcustomfieldsvalues')->updateOrInsert(
                ['relid' => $invoiceId, 'fieldid' => $field->id],
                ['value' => is_array($value) ? json_encode($value) : $value]
            );
        }
    }

    protected function calculateInvoiceTotals(int $invoiceId): void
    {
        $items = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoiceId)
            ->get();

        $subtotal = 0;
        $tax = 0;

        foreach ($items as $item) {
            $itemTotal = $item->amount * $item->quantity;
            $subtotal += $itemTotal;

            if ($item->taxed) {
                $tax += $itemTotal * ($item->taxrate / 100);
            }
        }

        Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->update([
                'subtotal' => $subtotal,
                'tax' => $tax,
                'total' => $subtotal + $tax
            ]);
    }
}
```

### Usage Example

```php
<?php
// Creating an invoice with the builder

$invoiceBuilder = new InvoiceBuilder();

$invoice = $invoiceBuilder
    ->forClient(123)
    ->withStatus('Draft')
    ->dueInDays(30)
    ->addTaxedItem('Web Hosting - Monthly', 29.99, 10)
    ->addItem('Domain Registration (.com)', 12.99, 1, ['taxed' => true, 'taxrate' => 0])
    ->addDiscountItem('First Month Discount', 5.00)
    ->setNotes('Thank you for your business!')
    ->setCustomField('po_number', 'PO-2024-1234')
    ->build();
```

### Order Builder

```php
<?php
// includes/Builder/OrderBuilder.php

namespace CustomModule\Builder;

use WHMCS\Database\Capsule;

class OrderBuilder implements BuilderInterface
{
    protected int $clientId;
    protected array $items = [];
    protected array $addons = [];
    protected ?string $orderStatus = null;
    protected ?string $paymentMethod = null;
    protected ?string $promoCode = null;
    protected ?string $orderNotes = null;
    protected array $configOptions = [];
    protected bool $sendEmail = true;
    protected bool $autoSetup = true;
    protected bool $autoDomain = true;

    public function reset(): self
    {
        $this->clientId = 0;
        $this->items = [];
        $this->addons = [];
        $this->orderStatus = null;
        $this->paymentMethod = null;
        $this->promoCode = null;
        $this->orderNotes = null;
        $this->configOptions = [];
        $this->sendEmail = true;
        $this->autoSetup = true;
        $this->autoDomain = true;

        return $this;
    }

    public function forClient(int $clientId): self
    {
        $this->clientId = $clientId;
        return $this;
    }

    public function addProduct(int $productId, array $config = []): self
    {
        $this->items[] = [
            'type' => 'product',
            'id' => $productId,
            'domain' => $config['domain'] ?? null,
            'billingcycle' => $config['billingcycle'] ?? 'monthly',
            'configoptions' => $config['configoptions'] ?? [],
            'customfields' => $config['customfields'] ?? [],
            'pricingoverride' => $config['price'] ?? null
        ];

        return $this;
    }

    public function addService(int $serviceId, array $config = []): self
    {
        $this->items[] = [
            'type' => 'service',
            'id' => $serviceId,
            'configoptions' => $config['configoptions'] ?? []
        ];

        return $this;
    }

    public function addAddon(int $addonId, int $hostingId = null, array $config = []): self
    {
        $this->addons[] = [
            'addon_id' => $addonId,
            'hosting_id' => $hostingId,
            'configoptions' => $config['configoptions'] ?? [],
            'qty' => $config['qty'] ?? 1
        ];

        return $this;
    }

    public function withStatus(string $status): self
    {
        $this->orderStatus = $status;
        return $this;
    }

    public function withPaymentMethod(string $method): self
    {
        $this->paymentMethod = $method;
        return $this;
    }

    public function applyPromoCode(string $code): self
    {
        $this->promoCode = $code;
        return $this;
    }

    public function withNotes(string $notes): self
    {
        $this->orderNotes = $notes;
        return $this;
    }

    public function skipEmail(): self
    {
        $this->sendEmail = false;
        return $this;
    }

    public function skipAutoSetup(): self
    {
        $this->autoSetup = false;
        return $this;
    }

    public function skipAutoDomain(): self
    {
        $this->autoDomain = false;
        return $this;
    }

    public function build(): array
    {
        // Validate
        if ($this->clientId === 0) {
            throw new \InvalidArgumentException('Client ID is required');
        }

        if (empty($this->items)) {
            throw new \InvalidArgumentException('At least one item is required');
        }

        // Apply promo code if provided
        $discount = $this->applyPromoCode();

        // Calculate totals
        $totals = $this->calculateTotals();

        // Create order
        $orderId = Capsule::table('tblorders')->insertGetId([
            'userid' => $this->clientId,
            'ordernum' => $this->generateOrderNumber(),
            'date' => date('Y-m-d H:i:s'),
            'status' => $this->orderStatus ?? 'Pending',
            'paymentmethod' => $this->paymentMethod ?? 'invoice',
            'ipaddress' => $_SERVER['REMOTE_ADDR'] ?? '',
            'notes' => $this->orderNotes,
            'promocode' => $this->promoCode,
            'promovalue' => $discount,
            'subtotal' => $totals['subtotal'],
            'discount' => $discount,
            'total' => $totals['total'],
            'sendemail' => $this->sendEmail ? 1 : 0
        ]);

        // Add order items
        foreach ($this->items as $item) {
            $this->addOrderItem($orderId, $item);
        }

        // Add order addons
        foreach ($this->addons as $addon) {
            $this->addOrderAddon($orderId, $addon);
        }

        // Handle automatic setup
        if ($this->autoSetup) {
            $this->processAutoSetup($orderId);
        }

        return [
            'order_id' => $orderId,
            'totals' => $totals,
            'items' => count($this->items),
            'addons' => count($this->addons)
        ];
    }

    protected function applyPromoCode(): float
    {
        if (!$this->promoCode) {
            return 0;
        }

        $promo = Capsule::table('tblpromotions')
            ->where('code', $this->promoCode)
            ->where('startdate', '<=', date('Y-m-d'))
            ->where('expirationdate', '>=', date('Y-m-d'))
            ->where('active', 1)
            ->first();

        if (!$promo) {
            return 0;
        }

        // Apply promo logic
        return $promo->value ?? 0;
    }

    protected function calculateTotals(): array
    {
        $subtotal = 0;

        foreach ($this->items as $item) {
            $price = $item['pricingoverride'] ?? $this->getProductPrice($item['id'], $item['billingcycle']);
            $subtotal += $price;
        }

        foreach ($this->addons as $addon) {
            $price = $this->getAddonPrice($addon['addon_id']);
            $subtotal += $price * ($addon['qty'] ?? 1);
        }

        $discount = $this->applyPromoCode();

        return [
            'subtotal' => $subtotal,
            'discount' => $discount,
            'total' => $subtotal - $discount
        ];
    }

    protected function getProductPrice(int $productId, string $billingCycle): float
    {
        $price = Capsule::table('tblpricing')
            ->where('relid', $productId)
            ->where('type', 'product')
            ->where('billingcycle', $billingCycle)
            ->first();

        return $price->monthly ?? 0;
    }

    protected function getAddonPrice(int $addonId): float
    {
        $price = Capsule::table('tblpricing')
            ->where('relid', $addonId)
            ->where('type', 'addon')
            ->first();

        return $price->monthly ?? 0;
    }

    protected function generateOrderNumber(): string
    {
        return 'ORD-' . time() . '-' . random_int(1000, 9999);
    }

    protected function addOrderItem(int $orderId, array $item): void
    {
        Capsule::table('tblorderitems')->insert([
            'orderid' => $orderId,
            'type' => $item['type'],
            'relid' => $item['id'],
            'domain' => $item['domain'] ?? '',
            'billingcycle' => $item['billingcycle'] ?? 'monthly',
            'configoptions' => json_encode($item['configoptions'] ?? []),
            'customfields' => json_encode($item['customfields'] ?? []),
            'amount' => $item['pricingoverride'] ?? 0
        ]);
    }

    protected function addOrderAddon(int $orderId, array $addon): void
    {
        Capsule::table('tblorderaddons')->insert([
            'order_id' => $orderId,
            'addon_id' => $addon['addon_id'],
            'hosting_id' => $addon['hosting_id'],
            'configoptions' => json_encode($addon['configoptions'] ?? []),
            'qty' => $addon['qty'] ?? 1
        ]);
    }

    protected function processAutoSetup(int $orderId): void
    {
        // Handle automatic provisioning
    }
}
```

### Report Builder

```php
<?php
// includes/Builder/ReportBuilder.php

namespace CustomModule\Builder;

class ReportBuilder implements BuilderInterface
{
    protected string $reportType;
    protected ?\DateTimeInterface $startDate = null;
    protected ?\DateTimeInterface $endDate = null;
    protected array $filters = [];
    protected array $groupBy = [];
    protected array $orderBy = [];
    protected int $limit = 1000;
    protected int $offset = 0;
    protected array $columns = [];
    protected array $calculations = [];

    public function reset(): self
    {
        $this->reportType = '';
        $this->startDate = null;
        $this->endDate = null;
        $this->filters = [];
        $this->groupBy = [];
        $this->orderBy = [];
        $this->limit = 1000;
        $this->offset = 0;
        $this->columns = [];
        $this->calculations = [];

        return $this;
    }

    public function type(string $type): self
    {
        $this->reportType = $type;
        return $this;
    }

    public function forPeriod(\DateTimeInterface $start, \DateTimeInterface $end): self
    {
        $this->startDate = $start;
        $this->endDate = $end;
        return $this;
    }

    public function thisMonth(): self
    {
        $this->startDate = new \DateTime('first day of this month');
        $this->endDate = new \DateTime('last day of this month');
        return $this;
    }

    public function thisYear(): self
    {
        $this->startDate = new \DateTime('first day of January this year');
        $this->endDate = new \DateTime('last day of December this year');
        return $this;
    }

    public function filter(string $field, $operator, $value): self
    {
        $this->filters[] = ['field' => $field, 'operator' => $operator, 'value' => $value];
        return $this;
    }

    public function groupBy(array $fields): self
    {
        $this->groupBy = $fields;
        return $this;
    }

    public function orderBy(array $fields): self
    {
        $this->orderBy = $fields;
        return $this;
    }

    public function select(array $columns): self
    {
        $this->columns = $columns;
        return $this;
    }

    public function calculate(string $function, string $field, string $alias): self
    {
        $this->calculations[] = ['function' => $function, 'field' => $field, 'alias' => $alias];
        return $this;
    }

    public function limit(int $limit): self
    {
        $this->limit = $limit;
        return $this;
    }

    public function offset(int $offset): self
    {
        $this->offset = $offset;
        return $this;
    }

    public function build(): Report
    {
        return new Report([
            'type' => $this->reportType,
            'start_date' => $this->startDate,
            'end_date' => $this->endDate,
            'filters' => $this->filters,
            'group_by' => $this->groupBy,
            'order_by' => $this->orderBy,
            'columns' => $this->columns,
            'calculations' => $this->calculations,
            'limit' => $this->limit,
            'offset' => $this->offset
        ]);
    }
}

// Usage
$report = (new ReportBuilder())
    ->type('sales')
    ->thisYear()
    ->filter('status', '=', 'Active')
    ->filter('amount', '>', 100)
    ->groupBy(['month', 'product_category'])
    ->select(['product_name', 'SUM(amount) as total'])
    ->calculate('SUM', 'amount', 'total_revenue')
    ->calculate('COUNT', '*', 'order_count')
    ->orderBy(['total_revenue' => 'DESC'])
    ->limit(50)
    ->build();
```

### Director Pattern

```php
<?php
// includes/Builder/Director.php

namespace CustomModule\Builder;

class ReportDirector
{
    public function buildMonthlyRevenueReport(ReportBuilder $builder, int $year): Report
    {
        return $builder
            ->reset()
            ->type('revenue')
            ->forPeriod(
                new \DateTime("{$year}-01-01"),
                new \DateTime("{$year}-12-31")
            )
            ->groupBy(['month'])
            ->calculate('SUM', 'amount', 'total_revenue')
            ->select(['month', 'total_revenue', 'order_count'])
            ->build();
    }

    public function buildClientActivityReport(ReportBuilder $builder, int $clientId): Report
    {
        return $builder
            ->reset()
            ->type('activity')
            ->forPeriod(
                new \DateTime('-30 days'),
                new \DateTime('today')
            )
            ->filter('client_id', '=', $clientId)
            ->build();
    }

    public function buildTopProductsReport(ReportBuilder $builder, int $limit = 10): Report
    {
        return $builder
            ->reset()
            ->type('products')
            ->thisYear()
            ->groupBy(['product_id', 'product_name'])
            ->calculate('SUM', 'quantity', 'total_sold')
            ->calculate('SUM', 'amount', 'total_revenue')
            ->orderBy(['total_revenue' => 'DESC'])
            ->limit($limit)
            ->build();
    }
}
```

## Pros

- **Readability**: Clear, chainable API for complex object creation
- **Single Responsibility**: Construction logic separated from business logic
- **Immutability**: Builder can be reused for similar objects
- **Flexibility**: Create different representations with same builder
- **Validation**: Validate inputs before building

## Cons

- **Complexity**: More classes and code than simple construction
- **Maintenance**: Must update builder when object changes
- **Overkill**: Too much overhead for simple objects

## Best Practices

1. Use fluent interface for method chaining
2. Make builder reusable with reset() method
3. Include validation in build() method
4. Consider director for common construction patterns
5. Keep build methods idempotent where possible