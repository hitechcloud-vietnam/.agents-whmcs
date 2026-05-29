# WHMCS Accounting Sync

## Concept
Accounting software integration.

## Code
```php
<?php
class AccountingSync {
    public static function syncInvoice($invoiceId) {
        $invoice = getInvoice($invoiceId);
        $this->accounting->createInvoice(["reference" => $invoice["invoicenum"], "amount" => $invoice["total"]]);
    }
    
    public static function syncPayment($paymentId) {
        $payment = getPayment($paymentId);
        $this->accounting->recordPayment(["reference" => $payment["transid"], "amount" => $payment["amount"]]);
    }
}
```
