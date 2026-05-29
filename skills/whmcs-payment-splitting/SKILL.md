# WHMCS Payment Splitting

## Overview
Master skill for splitting payments across multiple gateways or methods in WHMCS.

## Payment Splitter

```php
<?php
// /includes/managers/PaymentSplitter.php

namespace WHMCS\Billing;

class PaymentSplitter
{
    public function splitPayment(int $invoiceId, array $allocations): array
    {
        $invoice = \WHMCS\Billing\Invoice::find($invoiceId);
        
        if (!$invoice) {
            return ["success" => false, "error" => "Invoice not found"];
        }
        
        $totalAllocated = array_sum(array_column($allocations, "amount"));
        $balance = $invoice->getBalance();
        
        if (abs($totalAllocated - $balance) > 0.01) {
            return ["success" => false, "error" => "Allocation mismatch"];
        }
        
        $results = [];
        
        foreach ($allocations as $allocation) {
            $result = $this->processAllocation($invoiceId, $allocation);
            $results[] = $result;
        }
        
        return ["success" => true, "results" => $results];
    }
    
    private function processAllocation(int $invoiceId, array $allocation): array
    {
        $gateway = new \WHMCS\Module\Gateway();
        $gateway->load($allocation["gateway"]);
        
        $result = $gateway->capture([
            "amount" => $allocation["amount"],
            "currency" => $allocation["currency"] ?? "USD",
        ]);
        
        if ($result["status"] === "success") {
            addInvoicePayment(
                $invoiceId,
                $result["transid"],
                $allocation["amount"],
                0,
                $allocation["gateway"]
            );
        }
        
        return [
            "gateway" => $allocation["gateway"],
            "amount" => $allocation["amount"],
            "success" => $result["status"] === "success",
            "transaction_id" => $result["transid"] ?? null,
        ];
    }
    
    public function splitByPercentage(int $invoiceId, array $gateways): array
    {
        $invoice = \WHMCS\Billing\Invoice::find($invoiceId);
        $balance = $invoice->getBalance();
        
        $allocations = [];
        
        foreach ($gateways as $gateway => $percentage) {
            $amount = round($balance * ($percentage / 100), 2);
            $allocations[] = [
                "gateway" => $gateway,
                "amount" => $amount,
            ];
        }
        
        return $this->splitPayment($invoiceId, $allocations);
    }
}
```

## Best Practices

1. **Total Verification**: Always verify total equals invoice balance
2. **Partial Failures**: Handle partial payment failures
3. **Gateway Support**: Ensure all gateways support split payments
4. **Atomic Transactions**: Process all or none
5. **Receipts**: Generate receipts for each payment
6. **Error Handling**: Rollback on critical failures
7. **Logging**: Log all split payment attempts
8. **User Experience**: Show clear allocation breakdown
