# WHMCS Digital Marketplace Module

```php
<?php
/**
 * WHMCS Digital Marketplace Module
 * 
 * Digital marketplace for selling downloadable products,
 * software licenses, and digital services.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function marketplace_MetaData() {
    return array('DisplayName' => 'Digital Marketplace', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function marketplace_ConfigArray() {
    return array('FriendlyName' => array('Type' => 'System', 'Value' => 'Digital Marketplace'),
        'EnableReviews' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable product reviews'),
        'EnableDownloadLimit' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Limit download次数'),
        'MaxDownloads' => array('Type' => 'text', 'Size' => '10', 'Default' => '5', 'Description' => 'Max downloads per purchase'),
        'EnableLicense' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable license generation'),
        'CommissionRate' => array('Type' => 'text', 'Size' => '10', 'Default' => '15', 'Description' => 'Platform commission %'));
}

function marketplace_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_marketplace_products', "
            CREATE TABLE `mod_marketplace_products` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `product_key` VARCHAR(100) UNIQUE NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `category` VARCHAR(100) NOT NULL,
                `price` DECIMAL(10,2) NOT NULL,
                `sale_price` DECIMAL(10,2) NULL,
                `files` JSON NULL,
                `images` JSON NULL,
                `license_template` TEXT NULL,
                `version` VARCHAR(50) DEFAULT '1.0.0',
                `vendor_id` INT NOT NULL,
                `status` ENUM('draft', 'pending', 'approved', 'rejected', 'suspended') DEFAULT 'draft',
                `rating` DECIMAL(3,2) DEFAULT 0.00,
                `rating_count` INT DEFAULT 0,
                `sales_count` INT DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_marketplace_purchases', "
            CREATE TABLE `mod_marketplace_purchases` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `product_id` INT NOT NULL,
                `order_id` INT NOT NULL,
                `user_id` INT NOT NULL,
                `license_key` VARCHAR(255) NULL,
                `download_count` INT DEFAULT 0,
                `max_downloads` INT DEFAULT 5,
                `purchased_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `last_download` DATETIME NULL,
                UNIQUE KEY `unique_order_product` (`order_id`, `product_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_marketplace_reviews', "
            CREATE TABLE `mod_marketplace_reviews` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `product_id` INT NOT NULL,
                `user_id` INT NOT NULL,
                `rating` TINYINT NOT NULL,
                `title` VARCHAR(255) NULL,
                `comment` TEXT NULL,
                `status` ENUM('pending', 'approved', 'rejected') DEFAULT 'pending',
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_product_user` (`product_id`, `user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Digital Marketplace module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function marketplace_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function marketplace_CreateProduct($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $key = 'prod-' . substr(md5(uniqid()), 0, 12);
        Capsule::table('mod_marketplace_products')->insert(array('product_key' => $key, 'name' => $data['name'], 'description' => $data['description'] ?? '', 'category' => $data['category'], 'price' => $data['price'], 'sale_price' => $data['sale_price'] ?? null, 'files' => json_encode($data['files'] ?? array()), 'images' => json_encode($data['images'] ?? array()), 'license_template' => $data['license_template'] ?? null, 'vendor_id' => $data['vendor_id'], 'status' => 'approved'));
        return array('success' => true, 'product_key' => $key);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function marketplace_GetProduct($key) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $product = Capsule::table('mod_marketplace_products')->where('product_key', $key)->first();
    if ($product) { $product->files = json_decode($product->files, true); $product->images = json_decode($product->images, true); }
    return $product;
}

function marketplace_GetProducts($filters = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_marketplace_products')->where('status', 'approved');
    if (isset($filters['category'])) { $query->where('category', $filters['category']); }
    if (isset($filters['vendor_id'])) { $query->where('vendor_id', $filters['vendor_id']); }
    if (isset($filters['search'])) { $query->where('name', 'LIKE', '%' . $filters['search'] . '%'); }
    return $query->orderBy('sales_count', 'desc')->get();
}

function marketplace_GetCategories() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_marketplace_products')->where('status', 'approved')->selectRaw('category, COUNT(*) as count')->groupBy('category')->get();
}

function marketplace_PurchaseProduct($productKey, $orderId, $userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $product = marketplace_GetProduct($productKey);
        if (!$product) return array('success' => false, 'error' => 'Product not found');
        $licenseKey = marketplace_GenerateLicense($product);
        Capsule::table('mod_marketplace_purchases')->insert(array('product_id' => $product->id, 'order_id' => $orderId, 'user_id' => $userId, 'license_key' => $licenseKey));
        Capsule::table('mod_marketplace_products')->where('id', $product->id)->update(array('sales_count' => $product->sales_count + 1));
        return array('success' => true, 'license_key' => $licenseKey);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function marketplace_GenerateLicense($product) {
    if (!$product->license_template) return 'LIC-' . strtoupper(substr(md5(uniqid()), 0, 16));
    $license = $product->license_template;
    $license = str_replace('{YYYY}', date('Y'), $license);
    $license = str_replace('{MM}', date('m'), $license);
    $license = str_replace('{DD}', date('d'), $license);
    $license = str_replace('{RANDOM}', strtoupper(substr(md5(uniqid()), 0, 8)), $license);
    return $license;
}

function marketplace_GetPurchase($orderId, $productId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_marketplace_purchases')->where('order_id', $orderId)->where('product_id', $productId)->first();
}

function marketplace_GetUserPurchases($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_marketplace_purchases')->join('mod_marketplace_products', 'mod_marketplace_purchases.product_id', '=', 'mod_marketplace_products.id')->where('mod_marketplace_purchases.user_id', $userId)->select('mod_marketplace_products.*', 'mod_marketplace_purchases.license_key', 'mod_marketplace_purchases.purchased_at')->get();
}

function marketplace_RecordDownload($orderId, $productId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $purchase = marketplace_GetPurchase($orderId, $productId);
    if (!$purchase) return array('success' => false, 'error' => 'Purchase not found');
    if ($purchase->max_downloads > 0 && $purchase->download_count >= $purchase->max_downloads) { return array('success' => false, 'error' => 'Download limit reached'); }
    Capsule::table('mod_marketplace_purchases')->where('id', $purchase->id)->update(array('download_count' => $purchase->download_count + 1, 'last_download' => date('Y-m-d H:i:s')));
    return array('success' => true, 'download_count' => $purchase->download_count + 1, 'remaining' => $purchase->max_downloads - $purchase->download_count - 1);
}

function marketplace_AddReview($productId, $userId, $rating, $title, $comment) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_marketplace_reviews')->insert(array('product_id' => $productId, 'user_id' => $userId, 'rating' => $rating, 'title' => $title, 'comment' => $comment));
        $reviews = Capsule::table('mod_marketplace_reviews')->where('product_id', $productId)->where('status', 'approved')->get();
        $avgRating = array_sum(array_column($reviews, 'rating')) / count($reviews);
        Capsule::table('mod_marketplace_products')->where('id', $productId)->update(array('rating' => round($avgRating, 2), 'rating_count' => count($reviews)));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function marketplace_GetReviews($productId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_marketplace_reviews')->where('product_id', $productId)->where('status', 'approved')->get();
}
