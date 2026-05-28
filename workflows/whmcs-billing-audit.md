# WHMCS Billing Audit Workflow

## Purpose

Comprehensive guide to conducting billing audits in WHMCS, including invoice reconciliation, payment verification, revenue recognition, and compliance verification.

## Prerequisites

- WHMCS admin access
- Database read access
- Financial reporting requirements understanding
- Optional: Accounting software integration

## Workflow Steps

### Step 1: Invoice Reconciliation

Verify all invoices match financial records:

```php
// modules/addons/billing_audit/reconciliation.php

class BillingReconciliation
{
    private $auditPeriod;
    
    public function __construct(array $dateRange)
    {
        $this->auditPeriod = $dateRange;
    }
    
    /**
     * Perform complete reconciliation
     */
    public function reconcile(): array
    {
        $results = [
            'period' => $this->auditPeriod,
            'started_at' => date('Y-m-d H:i:s'),
            'invoices' => [],
            'payments' => [],
            'discrepancies' => [],
        ];
        
        // Reconcile invoices
        $results['invoices'] = $this->reconcileInvoices();
        
        // Reconcile payments
        $results['payments'] = $this->reconcilePayments();
        
        // Reconcile transactions
        $results['transactions'] = $this->reconcileTransactions();
        
        // Find discrepancies
        $results['discrepancies'] = $this->findDiscrepancies();
        
        $results['completed_at'] = date('Y-m-d H:i:s');
        
        // Log audit
        $this->logAudit($results);
        
        return $results;
    }
    
    /**
     * Reconcile invoices
     */
    private function reconcileInvoices(): array
    {
        $invoices = Capsule::table('tblinvoices')
            ->whereBetween('date', [$this->auditPeriod['start'], $this->auditPeriod['end']])
            ->get();
        
        $results = [
            'total' => count($invoices),
            'by_status' => [],
            'total_amount' => 0,
            'issues' => [],
        ];
        
        foreach ($invoices as $invoice) {
            // Status breakdown
            $results['by_status'][$invoice->status] = 
                ($results['by_status'][$invoice->status] ?? 0) + 1;
            
            $results['total_amount'] += $invoice->total;
            
            // Validate invoice
            $validation = $this->validateInvoice($invoice);
            
            if (!$validation['valid']) {
                $results['issues'][] = [
                    'invoice_id' => $invoice->id,
                    'invoice_num' => $invoice->invoicenum,
                    'issues' => $validation['issues'],
                ];
            }
        }
        
        return $results;
    }
    
    /**
     * Validate individual invoice
     */
    private function validateInvoice($invoice): array
    {
        $issues = [];
        
        // Check required fields
        if (empty($invoice->userid)) {
            $issues[] = 'Missing client ID';
        }
        
        if ($invoice->total < 0) {
            $issues[] = 'Negative total amount';
        }
        
        // Check line items match total
        $lineItems = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoice->id)
            ->selectRaw('SUM(amount) as total')
            ->first();
        
        if (abs($lineItems->total - $invoice->total) > 0.01) {
            $issues[] = "Line items sum ({$lineItems->total}) doesn't match total ({$invoice->total})";
        }
        
        // Check tax calculation
        if ($invoice->tax > 0 || $invoice->tax2 > 0) {
            $taxValidation = $this->validateTaxCalculation($invoice);
            if (!$taxValidation['valid']) {
                $issues[] = $taxValidation['issue'];
            }
        }
        
        return [
            'valid' => empty($issues),
            'issues' => $issues,
        ];
    }
    
    /**
     * Reconcile payments against invoices
     */
    private function reconcilePayments(): array
    {
        $payments = Capsule::table('tblaccounts')
            ->whereBetween('date', [$this->auditPeriod['start'], $this->auditPeriod['end']])
            ->where('transid', '!=', '')
            ->get();
        
        $results = [
            'total_payments' => count($payments),
            'total_amount' => 0,
            'by_gateway' => [],
            'unmatched' => [],
        ];
        
        foreach ($payments as $payment) {
            $results['total_amount'] += $payment->amountin;
            
            // Group by gateway
            $gateway = $payment->paymentmethod ?? 'unknown';
            $results['by_gateway'][$gateway] = 
                ($results['by_gateway'][$gateway] ?? 0) + $payment->amountin;
            
            // Match to invoice
            $matched = $this->matchPaymentToInvoice($payment);
            
            if (!$matched) {
                $results['unmatched'][] = [
                    'payment_id' => $payment->id,
                    'amount' => $payment->amountin,
                    'gateway' => $payment->paymentmethod,
                    'transid' => $payment->transid,
                ];
            }
        }
        
        return $results;
    }
    
    /**
     * Match payment to invoice
     */
    private function matchPaymentToInvoice($payment): bool
    {
        // Find invoice for this payment
        $invoicePayment = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $payment->invoiceid)
            ->where('type', 'Invoice')
            ->first();
        
        if (!$invoicePayment) {
            // Check accounts table for direct link
            $directMatch = Capsule::table('tblaccounts')
                ->where('invoiceid', $payment->invoiceid)
                ->where('id', '!=', $payment->id)
                ->exists();
            
            return $directMatch;
        }
        
        return true;
    }
}
```

### Step 2: Revenue Recognition

Implement revenue recognition tracking:

```php
// modules/addons/revenue_recognition/recognition.php

class RevenueRecognition
{
    /**
     * Calculate recognized revenue for period
     */
    public function calculateRecognizedRevenue(array $dateRange): array
    {
        $results = [
            'total_revenue' => 0,
            'recognized' => 0,
            'deferred' => 0,
            'by_service_type' => [],
        ];
        
        // Get all paid invoices in period
        $paidInvoices = Capsule::table('tblinvoices')
            ->whereBetween('date', [$dateRange['start'], $dateRange['end']])
            ->where('status', 'Paid')
            ->get();
        
        foreach ($paidInvoices as $invoice) {
            $results['total_revenue'] += $invoice->total;
            
            // Get invoice items
            $items = Capsule::table('tblinvoiceitems')
                ->where('invoiceid', $invoice->id)
                ->get();
            
            foreach ($items as $item) {
                $recognition = $this->calculateItemRecognition($item, $invoice);
                
                $results['recognized'] += $recognition['recognized'];
                $results['deferred'] += $recognition['deferred'];
                
                // Track by service type
                $serviceType = $this->getServiceType($item);
                $results['by_service_type'][$serviceType] = 
                    ($results['by_service_type'][$serviceType] ?? 0) + $recognition['recognized'];
            }
        }
        
        return $results;
    }
    
    /**
     * Calculate recognition for single item
     */
    private function calculateItemRecognition($item, $invoice): array
    {
        // Determine recognition method based on billing cycle
        $billingCycle = $this->getBillingCycle($item);
        
        switch ($billingCycle) {
            case 'One Time':
                return [
                    'recognized' => $item->amount,
                    'deferred' => 0,
                ];
                
            case 'Monthly':
                // Full recognition in current month
                return [
                    'recognized' => $item->amount,
                    'deferred' => 0,
                ];
                
            case 'Quarterly':
                // 1/3 recognized each month
                $monthlyAmount = $item->amount / 3;
                return [
                    'recognized' => $monthlyAmount,
                    'deferred' => $item->amount - $monthlyAmount,
                ];
                
            case 'Annually':
                // 1/12 recognized each month
                $monthlyAmount = $item->amount / 12;
                return [
                    'recognized' => $monthlyAmount,
                    'deferred' => $item->amount - $monthlyAmount,
                ];
                
            default:
                return [
                    'recognized' => $item->amount,
                    'deferred' => 0,
                ];
        }
    }
}
```

### Step 3: Refund and Credit Analysis

Analyze refunds and credits:

```php
// modules/addons/billing_audit/refunds.php

class RefundCreditAnalyzer
{
    /**
     * Analyze all refunds and credits
     */
    public function analyzeRefunds(array $dateRange): array
    {
        $results = [
            'total_refunds' => 0,
            'refund_count' => 0,
            'total_credits' => 0,
            'credit_count' => 0,
            'refund_rate' => 0,
            'by_reason' => [],
            'by_gateway' => [],
            'suspicious' => [],
        ];
        
        // Get refunds (negative transactions)
        $refunds = Capsule::table('tblaccounts')
            ->whereBetween('date', [$dateRange['start'], $dateRange['end']])
            ->where('amountout', '>', 0)
            ->get();
        
        foreach ($refunds as $refund) {
            $results['total_refunds'] += $refund->amountout;
            $results['refund_count']++;
            
            // Categorize by reason
            $reason = $this->categorizeRefund($refund);
            $results['by_reason'][$reason] = 
                ($results['by_reason'][$reason] ?? 0) + $refund->amountout;
            
            // Categorize by gateway
            $gateway = $refund->paymentmethod ?? 'unknown';
            $results['by_gateway'][$gateway] = 
                ($results['by_gateway'][$gateway] ?? 0) + $refund->amountout;
            
            // Check for suspicious patterns
            if ($this->isSuspiciousRefund($refund)) {
                $results['suspicious'][] = $refund;
            }
        }
        
        // Calculate refund rate
        $totalRevenue = $this->getTotalRevenue($dateRange);
        $results['refund_rate'] = $totalRevenue > 0 
            ? ($results['total_refunds'] / $totalRevenue) * 100 
            : 0;
        
        return $results;
    }
    
    /**
     * Categorize refund reason
     */
    private function categorizeRefund($refund): string
    {
        // Look for refund reason in notes or transaction
        $notes = $refund->notes ?? '';
        
        if (stripos($notes, 'duplicate') !== false) {
            return 'Duplicate Payment';
        }
        
        if (stripos($notes, 'customer request') !== false || 
            stripos($notes, 'cancellation') !== false) {
            return 'Customer Request';
        }
        
        if (stripos($notes, 'fraud') !== false || stripos($notes, 'unauthorized') !== false) {
            return 'Fraud/Unauthorized';
        }
        
        if (stripos($notes, 'service issue') !== false || stripos($notes, 'quality') !== false) {
            return 'Service Issue';
        }
        
        return 'Other';
    }
    
    /**
     * Check for suspicious refund patterns
     */
    private function isSuspiciousRefund($refund): bool
    {
        // Large refund amount
        if ($refund->amountout > 1000) {
            return true;
        }
        
        // Multiple refunds to same client
        $clientRefunds = Capsule::table('tblaccounts')
            ->where('userid', $refund->userid)
            ->where('amountout', '>', 0)
            ->count();
        
        if ($clientRefunds > 3) {
            return true;
        }
        
        // Refund within 24 hours of payment
        $originalPayment = Capsule::table('tblaccounts')
            ->where('userid', $refund->userid)
            ->where('amountin', '>=', $refund->amountout)
            ->where('date', '<=', $refund->date)
            ->orderBy('date', 'desc')
            ->first();
        
        if ($originalPayment) {
            $hoursDiff = (strtotime($refund->date) - strtotime($originalPayment->date)) / 3600;
            if ($hoursDiff < 24) {
                return true;
            }
        }
        
        return false;
    }
}
```

### Step 4: Tax Compliance Verification

Verify tax calculations and compliance:

```php
// modules/addons/tax_compliance/verifier.php

class TaxComplianceVerifier
{
    /**
     * Verify tax calculations for audit period
     */
    public function verifyTaxCalculations(array $dateRange): array
    {
        $results = [
            'total_tax_collected' => 0,
            'by_jurisdiction' => [],
            'errors' => [],
            'tax_exempt_clients' => [],
        ];
        
        $invoices = Capsule::table('tblinvoices')
            ->whereBetween('date', [$dateRange['start'], $dateRange['end']])
            ->where('status', 'Paid')
            ->get();
        
        foreach ($invoices as $invoice) {
            $results['total_tax_collected'] += $invoice->tax + $invoice->tax2;
            
            // Get client tax status
            $client = Capsule::table('tblclients')
                ->where('id', $invoice->userid)
                ->first();
            
            if ($client->taxexempt) {
                $results['tax_exempt_clients'][] = $client->id;
                
                // Verify no tax was charged
                if ($invoice->tax > 0 || $invoice->tax2 > 0) {
                    $results['errors'][] = [
                        'invoice_id' => $invoice->id,
                        'type' => 'tax_exempt_error',
                        'message' => 'Tax exempt client charged tax',
                    ];
                }
            }
            
            // Verify tax rates
            $this->verifyTaxRate($invoice, $client, $results);
        }
        
        return $results;
    }
    
    /**
     * Verify individual tax rate application
     */
    private function verifyTaxRate($invoice, $client, &$results): void
    {
        // Get applicable tax rates
        $taxLevel1 = Capsule::table('tbltax')
            ->where('country', $client->country)
            ->where('state', $client->state)
            ->where('level', 1)
            ->first();
        
        $taxLevel2 = Capsule::table('tbltax')
            ->where('country', $client->country)
            ->where('state', $client->state)
            ->where('level', 2)
            ->first();
        
        // Get invoice items total (excluding tax)
        $subtotal = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoice->id)
            ->where('type', '!=', 'Tax')
            ->selectRaw('SUM(amount) as total')
            ->first();
        
        // Calculate expected tax
        $expectedTax1 = $subtotal->total * (($taxLevel1->taxrate ?? 0) / 100);
        $expectedTax2 = $subtotal->total * (($taxLevel2->taxrate ?? 0) / 100);
        
        // Compare with actual
        if (abs($expectedTax1 - $invoice->tax) > 0.01) {
            $results['errors'][] = [
                'invoice_id' => $invoice->id,
                'type' => 'tax_mismatch',
                'expected' => $expectedTax1,
                'actual' => $invoice->tax,
            ];
        }
        
        // Track by jurisdiction
        $jurisdiction = ($client->state ?: $client->country);
        $results['by_jurisdiction'][$jurisdiction] = 
            ($results['by_jurisdiction'][$jurisdiction] ?? 0) + $invoice->tax;
    }
}
```

### Step 5: Audit Report Generation

Generate comprehensive audit reports:

```php
// modules/addons/billing_audit/report.php

/**
 * Generate comprehensive billing audit report
 */
function generateBillingAuditReport(array $dateRange): array
{
    $report = [
        'report_date' => date('Y-m-d H:i:s'),
        'audit_period' => $dateRange,
        'sections' => [],
    ];
    
    // Invoice reconciliation
    $reconciliation = new BillingReconciliation($dateRange);
    $report['sections']['reconciliation'] = $reconciliation->reconcile();
    
    // Revenue recognition
    $revenue = new RevenueRecognition();
    $report['sections']['revenue'] = $revenue->calculateRecognizedRevenue($dateRange);
    
    // Refund analysis
    $refunds = new RefundCreditAnalyzer();
    $report['sections']['refunds'] = $refunds->analyzeRefunds($dateRange);
    
    // Tax compliance
    $taxVerifier = new TaxComplianceVerifier();
    $report['sections']['tax'] = $taxVerifier->verifyTaxCalculations($dateRange);
    
    // Summary statistics
    $report['summary'] = [
        'total_revenue' => $report['sections']['revenue']['total_revenue'],
        'total_refunds' => $report['sections']['refunds']['total_refunds'],
        'net_revenue' => $report['sections']['revenue']['total_revenue'] - 
                        $report['sections']['refunds']['total_refunds'],
        'refund_rate' => $report['sections']['refunds']['refund_rate'],
        'total_tax_collected' => $report['sections']['tax']['total_tax_collected'],
        'error_count' => count($report['sections']['reconciliation']['discrepancies']) +
                        count($report['sections']['tax']['errors']),
    ];
    
    // Store report
    Capsule::table('mod_billing_audit_reports')->insert([
        'report_data' => json_encode($report),
        'period_start' => $dateRange['start'],
        'period_end' => $dateRange['end'],
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    return $report;
}
```

## Best Practices

1. **Regular audits** - Monthly reconciliation recommended
2. **Automated checks** - Reduce human error
3. **Clear documentation** - Record all adjustments
4. **Segregation of duties** - Multiple people in process
5. **Threshold alerts** - Flag unusual transactions
6. **Trend analysis** - Compare periods
7. **Tax compliance** - Verify calculations
8. **Clean audit trail** - Track all changes

## Common Pitfalls to Avoid

1. **Manual errors** - Spreadsheet mistakes
2. **Missing transactions** - Gaps in records
3. **Tax miscalculations** - Compliance issues
4. **Refund abuse** - Not monitoring patterns
5. **Unreconciled amounts** - Small discrepancies ignored
6. **No documentation** - Can't explain variances
7. **Delayed audits** - Problems found too late
8. **Incomplete scope** - Missing data sources
