# WHMCS Cost Calculator Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building cost estimation and profit margin calculators.

## When to Use

- Creating pricing calculators
- Building profit margin modules
- Managing distributor pricing

## Cost Calculator Patterns

```php
<?php
// modules/addons/{costcalculator}/{costcalculator}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {costcalculator}_config(): array {
    return [
        'name' => 'Cost Calculator',
        'description' => 'Cost estimation and profit margin tool',
        'version' => '1.0',
        'author' => 'Author',
        'default_markup' => ['FriendlyName' => 'Default Markup (%)', 'Type' => 'text', 'Default' => '20'],
        'tax_rate' => ['FriendlyName' => 'Tax Rate (%)', 'Type' => 'text', 'Default' => '10'],
    ];
}

function {costcalculator}_activate(): array {
    Capsule::schema()->create('mod_cost_products', function($t) {
        $t->increments('id');
        $t->string('product_type', 50);
        $t->string('name', 255);
        $t->decimal('cost_price', 10, 2);
        $t->decimal('reseller_price', 10, 2)->nullable();
        $t->decimal('retail_price', 10, 2)->nullable();
        $t->decimal('markup_percent', 5, 2)->default(20.00);
        $t->boolean('active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_cost_calculation_logs', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->string('product_type', 50);
        $t->decimal('cost_price', 10, 2);
        $t->decimal('selling_price', 10, 2);
        $t->decimal('profit_margin', 10, 2);
        $t->decimal('margin_percent', 5, 2);
        $t->timestamp('calculated_at');
    });

    Capsule::schema()->create('mod_cost_price_lists', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('type', 50);
        $t->json('prices');
        $t->boolean('is_default')->default(false);
        $t->timestamps();
    });

    return ['status' => 'success', 'description' => 'Cost calculator activated'];
}

function {costcalculator}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_cost_products');
    Capsule::schema()->dropIfExists('mod_cost_calculation_logs');
    Capsule::schema()->dropIfExists('mod_cost_price_lists');
    return ['status' => 'success', 'description' => 'Cost calculator deactivated'];
}

function {costcalculator}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'calculator';

    if ($action === 'calculator') {
        include __DIR__ . '/templates/admin/calculator.tpl';
    } elseif ($action === 'products') {
        include __DIR__ . '/templates/admin/products.tpl';
    } elseif ($action === 'reports') {
        include __DIR__ . '/templates/admin/reports.tpl';
    }
}

// Core calculation functions
function calculatePrice(float $costPrice, float $markupPercent, string $method = 'markup'): array {
    switch ($method) {
        case 'markup':
            $profit = $costPrice * ($markupPercent / 100);
            $sellingPrice = $costPrice + $profit;
            break;

        case 'margin':
            $sellingPrice = $costPrice / (1 - ($markupPercent / 100));
            $profit = $sellingPrice - $costPrice;
            break;

        case 'reseller':
            $resellerPrice = $costPrice * 1.1;
            $sellingPrice = $resellerPrice * (1 + ($markupPercent / 100));
            $profit = $sellingPrice - $resellerPrice;
            break;
    }

    $marginPercent = ($sellingPrice > 0) ? ($profit / $sellingPrice) * 100 : 0;

    return [
        'cost_price' => $costPrice,
        'selling_price' => round($sellingPrice, 2),
        'profit' => round($profit, 2),
        'margin_percent' => round($marginPercent, 2),
        'markup_percent' => $markupPercent,
    ];
}

function calculateBulkPricing(array $items, float $globalMarkup): array {
    $results = [];
    $totalCost = 0;
    $totalSelling = 0;

    foreach ($items as $item) {
        $product = Capsule::table('mod_cost_products')
            ->where('product_type', $item['type'])
            ->where('name', $item['name'])
            ->where('active', 1)
            ->first();

        if ($product) {
            $calc = calculatePrice($product->cost_price, $item['markup'] ?? $globalMarkup);
            $calc['quantity'] = $item['quantity'] ?? 1;
            $calc['line_total'] = $calc['selling_price'] * $calc['quantity'];
            $calc['line_cost'] = $product->cost_price * $calc['quantity'];

            $results[] = $calc;
            $totalCost += $calc['line_cost'];
            $totalSelling += $calc['line_total'];
        }
    }

    return [
        'items' => $results,
        'total_cost' => round($totalCost, 2),
        'total_selling' => round($totalSelling, 2),
        'total_profit' => round($totalSelling - $totalCost, 2),
        'average_margin' => $totalSelling > 0 ? round((($totalSelling - $totalCost) / $totalSelling) * 100, 2) : 0,
    ];
}

// Product management
function addProduct(string $type, string $name, float $costPrice, float $markup = null): int {
    $defaultMarkup = (float) Capsule::table('mod_configuration')
        ->where('setting', 'default_markup')
        ->value('value') ?? 20;

    return Capsule::table('mod_cost_products')->insertGetId([
        'product_type' => $type,
        'name' => $name,
        'cost_price' => $costPrice,
        'markup_percent' => $markup ?? $defaultMarkup,
        'active' => 1,
        'created_at' => date('Y-m-d H:i:s'),
        'updated_at' => date('Y-m-d H:i:s'),
    ]);
}

function updateProductPrices(int $productId, float $markup): void {
    $product = Capsule::table('mod_cost_products')->where('id', $productId)->first();

    $calc = calculatePrice($product->cost_price, $markup);

    Capsule::table('mod_cost_products')
        ->where('id', $productId)
        ->update([
            'markup_percent' => $markup,
            'reseller_price' => $calc['selling_price'] * 1.1,
            'retail_price' => $calc['selling_price'],
            'updated_at' => date('Y-m-d H:i:s'),
        ]);
}

function bulkUpdateMarkups(string $productType, float $newMarkup): int {
    return Capsule::table('mod_cost_products')
        ->where('product_type', $productType)
        ->update([
            'markup_percent' => $newMarkup,
            'updated_at' => date('Y-m-d H:i:s'),
        ]);
}

// Log calculations for reporting
function logCalculation(int $userId, array $result): void {
    foreach ($result['items'] ?? [] as $item) {
        Capsule::table('mod_cost_calculation_logs')->insert([
            'user_id' => $userId,
            'product_type' => $item['type'] ?? 'unknown',
            'cost_price' => $item['cost_price'] ?? 0,
            'selling_price' => $item['selling_price'] ?? 0,
            'profit_margin' => $item['profit'] ?? 0,
            'margin_percent' => $item['margin_percent'] ?? 0,
            'calculated_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

// Reports
function getProfitReport(array $filters = []): array {
    $query = Capsule::table('mod_cost_calculation_logs')
        ->selectRaw('product_type, COUNT(*) as calculations, SUM(cost_price) as total_cost, SUM(selling_price) as total_selling, AVG(margin_percent) as avg_margin')
        ->groupBy('product_type');

    if (!empty($filters['from_date'])) {
        $query->where('calculated_at', '>=', $filters['from_date']);
    }

    if (!empty($filters['to_date'])) {
        $query->where('calculated_at', '<=', $filters['to_date']);
    }

    if (!empty($filters['product_type'])) {
        $query->where('product_type', $filters['product_type']);
    }

    return $query->get();
}
```

### Calculator Template

```smarty
<div class="cost-calculator">
    <h2>Cost Calculator</h2>

    <form method="post" action="?m={module}&action=calculate">
        <input type="hidden" name="token" value="{$token}">

        <div class="form-group">
            <label>Calculation Method</label>
            <select name="method" id="calc-method">
                <option value="markup">Markup</option>
                <option value="margin">Margin</option>
                <option value="reseller">Reseller</option>
            </select>
        </div>

        <div class="form-group">
            <label>Cost Price</label>
            <input type="number" name="cost_price" step="0.01" min="0" required>
        </div>

        <div class="form-group">
            <label>Markup/Margin (%)</label>
            <input type="number" name="markup" step="0.01" min="0" value="20">
        </div>

        <button type="submit" class="btn btn-primary">Calculate</button>
    </form>

    {if $result}
    <div class="calculation-results">
        <h3>Results</h3>
        <table class="results-table">
            <tr>
                <td>Cost Price:</td>
                <td>{$result.cost_price|string_format:"%.2f"}</td>
            </tr>
            <tr>
                <td>Selling Price:</td>
                <td class="highlight">{$result.selling_price|string_format:"%.2f"}</td>
            </tr>
            <tr>
                <td>Profit:</td>
                <td>{$result.profit|string_format:"%.2f"}</td>
            </tr>
            <tr>
                <td>Margin %:</td>
                <td>{$result.margin_percent|string_format:"%.2f"}%</td>
            </tr>
        </table>
    </div>
    {/if}
</div>
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-reporting
- whmcs-admin-ui-builder
