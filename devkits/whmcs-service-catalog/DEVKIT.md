# WHMCS Service Catalog Module - DEVKIT

## Module Information
- **Name**: Service Catalog
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Product catalog management with categorization for WHMCS

## Installation
1. Copy to `/modules/addons/service_catalog/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('ClientAreaPage', 1, function($vars) {
    return ['catalogCategories' => ServiceCatalog::getCategories()];
});

add_hook('ProductConfigured', 1, function($vars) {
    ServiceCatalog::trackProductView($vars);
});
```

### includes/ServiceCatalog.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class ServiceCatalog
{
    private static $table = 'mod_service_catalog';
    
    public static function getCategories()
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . "mod_service_catalog_categories
            WHERE active = 1
            ORDER BY sort_order ASC
        ");
        
        $categories = [];
        while ($row = mysql_fetch_array($result)) {
            $row['products'] = self::getCategoryProducts($row['id']);
            $categories[] = $row;
        }
        
        return $categories;
    }
    
    public static function getCategoryProducts($categoryId)
    {
        $result = full_query("
            SELECT p.*, pc.category_id
            FROM " . TABLE_PREFIX . "tblproducts p
            JOIN " . TABLE_PREFIX . "mod_service_catalog_products pc ON p.id = pc.product_id
            WHERE pc.category_id = " . (int)$categoryId . "
            AND p.hidden = 0
            ORDER BY p.order ASC
        ");
        
        $products = [];
        while ($row = mysql_fetch_array($result)) {
            $products[] = $row;
        }
        
        return $products;
    }
    
    public static function trackProductView($vars)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . "mod_service_catalog_views
            (product_id, user_id, viewed_at)
            VALUES (" . (int)($vars['pid'] ?? 0) . ", " . (int)($_SESSION['uid'] ?? 0) . ", NOW())
        ");
    }
    
    public static function getPopularProducts($limit = 10)
    {
        $result = full_query("
            SELECT p.*, COUNT(v.id) as view_count
            FROM " . TABLE_PREFIX . "tblproducts p
            LEFT JOIN " . TABLE_PREFIX . "mod_service_catalog_views v ON p.id = v.product_id
            WHERE p.hidden = 0
            GROUP BY p.id
            ORDER BY view_count DESC
            LIMIT " . (int)$limit
        );
        
        $products = [];
        while ($row = mysql_fetch_array($result)) {
            $products[] = $row;
        }
        
        return $products;
    }
    
    public static function searchProducts($query, $filters = [])
    {
        $where = "WHERE p.hidden = 0 AND (p.name LIKE '%" . db_escape_string($query) . "%' OR p.description LIKE '%" . db_escape_string($query) . "%')";
        
        if (!empty($filters['category'])) {
            $where .= " AND pc.category_id = " . (int)$filters['category'];
        }
        
        if (!empty($filters['min_price'])) {
            $where .= " AND p.monthly FROM >= " . (float)$filters['min_price'];
        }
        
        $result = full_query("
            SELECT p.*, c.name as category_name
            FROM " . TABLE_PREFIX . "tblproducts p
            LEFT JOIN " . TABLE_PREFIX . "mod_service_catalog_products pc ON p.id = pc.product_id
            LEFT JOIN " . TABLE_PREFIX . "mod_service_catalog_categories c ON pc.category_id = c.id
            " . $where . "
            ORDER BY p.name ASC
        ");
        
        $products = [];
        while ($row = mysql_fetch_array($result)) {
            $products[] = $row;
        }
        
        return $products;
    }
}

function service_catalog_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_service_catalog_categories (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            description TEXT,
            icon VARCHAR(100),
            sort_order INT DEFAULT 0,
            active TINYINT(1) DEFAULT 1,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_service_catalog_products (
            id INT AUTO_INCREMENT PRIMARY KEY,
            product_id INT NOT NULL,
            category_id INT NOT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_service_catalog_views (
            id INT AUTO_INCREMENT PRIMARY KEY,
            product_id INT NOT NULL,
            user_id INT DEFAULT 0,
            viewed_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    return ['status' => 'success', 'description' => 'Service Catalog activated'];
}

function service_catalog_deactivate()
{
    return ['status' => 'success', 'description' => 'Service Catalog deactivated'];
}

function service_catalog_config()
{
    return [
        'name' => 'Service Catalog',
        'description' => 'Product catalog management with categorization',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'show_popular' => [
                'Type' => 'yesno',
                'FriendlyName' => 'Show Popular Products',
                'Default' => '1'
            ],
            'items_per_page' => [
                'Type' => 'text',
                'FriendlyName' => 'Items Per Page',
                'Default' => '12'
            ]
        ]
    ];
}
```