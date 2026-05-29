# WHMCS Product Builder - Complete Module

## Module Definition

```php
<?php
/**
 * WHMCS Product Builder Module
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function whmcs_product_builder_config()
{
    return [
        'name' => 'Product Builder',
        'description' => 'Visual product configuration builder with dependencies and pricing',
        'author' => 'DevKit Generator',
        'version' => '1.0.0',
        'fields' => [
            'enable_drag_drop' => [
                'Type' => 'yesno',
                'Default' => 'on',
                'Description' => 'Enable drag-and-drop interface'
            ],
            'max_options' => [
                'Type' => 'text',
                'Default' => '50',
                'Description' => 'Maximum configurable options per product'
            ],
            'enable_dependencies' => [
                'Type' => 'yesno',
                'Default' => 'on',
                'Description' => 'Enable product option dependencies'
            ],
            'allow_quantity' => [
                'Type' => 'yesno',
                'Default' => 'on',
                'Description' => 'Allow quantity selection in products'
            ]
        ]
    ];
}

function whmcs_product_builder_activate()
{
    $sql = "CREATE TABLE IF NOT EXISTS `mod_product_builder_configs` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `product_id` INT(11) NOT NULL,
        `config_data` TEXT NOT NULL,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`),
        KEY `product_id` (`product_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    full_query($sql);

    $sql2 = "CREATE TABLE IF NOT EXISTS `mod_product_builder_options` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `config_id` INT(11) NOT NULL,
        `option_name` VARCHAR(255) NOT NULL,
        `option_type` ENUM('select','checkbox','radio','text','number') DEFAULT 'select',
        `option_values` TEXT,
        `price_modifier` DECIMAL(10,2) DEFAULT 0.00,
        `sort_order` INT(11) DEFAULT 0,
        `is_required` TINYINT(1) DEFAULT 0,
        `depends_on` VARCHAR(255) DEFAULT NULL,
        PRIMARY KEY (`id`),
        KEY `config_id` (`config_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    full_query($sql2);

    $sql3 = "CREATE TABLE IF NOT EXISTS `mod_product_builder_templates` (
        `id` INT(11) NOT NULL AUTO_INCREMENT,
        `template_name` VARCHAR(255) NOT NULL,
        `template_data` TEXT NOT NULL,
        `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    full_query($sql3);

    return ['status' => 'success', 'description' => 'Product Builder activated'];
}

function whmcs_product_builder_deactivate()
{
    return ['status' => 'success', 'description' => 'Product Builder deactivated'];
}

function whmcs_product_builder_output($vars)
{
    $modulelink = $vars['modulelink'];
    
    echo '<div class="whmcs-module-admin">';
    echo '<h2>Product Builder Configuration</h2>';
    
    // Product configuration form
    echo '<div class="panel panel-default">';
    echo '<div class="panel-heading">Create New Product Configuration</div>';
    echo '<div class="panel-body">';
    echo '<form method="post" action="">';
    echo '<div class="form-group">';
    echo '<label>Select Product</label>';
    echo '<select name="product_id" class="form-control">';
    
    $result = select_query("tbld products", "id, name", "", "name", "ASC");
    while ($data = mysql_fetch_array($result)) {
        echo '<option value="' . $data['id'] . '">' . $data['name'] . '</option>';
    }
    
    echo '</select>';
    echo '</div>';
    echo '<div class="form-group">';
    echo '<label>Configuration Name</label>';
    echo '<input type="text" name="config_name" class="form-control" required />';
    echo '</div>';
    echo '<div class="form-group">';
    echo '<label>Options (JSON)</label>';
    echo '<textarea name="options_json" class="form-control" rows="10" placeholder=\'{"options": [{"name": "RAM", "type": "select", "values": ["2GB", "4GB", "8GB"], "price_modifier": 5]}]\'></textarea>';
    echo '</div>';
    echo '<button type="submit" name="save_config" class="btn btn-primary">Save Configuration</button>';
    echo '</form>';
    echo '</div>';
    echo '</div>';
    
    // List existing configurations
    echo '<div class="panel panel-default" style="margin-top: 20px;">';
    echo '<div class="panel-heading">Existing Configurations</div>';
    echo '<div class="panel-body">';
    
    $configs = select_query("mod_product_builder_configs", "*", "", "id", "DESC");
    if (mysql_num_rows($configs) > 0) {
        echo '<table class="datatable">';
        echo '<thead><tr><th>Product</th><th>Created</th><th>Actions</th></tr></thead>';
        echo '<tbody>';
        while ($config = mysql_fetch_array($configs)) {
            $product = localAPI("GetProducts", ["pid" => $config['product_id']]);
            echo '<tr>';
            echo '<td>' . ($product['products']['product'][0]['name'] ?? 'Unknown') . '</td>';
            echo '<td>' . $config['created_at'] . '</td>';
            echo '<td><a href="?module=whmcs_product_builder&action=edit&id=' . $config['id'] . '" class="btn btn-xs btn-default">Edit</a></td>';
            echo '</tr>';
        }
        echo '</tbody></table>';
    } else {
        echo '<p>No configurations found.</p>';
    }
    
    echo '</div>';
    echo '</div>';
    echo '</div>';
}

add_hook('ClientAreaPageProductConfig', 1, function($vars) {
    $pid = $vars['pid'];
    $config = select_query("mod_product_builder_configs", "*", ["product_id" => $pid]);
    
    if (mysql_num_rows($config) > 0) {
        $configData = mysql_fetch_array($config);
        return [
            'builder_enabled' => true,
            'config_data' => json_decode($configData['config_data'], true)
        ];
    }
});
```

## Hooks Implementation

```php
<?php
// Product Builder Hooks

add_hook('ProductDetailsPreOutput', 1, function($vars) {
    $pid = $vars['pid'];
    $config = select_query("mod_product_builder_configs", "config_data", ["product_id" => $pid]);
    
    if ($row = mysql_fetch_array($config)) {
        return ['custom_config' => json_decode($row['config_data'], true)];
    }
});

add_hook('AddToCart', 1, function($vars) {
    if (!empty($_POST['builder_options'])) {
        logActivity('Product built with custom options: ' . json_encode($_POST['builder_options']));
    }
});

add_hook('OrderFormProductBundles', 1, function($vars) {
    return ['show_bundle_builder' => true];
});
```

## Template File

```smarty
<div class="product-builder" id="productBuilder">
    <div class="builder-header">
        <h3>Configure Your Product</h3>
    </div>
    
    <div class="builder-options" id="builderOptions">
        {foreach from=$config_data.options item=option}
            <div class="builder-option" data-option-id="{$option.id}" 
                 {if $option.depends_on}data-depends-on="{$option.depends_on}"{/if}>
                <label class="option-label">{$option.name}
                    {if $option.is_required}<span class="required">*</span>{/if}
                </label>
                
                {if $option.type == 'select'}
                    <select name="builder_options[{$option.id}]" 
                            class="form-control" {if $option.is_required}required{/if}>
                        <option value="">Select...</option>
                        {foreach from=$option.values item=value}
                            <option value="{$value.id}" data-price="{$value.price_modifier}">
                                {$value.name} {if $value.price_modifier > 0}+${$value.price_modifier}{/if}
                            </option>
                        {/foreach}
                    </select>
                {elseif $option.type == 'radio'}
                    {foreach from=$option.values item=value}
                        <label class="radio-label">
                            <input type="radio" name="builder_options[{$option.id}]" 
                                   value="{$value.id}" data-price="{$value.price_modifier}" 
                                   {if $option.is_required}required{/if} />
                            {$value.name} {if $value.price_modifier > 0}+${$value.price_modifier}{/if}
                        </label>
                    {/foreach}
                {elseif $option.type == 'checkbox'}
                    {foreach from=$option.values item=value}
                        <label class="checkbox-label">
                            <input type="checkbox" name="builder_options[{$option.id}][]" 
                                   value="{$value.id}" data-price="{$value.price_modifier}" />
                            {$value.name} {if $value.price_modifier > 0}+${$value.price_modifier}{/if}
                        </label>
                    {/foreach}
                {elseif $option.type == 'number'}
                    <input type="number" name="builder_options[{$option.id}]" 
                           class="form-control" min="0" data-price="{$option.price_modifier}"
                           {if $option.is_required}required{/if} />
                {else}
                    <input type="text" name="builder_options[{$option.id}]" 
                           class="form-control" {if $option.is_required}required{/if} />
                {/if}
            </div>
        {/foreach}
    </div>
    
    <div class="builder-summary">
        <h4>Configuration Summary</h4>
        <div id="summaryContent">
            <p class="placeholder">Select options to see summary</p>
        </div>
        <div class="total-price">
            <span>Total: </span>
            <span id="totalPrice">{$product.base_price}</span>
        </div>
    </div>
</div>

<script>
(function() {
    var basePrice = {$product.base_price};
    var totalPrice = basePrice;
    
    document.querySelectorAll('[data-price]').forEach(function(el) {
        el.addEventListener('change', function() {
            calculateTotal();
        });
    });
    
    function calculateTotal() {
        totalPrice = basePrice;
        document.querySelectorAll('[data-price]').forEach(function(el) {
            if (el.type === 'checkbox') {
                if (el.checked) {
                    totalPrice += parseFloat(el.dataset.price);
                }
            } else if (el.type === 'radio' || el.type === 'select') {
                if (el.selectedOptions[0]) {
                    totalPrice += parseFloat(el.selectedOptions[0].dataset.price || 0);
                }
            } else if (el.type === 'number') {
                totalPrice += parseFloat(el.dataset.price || 0) * parseInt(el.value || 0);
            }
        });
        document.getElementById('totalPrice').textContent = '$' + totalPrice.toFixed(2);
    }
})();
</script>
```

## Database Schema

```sql
CREATE TABLE `mod_product_builder_configs` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `product_id` INT(11) NOT NULL,
    `config_data` TEXT NOT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `product_id` (`product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `mod_product_builder_options` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `config_id` INT(11) NOT NULL,
    `option_name` VARCHAR(255) NOT NULL,
    `option_type` ENUM('select','checkbox','radio','text','number') DEFAULT 'select',
    `option_values` TEXT,
    `price_modifier` DECIMAL(10,2) DEFAULT 0.00,
    `sort_order` INT(11) DEFAULT 0,
    `is_required` TINYINT(1) DEFAULT 0,
    `depends_on` VARCHAR(255) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `config_id` (`config_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `mod_product_builder_templates` (
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    `template_name` VARCHAR(255) NOT NULL,
    `template_data` TEXT NOT NULL,
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```