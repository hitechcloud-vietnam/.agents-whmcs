# WHMCS Product Configurator Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building product configurator modules with options and addons.

## When to Use

- Creating product builder modules
- Building configurable product options
- Managing product bundles

## Product Configurator Patterns

```php
<?php
// modules/addons/{configurator}/{configurator}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {configurator}_config(): array {
    return [
        'name' => 'Product Configurator',
        'description' => 'Advanced product configuration system',
        'version' => '1.0',
        'author' => 'Author',
        'allow_user_modifications' => ['FriendlyName' => 'Allow User Modifications', 'Type' => 'yesno'],
    ];
}

function {configurator}_activate(): array {
    Capsule::schema()->create('mod_configurator_groups', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->text('description')->nullable();
        $t->integer('sort_order')->default(0);
        $t->boolean('required')->default(true);
        $t->string('display_type', 20)->default('select');
        $t->boolean('active')->default(true);
    });

    Capsule::schema()->create('mod_configurator_options', function($t) {
        $t->increments('id');
        $t->integer('group_id')->unsigned();
        $t->string('name', 100);
        $t->text('description')->nullable();
        $t->decimal('price', 10, 2)->default(0.00);
        $t->decimal('setup_fee', 10, 2)->default(0.00);
        $t->integer('quantity_min')->default(1);
        $t->integer('quantity_max')->default(1);
        $t->string('recurring_period', 20)->default('monthly');
        $t->boolean('default_selected')->default(false);
        $t->integer('sort_order')->default(0);
        $t->boolean('active')->default(true);
    });

    Capsule::schema()->create('mod_configurator_configs', function($t) {
        $t->increments('id');
        $t->integer('product_id')->unsigned();
        $t->integer('group_id')->unsigned();
        $t->json('selected_options');
        $t->decimal('base_price', 10, 2);
        $t->decimal('total_price', 10, 2);
        $t->timestamp('configured_at');
    });

    return ['status' => 'success', 'description' => 'Configurator activated'];
}

function {configurator}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_configurator_groups');
    Capsule::schema()->dropIfExists('mod_configurator_options');
    Capsule::schema()->dropIfExists('mod_configurator_configs');
    return ['status' => 'success', 'description' => 'Configurator deactivated'];
}
```

### Configuration Engine

```php
function getConfiguratorGroups(int $productId): array {
    return Capsule::table('mod_configurator_groups')
        ->where('active', 1)
        ->orderBy('sort_order')
        ->get();
}

function getGroupOptions(int $groupId): array {
    return Capsule::table('mod_configurator_options')
        ->where('group_id', $groupId)
        ->where('active', 1)
        ->orderBy('sort_order')
        ->get();
}

function calculateConfigurationPrice(int $productId, array $selectedOptions): array {
    $baseProduct = Capsule::table('tblproducts')->where('id', $productId)->first();
    $basePrice = $baseProduct->monthlycost ?? 0;

    $optionsTotal = 0;
    $setupTotal = 0;
    $optionsBreakdown = [];

    foreach ($selectedOptions as $groupId => $optionIds) {
        if (!is_array($optionIds)) {
            $optionIds = [$optionIds];
        }

        foreach ($optionIds as $optionId) {
            $option = Capsule::table('mod_configurator_options')
                ->where('id', $optionId)
                ->first();

            if ($option) {
                $optionsTotal += $option->price * ($option->quantity_max ?: 1);
                $setupTotal += $option->setup_fee;
                $optionsBreakdown[] = [
                    'id' => $option->id,
                    'name' => $option->name,
                    'price' => $option->price,
                    'setup_fee' => $option->setup_fee,
                ];
            }
        }
    }

    return [
        'base_price' => $basePrice,
        'options_total' => $optionsTotal,
        'setup_total' => $setupTotal,
        'total_price' => $basePrice + $optionsTotal,
        'total_setup' => $setupTotal,
        'breakdown' => $optionsBreakdown,
    ];
}

function saveConfiguration(int $productId, int $serviceId, array $selectedOptions): int {
    $pricing = calculateConfigurationPrice($productId, $selectedOptions);

    // Save or update config
    $existing = Capsule::table('mod_configurator_configs')
        ->where('service_id', $serviceId)
        ->first();

    $data = [
        'product_id' => $productId,
        'service_id' => $serviceId,
        'selected_options' => json_encode($selectedOptions),
        'base_price' => $pricing['base_price'],
        'total_price' => $pricing['total_price'],
        'configured_at' => date('Y-m-d H:i:s'),
    ];

    if ($existing) {
        Capsule::table('mod_configurator_configs')
            ->where('id', $existing->id)
            ->update($data);
        return $existing->id;
    }

    return Capsule::table('mod_configurator_configs')->insertGetId($data);
}

function getServiceConfiguration(int $serviceId): ?array {
    $config = Capsule::table('mod_configurator_configs')
        ->where('service_id', $serviceId)
        ->first();

    if (!$config) {
        return null;
    }

    $selectedOptions = json_decode($config->selected_options, true);
    $optionsDetails = [];

    foreach ($selectedOptions as $groupId => $optionIds) {
        if (!is_array($optionIds)) {
            $optionIds = [$optionIds];
        }

        foreach ($optionIds as $optionId) {
            $option = Capsule::table('mod_configurator_options')->where('id', $optionId)->first();
            if ($option) {
                $optionsDetails[] = $option;
            }
        }
    }

    return [
        'base_price' => $config->base_price,
        'total_price' => $config->total_price,
        'selected_options' => $optionsDetails,
    ];
}
```

### Configurator Hook for Cart

```php
add_hook('ProductDetailsPreOutput', 1, function($vars) {
    $productId = $vars['pid'];
    $groups = getConfiguratorGroups($productId);

    if (empty($groups)) {
        return;
    }

    $groupsWithOptions = [];
    foreach ($groups as $group) {
        $groupsWithOptions[$group->id] = [
            'group' => $group,
            'options' => getGroupOptions($group->id),
        ];
    }

    return [
        'configurator_enabled' => true,
        'configurator_groups' => $groupsWithOptions,
    ];
});

add_hook('CartItemConfigValidation', 1, function($vars) {
    $groups = getConfiguratorGroups($vars['pid']);

    foreach ($groups as $group) {
        if ($group->required) {
            $selected = $_POST['configurator'][$group->id] ?? [];
            if (empty($selected)) {
                return [
                    'error' => true,
                    'message' => "Please select an option for: {$group->name}",
                ];
            }
        }
    }
});
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-clientarea-builder
- whmcs-database-design
