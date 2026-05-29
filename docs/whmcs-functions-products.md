# WHMCS Product Functions

Complete reference for product management functions in WHMCS.

## Overview

WHMCS provides comprehensive product management including creation, pricing, configuration, and inventory management.

## Product CRUD Operations

### createProduct()

Creates a new product.

```php
/**
 * Create a new product
 * 
 * @param array $data Product data
 * @return int Product ID
 */
function createProduct(array $data): int
{
    return Capsule::table('tblproducts')->insertGetId([
        'type' => $data['type'] ?? 'hosting',
        'name' => $data['name'],
        'slug' => $data['slug'] ?? '',
        'description' => $data['description'] ?? '',
        'descriptionHtml' => $data['description_html'] ?? '',
        'module' => $data['module'] ?? '',
        'servergroup' => $data['servergroup'] ?? 0,
        'hidden' => $data['hidden'] ?? 0,
        'showdomainoptions' => $data['show_domain_options'] ?? 0,
        'stockcontrol' => $data['stock_control'] ?? 0,
        'stocklevel' => $data['stock_level'] ?? 0,
        'is_featured' => $data['is_featured'] ?? 0,
        'tshirt' => $data['tshirt'] ?? 0,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}
```

**Example:**
```php
$productId = createProduct([
    'type' => 'hosting',
    'name' => 'Premium Hosting',
    'slug' => 'premium-hosting',
    'description' => 'Premium web hosting with all features',
    'module' => 'cpanel',
    'servergroup' => 1,
    'is_featured' => 1
]);
```

### getProduct()

Retrieves a product by ID.

```php
/**
 * Get product by ID
 * 
 * @param int $productId Product ID
 * @return array|null Product data
 */
function getProduct(int $productId): ?array
{
    $result = Capsule::table('tblproducts')
        ->where('id', $productId)
        ->first();
    
    return $result ? (array) $result : null;
}
```

**Example:**
```php
$product = getProduct(1);

echo "Product: {$product['name']}";
echo "Price: $" . number_format($product['pricing']['monthly'], 2);
```

### getProducts()

Retrieves all products with optional filtering.

```php
/**
 * Get all products
 * 
 * @param string $type Product type filter
 * @param bool $includeHidden Include hidden products
 * @return array Products
 */
function getProducts(string $type = '', bool $includeHidden = false): array
{
    $query = Capsule::table('tblproducts')
        ->select('tblproducts.*', 'tblproductgroups.name as group_name')
        ->leftJoin('tblproductgroups', 'tblproductgroups.id', '=', 'tblproducts.gid')
        ->orderBy('tblproductgroups.sort', 'asc')
        ->orderBy('tblproducts.name', 'asc');
    
    if ($type) {
        $query->where('tblproducts.type', $type);
    }
    
    if (!$includeHidden) {
        $query->where('tblproducts.hidden', 0);
    }
    
    return $query->get()->toArray();
}
```

**Example:**
```php
// Get all hosting products
$hostingProducts = getProducts('hosting');

// Get all products including hidden
$allProducts = getProducts('', true);
```

### updateProduct()

Updates an existing product.

```php
/**
 * Update a product
 * 
 * @param int $productId Product ID
 * @param array $data Updated data
 * @return bool Success status
 */
function updateProduct(int $productId, array $data): bool
{
    $data['updated_at'] = date('Y-m-d H:i:s');
    
    return Capsule::table('tblproducts')
        ->where('id', $productId)
        ->update($data) > 0;
}
```

**Example:**
```php
updateProduct(1, [
    'name' => 'Premium Hosting Plus',
    'description' => 'Enhanced premium hosting with more resources',
    'stocklevel' => 100
]);
```

### deleteProduct()

Deletes a product.

```php
/**
 * Delete a product
 * 
 * @param int $productId Product ID
 * @param bool $force Force delete even with active services
 * @return bool Success status
 */
function deleteProduct(int $productId, bool $force = false): bool
{
    // Check for active services
    if (!$force) {
        $activeServices = Capsule::table('tblhosting')
            ->where('packageid', $productId)
            ->where('domainstatus', 'Active')
            ->count();
        
        if ($activeServices > 0) {
            return false;
        }
    }
    
    // Delete associated pricing
    Capsule::table('tblpricing')
        ->where('relid', $productId)
        ->where('type', 'product')
        ->delete();
    
    // Delete product
    return Capsule::table('tblproducts')
        ->where('id', $productId)
        ->delete() > 0;
}
```

## Product Pricing

### setProductPricing()

Sets pricing for a product.

```php
/**
 * Set product pricing
 * 
 * @param int $productId Product ID
 * @param array $pricing Pricing structure
 * @param int $currencyId Currency ID
 * @return bool Success status
 */
function setProductPricing(int $productId, array $pricing, int $currencyId = 1): bool
{
    $billingCycles = [
        'monthly' => 'monthly',
        'quarterly' => 'quarterly',
        'semiannually' => 'semiannually',
        'annually' => 'annually',
        'biennially' => 'biennially',
        'triennially' => 'triennially',
    ];
    
    foreach ($billingCycles as $cycle => $column) {
        $amountKey = $cycle;
        
        // Set pricing for this currency
        Capsule::table('tblpricing')->updateOrInsert(
            [
                'relid' => $productId,
                'type' => 'product',
                'currency' => $currencyId,
                'tsetup' => 0,
            ],
            [
                $column => $pricing[$amountKey] ?? 0,
                'msetup' => $pricing['monthly_setup'] ?? 0,
                'qsetup' => $pricing['quarterly_setup'] ?? 0,
                'ssetup' => $pricing['semiannually_setup'] ?? 0,
                'asetup' => $pricing['annually_setup'] ?? 0,
                'bsetup' => $pricing['biennially_setup'] ?? 0,
                'tsetup' => $pricing['triennially_setup'] ?? 0,
            ]
        );
    }
    
    return true;
}
```

**Example:**
```php
setProductPricing(1, [
    'monthly' => 9.99,
    'quarterly' => 26.97,
    'semiannually' => 49.95,
    'annually' => 89.99,
    'biennially' => 159.99,
    'monthly_setup' => 0,
    'annually_setup' => 0
], 1);
```

### getProductPricing()

Gets pricing for a product.

```php
/**
 * Get product pricing
 * 
 * @param int $productId Product ID
 * @param int $currencyId Currency ID
 * @return array Pricing
 */
function getProductPricing(int $productId, int $currencyId = 1): array
{
    $pricing = Capsule::table('tblpricing')
        ->where('relid', $productId)
        ->where('type', 'product')
        ->where('currency', $currencyId)
        ->first();
    
    return $pricing ? (array) $pricing : [];
}
```

## Product Groups

### createProductGroup()

Creates a product group.

```php
/**
 * Create product group
 * 
 * @param array $data Group data
 * @return int Group ID
 */
function createProductGroup(array $data): int
{
    return Capsule::table('tblproductgroups')->insertGetId([
        'name' => $data['name'],
        'headline' => $data['headline'] ?? '',
        'tagline' => $data['tagline'] ?? '',
        'description' => $data['description'] ?? '',
        'hidden' => $data['hidden'] ?? 0,
        'order' => $data['order'] ?? 0,
    ]);
}
```

**Example:**
```php
$groupId = createProductGroup([
    'name' => 'Web Hosting',
    'headline' => 'Reliable Web Hosting Solutions',
    'tagline' => 'Fast and secure hosting',
    'order' => 1
]);
```

### getProductGroups()

Retrieves all product groups.

```php
/**
 * Get product groups
 * 
 * @param bool $includeHidden Include hidden groups
 * @return array Groups
 */
function getProductGroups(bool $includeHidden = false): array
{
    $query = Capsule::table('tblproductgroups')
        ->orderBy('order', 'asc');
    
    if (!$includeHidden) {
        $query->where('hidden', 0);
    }
    
    return $query->get()->toArray();
}
```

## Product Config Options

### getProductConfigOptions()

Gets configuration options for a product.

```php
/**
 * Get product config options
 * 
 * @param int $productId Product ID
 * @return array Config options
 */
function getProductConfigOptions(int $productId): array
{
    return Capsule::table('tblproductconfigoptions')
        ->select('tblproductconfigoptions.*', 'tblproductconfiggroups.name as group_name')
        ->leftJoin('tblproductconfiggroups', 'tblproductconfiggroups.id', '=', 'tblproductconfigoptions.gid')
        ->where('tblproductconfigoptions.productid', $productId)
        ->orderBy('tblproductconfiggroups.sort', 'asc')
        ->orderBy('tblproductconfigoptions.sort', 'asc')
        ->get()
        ->toArray();
}
```

### addProductConfigOption()

Adds a configuration option to a product.

```php
/**
 * Add config option to product
 * 
 * @param int $productId Product ID
 * @param int $configOptionId Config option ID
 * @return bool Success status
 */
function addProductConfigOption(int $productId, int $configOptionId): bool
{
    return Capsule::table('tblproductconfiglinks')->insert([
        'gid' => 0,
        'productid' => $productId,
        'configid' => $configOptionId,
    ]) > 0;
}
```

## Product Custom Fields

### getProductCustomFields()

Gets custom fields for a product.

```php
/**
 * Get product custom fields
 * 
 * @param int $productId Product ID
 * @return array Custom fields
 */
function getProductCustomFields(int $productId): array
{
    return Capsule::table('tblcustomfields')
        ->where('relid', $productId)
        ->where('type', 'product')
        ->orderBy('sort', 'asc')
        ->get()
        ->toArray();
}
```

### createProductCustomField()

Creates a custom field for a product.

```php
/**
 * Create product custom field
 * 
 * @param int $productId Product ID
 * @param array $data Field data
 * @return int Field ID
 */
function createProductCustomField(int $productId, array $data): int
{
    return Capsule::table('tblcustomfields')->insertGetId([
        'relid' => $productId,
        'type' => 'product',
        'fieldname' => $data['fieldname'],
        'fieldtype' => $data['fieldtype'] ?? 'text',
        'description' => $data['description'] ?? '',
        'fieldoptions' => $data['fieldoptions'] ?? '',
        'required' => $data['required'] ?? 0,
        'showorder' => $data['showorder'] ?? 1,
        'adminonly' => $data['adminonly'] ?? 0,
        'sortorder' => $data['sortorder'] ?? 0,
    ]);
}
```

**Example:**
```php
createProductCustomField(1, [
    'fieldname' => 'control_panel_url',
    'fieldtype' => 'text',
    'description' => 'Client cPanel URL',
    'adminonly' => 1
]);

createProductCustomField(1, [
    'fieldname' => 'hosting_type',
    'fieldtype' => 'dropdown',
    'description' => 'Hosting Type',
    'fieldoptions' => "Standard\nPremium\nEnterprise",
    'required' => 1
]);
```

## Product Modules

### getProductModule()

Gets the provisioning module for a product.

```php
/**
 * Get product provisioning module
 * 
 * @param int $productId Product ID
 * @return array|null Module info
 */
function getProductModule(int $productId): ?array
{
    $product = getProduct($productId);
    
    if (!$product || empty($product['module'])) {
        return null;
    }
    
    return Capsule::table('tblmodules')
        ->where('type', 'server')
        ->where('name', $product['module'])
        ->first();
}
```

### setProductModule()

Sets the provisioning module for a product.

```php
/**
 * Set product provisioning module
 * 
 * @param int $productId Product ID
 * @param string $moduleName Module name
 * @param array $moduleConfig Module configuration
 * @return bool Success status
 */
function setProductModule(int $productId, string $moduleName, array $moduleConfig = []): bool
{
    return updateProduct($productId, [
        'module' => $moduleName,
        'servergroup' => $moduleConfig['servergroup'] ?? 0,
    ]);
}
```

## Product Search

```php
/**
 * Search products
 * 
 * @param string $term Search term
 * @param string $type Product type
 * @return array Matching products
 */
function searchProducts(string $term, string $type = ''): array
{
    $query = Capsule::table('tblproducts')
        ->select('tblproducts.*', 'tblproductgroups.name as group_name')
        ->leftJoin('tblproductgroups', 'tblproductgroups.id', '=', 'tblproducts.gid')
        ->where(function($q) use ($term) {
            $q->where('tblproducts.name', 'like', '%' . $term . '%')
              ->orWhere('tblproducts.description', 'like', '%' . $term . '%');
        });
    
    if ($type) {
        $query->where('tblproducts.type', $type);
    }
    
    return $query->get()->toArray();
}
```

## Product Availability

```php
/**
 * Check product availability
 * 
 * @param int $productId Product ID
 * @return array Availability info
 */
function checkProductAvailability(int $productId): array
{
    $product = getProduct($productId);
    
    if (!$product) {
        return ['available' => false, 'reason' => 'Product not found'];
    }
    
    // Check stock
    if ($product['stockcontrol']) {
        if ($product['stocklevel'] <= 0) {
            return ['available' => false, 'reason' => 'Out of stock'];
        }
    }
    
    // Check hidden status
    if ($product['hidden']) {
        return ['available' => false, 'reason' => 'Product hidden'];
    }
    
    return [
        'available' => true,
        'in_stock' => $product['stocklevel'],
        'stock_control' => (bool) $product['stockcontrol']
    ];
}
```

## Product Statistics

```php
/**
 * Get product statistics
 * 
 * @param int $productId Product ID
 * @return array Statistics
 */
function getProductStatistics(int $productId): array
{
    // Active services count
    $activeServices = Capsule::table('tblhosting')
        ->where('packageid', $productId)
        ->where('domainstatus', 'Active')
        ->count();
    
    // Total revenue
    $totalRevenue = Capsule::table('tblhosting')
        ->where('packageid', $productId)
        ->whereIn('domainstatus', ['Active', 'Suspended'])
        ->sum('amount');
    
    // Orders this month
    $monthStart = date('Y-m-01');
    $monthOrders = Capsule::table('tblorderitems')
        ->where('productid', $productId)
        ->where('type', 'Hosting')
        ->whereHas('order', function($q) use ($monthStart) {
            $q->where('date', '>=', $monthStart);
        })
        ->count();
    
    return [
        'product_id' => $productId,
        'active_services' => $activeServices,
        'total_revenue' => $totalRevenue,
        'orders_this_month' => $monthOrders,
    ];
}
```

## Related Functions

- [whmcs-functions-orders.md](whmcs-functions-orders.md) - Order management
- [whmcs-functions-services.md](whmcs-functions-services.md) - Service provisioning
- [whmcs-schema-products.md](whmcs-schema-products.md) - Product database schema