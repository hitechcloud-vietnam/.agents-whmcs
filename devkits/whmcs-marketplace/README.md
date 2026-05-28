# WHMCS Digital Marketplace Module

Digital marketplace for selling downloadable products, software licenses, and digital services.

## Features

- Product management
- Download tracking
- License key generation
- Review system
- Sales statistics
- Vendor management

## Installation

Copy module to `/path/to/whmcs/modules/servers/marketplace/` and activate.

## Usage

```php
// Create product
marketplace_CreateProduct(array(
    'name' => 'Premium Plugin',
    'description' => 'A premium WordPress plugin',
    'category' => 'plugins',
    'price' => 49.99,
    'files' => array('file.zip', 'docs.pdf'),
    'images' => array('screenshot1.png'),
    'license_template' => 'PRO-{YYYY}-{RANDOM}',
    'vendor_id' => $adminId
));

// Get product
$product = marketplace_GetProduct($productKey);

// Get products
$products = marketplace_GetProducts(array('category' => 'plugins', 'search' => 'search term'));

// Get categories
$categories = marketplace_GetCategories();

// Purchase product
$purchase = marketplace_PurchaseProduct($productKey, $orderId, $userId);
// Returns: license_key

// Get user purchases
$purchases = marketplace_GetUserPurchases($userId);

// Record download
$result = marketplace_RecordDownload($orderId, $productId);
// Returns: download_count, remaining

// Add review
marketplace_AddReview($productId, $userId, 5, 'Great product!', 'Works perfectly.');

// Get reviews
$reviews = marketplace_GetReviews($productId);
```

## API Functions

| Function | Description |
|----------|-------------|
| `marketplace_CreateProduct()` | Create marketplace product |
| `marketplace_GetProduct()` | Get product by key |
| `marketplace_GetProducts()` | Get products with filters |
| `marketplace_GetCategories()` | Get all categories |
| `marketplace_PurchaseProduct()` | Record purchase |
| `marketplace_GetPurchase()` | Get purchase record |
| `marketplace_GetUserPurchases()` | Get user purchases |
| `marketplace_RecordDownload()` | Record download |
| `marketplace_AddReview()` | Add product review |
| `marketplace_GetReviews()` | Get product reviews |
