# WHMCS Chargeback Handling

## Concept Explanation
Chargebacks occur when customers dispute charges with their bank. Effective handling minimizes financial loss.

### Chargeback Process
1. **Alert**: Notification of dispute
2. **Evidence**: Gather transaction proof
3. **Response**: Submit evidence to processor
4. **Resolution**: Win/lose/settle

## Code Patterns

```php
<?php
class ChargebackHandler {
    
    public static function processChargeback($data) {
        $payment = getPaymentByTransaction($data['transaction_id']);
        if (!$payment) return false;
        
        insert_query('tbl_chargebacks', [
            'transaction_id' => $data['transaction_id'],
            'invoice_id' => $payment['invoice_id'],
            'client_id' => $payment['client_id'],
            'amount' => $data['amount'],
            'status' => 'pending'
        ]);
        
        // Reverse payment and suspend service
        reverseInvoicePayment($payment['invoice_id'], $data['transaction_id']);
        
        foreach (getServicesByInvoice($payment['invoice_id']) as $service) {
            localAPI('ModuleSuspend', ['serviceid' => $service['id']]);
        }
        
        return true;
    }
}
```
