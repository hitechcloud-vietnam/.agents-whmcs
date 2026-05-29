# WHMCS Product Catalog

## Concept
External catalog synchronization.

## Code
```php
<?php
class ProductCatalogSync {
    public static function importProducts($source) {
        $products = $source->getProducts();
        foreach ($products as $product) {
            $existing = getProductBySKU($product["sku"]);
            if ($existing) {
                update_query("tblproducts", $product, ["id" => $existing["id"]]);
            } else {
                insert_query("tblproducts", $product);
            }
        }
    }
}
```
