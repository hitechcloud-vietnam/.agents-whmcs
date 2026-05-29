# WHMCS API Invoice Sync Workflow

## Purpose
Guide developers through synchronizing invoices between WHMCS and external accounting systems.

## Prerequisites
- WHMCS installation
- Invoice API access
- External accounting software
- PHP skills

## Steps

### Phase 1: Invoice Data Structure

1. Invoice sync fields
   ```
   Invoice Data:
   - Invoice ID / Number
   - Client information
   - Line items
   - Tax amounts
   - Total/Due
   - Payment status
   - Due date
   ```

2. Invoice sync implementation
   ```php
   class InvoiceSyncService {
       public function syncInvoice($invoiceId): array {
           $invoice = localAPI('GetInvoice', ['invoiceid' => $invoiceId]);
           
           $accountingData = [
               'invoice_number' => $invoice['invoicenum'],
               'client_id' => $invoice['userid'],
               'issue_date' => $invoice['date'],
               'due_date' => $invoice['duedate'],
               'line_items' => $this->extractLineItems($invoice),
               'subtotal' => $invoice['subtotal'],
               'tax' => $invoice['tax'],
               'total' => $invoice['total'],
               'paid' => $invoice['paid'],
               'balance' => $invoice['balance'],
           ];
           
           return $this->accountingApi->createInvoice($accountingData);
       }
   }
   ```

### Phase 2: Payment Recording

1. Record payments
   ```php
   add_hook('InvoicePaid', 1, function($vars) {
       $invoiceId = $vars['invoice_id'];
       $invoice = localAPI('GetInvoice', ['invoiceid' => $invoiceId]);
       
       $accountingService->recordPayment([
           'invoice_id' => $invoiceId,
           'amount' => $invoice['total'],
           'payment_date' => date('Y-m-d'),
           'payment_method' => $invoice['paymentmethod'],
       ]);
   });
   ```

## Related Workflows
- whmcs-api-order-sync
- whmcs-api-integration
- whmcs-api-payment-process
