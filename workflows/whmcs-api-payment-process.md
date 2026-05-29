# WHMCS API Payment Process Workflow

## Purpose
Guide developers through processing payments via WHMCS API.

## Prerequisites
- WHMCS installation
- Payment gateway configured
- Invoice ID
- PHP skills

## Steps

### Phase 1: Payment Processing

1. Add payment to invoice
   ```php
   function applyPayment($invoiceId, $amount, $paymentMethod): array {
       return localAPI('AddInvoicePayment', [
           'invoiceid' => $invoiceId,
           'amount' => $amount,
           'paymentmethod' => $paymentMethod,
           'transid' => generateTransactionId(),
       ]);
   }
   ```

2. Capture payment
   ```php
   $result = localAPI('CapturePayment', [
       'invoiceid' => $invoiceId,
       'gateway' => 'stripe',
   ]);
   ```

### Phase 2: Refund Processing

1. Process refund
   ```php
   function refundPayment($invoiceId, $amount, $reason): array {
       return localAPI('RefundPayment', [
           'invoiceid' => $invoiceId,
           'amount' => $amount,
           'reason' => $reason,
       ]);
   }
   ```

## Related Workflows
- whmcs-api-invoice-sync
- whmcs-api-integration
- whmcs-api-reporting
