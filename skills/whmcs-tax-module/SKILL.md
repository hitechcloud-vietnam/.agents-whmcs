# WHMCS Tax Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build custom tax calculation modules for regional tax compliance.

## Tax Module Structure

```php
<?php
/**
 * Tax Module (Addon)
 * Location: modules/addons/{module}/
 */

function {module}_config(): array {
    return [
        'name' => 'Advanced Tax',
        'description' => 'Multi-region tax calculation',
        'version' => '1.0',
    ];
}

function {module}_activate(): array {
    Capsule::schema()->create('mod_tax_rules', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('country', 2)->nullable();
        $t->string('state', 100)->nullable();
        $t->string('postal_pattern', 50)->nullable();
        $t->string('tax_type', 20);
        $t->decimal('rate', 6, 4);
        $t->boolean('is_compound')->default(false);
        $t->boolean('is_active')->default(true);
        $t->timestamps();

        $t->index(['country', 'state']);
    });

    Capsule::schema()->create('mod_tax_exemptions', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->string('exemption_type', 50);
        $t->string('certificate_number', 100)->nullable();
        $t->date('valid_from');
        $t->date('valid_until')->nullable();
        $t->string('state', 2)->nullable();
        $t->timestamps();

        $t->index('user_id');
    });

    return ['status' => 'success', 'description' => 'Tax module activated'];
}

function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_tax_rules');
    Capsule::schema()->dropIfExists('mod_tax_exemptions');
    return ['status' => 'success'];
}
```

## Tax Calculation

```php
<?php
class TaxCalculator {
    public function calculateTax(array $item, array $location): array {
        $rules = $this->getMatchingRules($location);
        $subtotal = $item['amount'] ?? 0;
        $taxes = [];
        $totalTax = 0;

        foreach ($rules as $rule) {
            if ($rule->is_compound) {
                $taxable = $subtotal + $totalTax;
            } else {
                $taxable = $subtotal;
            }

            $taxAmount = $taxable * ($rule->rate / 100);
            $totalTax += $taxAmount;

            $taxes[] = [
                'name' => $rule->name,
                'rate' => $rule->rate,
                'amount' => $taxAmount,
                'type' => $rule->tax_type,
            ];
        }

        return [
            'subtotal' => $subtotal,
            'taxes' => $taxes,
            'total_tax' => $totalTax,
            'total' => $subtotal + $totalTax,
        ];
    }

    private function getMatchingRules(array $location): array {
        return Capsule::table('mod_tax_rules')
            ->where('is_active', 1)
            ->where(function($q) use ($location) {
                $q->where('country', $location['country'] ?? '')
                  ->orWhereNull('country');

                if (!empty($location['state'])) {
                    $q->where(function($q2) use ($location) {
                        $q2->where('state', $location['state'])
                           ->orWhereNull('state');
                    });
                }
            })
            ->orderBy('tax_type')
            ->get()
            ->toArray();
    }

    public function isExempt(int $userId, array $location): bool {
        $exemption = Capsule::table('mod_tax_exemptions')
            ->where('user_id', $userId)
            ->where('valid_from', '<=', date('Y-m-d'))
            ->where(function($q) {
                $q->whereNull('valid_until')
                  ->orWhere('valid_until', '>=', date('Y-m-d'));
            })
            ->where(function($q) use ($location) {
                $q->whereNull('state')
                  ->orWhere('state', $location['state'] ?? '');
            })
            ->first();

        return $exemption !== null;
    }
}
```

## Tax Reporting

```php
<?php
class TaxReporting {
    public function generateReport(string $startDate, string $endDate, ?string $jurisdiction = null): array {
        $query = Capsule::table('mod_tax_collected')
            ->selectRaw('tax_type, SUM(amount) as total')
            ->whereBetween('collected_at', [$startDate, $endDate])
            ->groupBy('tax_type');

        if ($jurisdiction) {
            $query->where('jurisdiction', $jurisdiction);
        }

        $collected = $query->get();

        $refunds = Capsule::table('mod_tax_refunds')
            ->selectRaw('tax_type, SUM(amount) as total')
            ->whereBetween('refunded_at', [$startDate, $endDate])
            ->groupBy('tax_type')
            ->get();

        $report = [];
        foreach ($collected as $item) {
            $refund = $refunds->firstWhere('tax_type', $item->tax_type);
            $report[$item->tax_type] = [
                'collected' => $item->total,
                'refunded' => $refund ? $refund->total : 0,
                'net' => $item->total - ($refund ? $refund->total : 0),
            ];
        }

        return $report;
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-invoice-customization