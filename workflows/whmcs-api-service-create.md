# WHMCS API Service Create Workflow

## Purpose
Guide developers through creating services via WHMCS API.

## Prerequisites
- WHMCS installation
- Order/Service API access
- Product ID and configuration
- PHP skills

## Steps

### Phase 1: Service Creation API

1. Create service via API
   ```php
   function createServiceViaApi($clientId, $productId, $params = []): array {
       $postData = [
           'clientid' => $clientId,
           'pid' => $productId,
           'billingcycle' => $params['billingcycle'] ?? 'Monthly',
           'domain' => $params['domain'] ?? '',
           'customfields' => $params['customfields'] ?? [],
           'configoptions' => $params['configoptions'] ?? [],
       ];
       
       return localAPI('AddOrder', $postData);
   }
   ```

2. Full service creation
   ```php
   $result = localAPI('AddOrder', [
       'clientid' => 123,
       'pid' => 1,
       'billingcycle' => 'Annual',
       'domaintype' => 'register',
       'domain' => 'example.com',
       'paymentmethod' => 'paypal',
       'customfields' => [
           'field1' => 'value1',
       ],
       'configoptions' => [
           'option_id' => 'selection_id',
       ],
   ]);
   
   if ($result['result'] === 'success') {
       $orderId = $result['orderid'];
   }
   ```

### Phase 2: Service Configuration

1. Configure service options
   ```php
   public function configureService($serviceId, $config) {
       return localAPI('ModuleChangePackage', [
           'serviceid' => $serviceId,
           'newproductid' => $config['new_pid'],
           'newconfigoptions' => $config['options'],
       ]);
   }
   ```

## Related Workflows
- whmcs-api-order-sync
- whmcs-api-product-sync
- whmcs-api-integration
