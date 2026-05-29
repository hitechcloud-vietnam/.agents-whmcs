# WHMCS Refund Processing

## Concept Explanation
Refund processing handles customer requests for money returns with clear policies and proper audit trails.

### Refund Types
- **Full Refund**: Complete return of payment
- **Partial Refund**: Proportional return
- **Prorated Refund**: Time-based calculation
- **Store Credit**: Refund as account credit

## Code Patterns

```php
<?php
class RefundProcessor {
    
    public static function processRefund($invoiceId, $amount = null, $method = 'original') {
        $invoice = getInvoice($invoiceId);
        $amount = $amount ?? $invoice['total'];
        
        $refundId = insert_query('tbl_refunds', [
            'invoice_id' => $invoiceId,
            'amount' => $amount,
            'method' => $method,
            'status' => 'processing'
        ]);
        
        if ($method === 'credit') {
            CreditManager::addCredit($invoice['userid'], $amount, "Refund for Invoice #" . $invoice['invoicenum']);
        } else {
            refundToGateway($invoiceId, $amount);
        }
        
        update_query('tbl_refunds', ['status' => 'completed'], ['id' => $refundId]);
        return $refundId;
    }
    
    public static function calculateProratedRefund($serviceId) {
        $proration = ProrationCalculator::calculateUnusedCredit($serviceId);
        return $proration['credit_amount'];
    }
}
```
