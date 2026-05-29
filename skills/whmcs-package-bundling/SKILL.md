# WHMCS Package Bundling

## Concept Explanation
Package bundling combines multiple products or services at a discounted price to increase average order value.

### Bundle Types
- **Standard Bundles**: Fixed product combinations
- **Dynamic Bundles**: Customer selects from options
- **Add-on Bundles**: Pre-configured add-on packages

## Code Patterns

```php
<?php
// includes/BundleManager.php

class BundleManager {
    
    public static function createBundle($name, $products, $bundlePrice, $description = '') {
        $bundleId = insert_query('tblbundles', [
            'name' => $name,
            'description' => $description,
            'price' => $bundlePrice,
            'created_at' => date('Y-m-d H:i:s')
        ]);
        
        foreach ($products as $productId => $options) {
            insert_query('tblbundle_products', [
                'bundle_id' => $bundleId,
                'product_id' => $productId,
                'config_options' => json_encode($options)
            ]);
        }
        
        return $bundleId;
    }
    
    public static function calculateBundleSavings($bundleId) {
        $bundle = getBundle($bundleId);
        $products = getBundleProducts($bundleId);
        
        $totalIndividual = 0;
        foreach ($products as $product) {
            $totalIndividual += getProductPrice($product['product_id']);
        }
        
        $savings = $totalIndividual - $bundle['price'];
        $savingsPercent = ($savings / $totalIndividual) * 100;
        
        return [
            'individual_total' => $totalIndividual,
            'bundle_price' => $bundle['price'],
            'savings' => $savings,
            'savings_percent' => round($savingsPercent, 1)
        ];
    }
}
```
