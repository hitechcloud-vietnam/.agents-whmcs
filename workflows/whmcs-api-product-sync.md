# WHMCS API Product Sync Workflow

## Purpose
Guide developers through synchronizing products between WHMCS and external systems.

## Prerequisites
- WHMCS installation
- Product API access
- External inventory system
- PHP skills

## Steps

### Phase 1: Product Data Mapping

1. Product fields mapping
   ```
   Product Sync Fields:
   - Product ID / Name
   - Pricing (monthly, annual)
   - Product group
   - Configurable options
   - Custom fields
   ```

2. Sync configuration
   ```php
   class ProductSync {
       private $api;
       
       public function syncProducts($externalProducts): array {
           $results = ['synced' => 0, 'created' => 0, 'errors' => []];
           
           foreach ($externalProducts as $product) {
               $whmcsProductId = $this->findExistingProduct($product['sku']);
               
               if ($whmcsProductId) {
                   $this->updateProduct($whmcsProductId, $product);
                   $results['synced']++;
               } else {
                   $this->createProduct($product);
                   $results['created']++;
               }
           }
           
           return $results;
       }
   }
   ```

### Phase 2: Price Synchronization

1. Price sync
   ```php
   public function syncPricing($productId, $pricingData) {
       $this->api->call('UpdateProduct', [
           'pid' => $productId,
           'pricing' => [
               'monthly' => $pricingData['monthly'],
               'quarterly' => $pricingData['quarterly'],
               'annually' => $pricingData['annually'],
               'biennially' => $pricingData['biennially'],
               'triennially' => $pricingData['triennially'],
           ],
       ]);
   }
   ```

### Phase 3: Inventory Sync

1. Stock management
   ```php
   public function syncInventory($productId, $quantity) {
       // Update stock quantities
       // Handle out of stock states
       // Sync with provisioning limits
   }
   ```

## Related Workflows
- whmcs-api-product-sync (this workflow)
- whmcs-api-integration
- whmcs-api-order-sync
