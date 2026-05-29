# WHMCS API Order Sync Workflow

## Purpose
Guide developers through synchronizing orders between WHMCS and external systems.

## Prerequisites
- WHMCS installation
- Order API access
- External order management
- PHP skills

## Steps

### Phase 1: Order Data Structure

1. Order sync fields
   ```
   Order Data:
   - Order ID
   - Client ID
   - Products
   - Order status
   - Total amount
   - Payment status
   - Dates
   ```

2. Order sync implementation
   ```php
   class OrderSyncService {
       public function syncOrder($orderId): array {
           $whmcsOrder = $this->getWHMCOrder($orderId);
           
           $externalOrder = [
               'order_id' => $whmcsOrder['id'],
               'customer_email' => $whmcsOrder['email'],
               'items' => $this->extractItems($whmcsOrder),
               'total' => $whmcsOrder['total'],
               'status' => $this->mapStatus($whmcsOrder['status']),
           ];
           
           return $this->externalApi->createOrder($externalOrder);
       }
       
       private function mapStatus($whmcsStatus): string {
           $map = [
               'Pending' => 'pending',
               'Active' => 'completed',
               'Cancelled' => 'cancelled',
               'Fraud' => 'disputed',
           ];
           return $map[$whmcsStatus] ?? 'unknown';
       }
   }
   ```

### Phase 2: Order Status Sync

1. Status updates
   ```php
   add_hook('OrderStatusChange', 1, function($vars) {
       $orderId = $vars['order_id'];
       $newStatus = $vars['new_status'];
       
       $externalService->updateOrderStatus($orderId, $newStatus);
   });
   ```

## Related Workflows
- whmcs-api-product-sync
- whmcs-api-invoice-sync
- whmcs-api-integration
