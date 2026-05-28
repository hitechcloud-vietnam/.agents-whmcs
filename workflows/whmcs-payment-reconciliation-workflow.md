# WHMCS Payment Reconciliation Workflow

## Purpose

Reconcile WHMCS payment records with payment gateway transactions, identify discrepancies, and ensure accurate financial accounting.

## Prerequisites

- WHMCS with payment records
- Payment gateway with API access
- Admin access to financial records
- Reconciliation schedule defined

## Workflow Steps

### Step 1: Configure Reconciliation Settings

Set up reconciliation configuration:

```php
// Database configuration for reconciliation
INSERT INTO tblconfiguration (setting, value) VALUES 
('ReconciliationEnabled', 'on'),
('AutoReconciliation', 'on'),
('ReconciliationThreshold', '0.01'),
('ReconciliationSchedule', 'daily'),
('DiscrepancyAlertEmail', 'accounting@example.com');

// Create reconciliation tracking tables
Capsule::schema()->create('mod_reconciliation_runs', function($t) {
    $t->increments('id');
    $t->date('reconciliation_date');
    $t->string('status'); // running, completed, failed
    $t->integer('total_transactions');
    $t->integer('matched_count');
    $t->integer('discrepancy_count');
    $t->decimal('total_amount', 12, 2);
    $t->decimal('discrepancy_amount', 12, 2);
    $t->text('details');
    $t->timestamp('created_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});

Capsule::schema()->create('mod_reconciliation_discrepancies', function($t) {
    $t->increments('id');
    $t->integer('run_id');
    $t->string('type'); // missing_in_whmcs, missing_in_gateway, amount_mismatch
    $t->string('whmcs_transaction_id');
    $t->string('gateway_transaction_id');
    $t->decimal('whmcs_amount', 10, 2);
    $t->decimal('gateway_amount', 10, 2);
    $t->decimal('difference', 10, 2);
    $t->string('status'); // pending, resolved, escalated
    $t->text('notes');
    $t->timestamp('resolved_at')->nullable();
});
```

### Step 2: Create Reconciliation Service

Build reconciliation functionality:

```php
// File: /includes/classes/PaymentReconciliation.php

namespace WHMCS\Billing;

class PaymentReconciliation {
    
    private $threshold;
    private $results = [];
    
    public function __construct($threshold = 0.01) {
        $this->threshold = $threshold;
    }
    
    public function runReconciliation($startDate, $endDate, $gateway = null) {
        $runId = $this->startReconciliationRun($startDate, $endDate);
        
        try {
            // Get all paid invoices in period
            $whmcsPayments = $this->getWHMCSPayments($startDate, $endDate, $gateway);
            
            // Get gateway transactions
            $gatewayTransactions = $this->getGatewayTransactions($startDate, $endDate, $gateway);
            
            // Match transactions
            $matched = $this->matchTransactions($whmcsPayments, $gatewayTransactions);
            
            // Find discrepancies
            $discrepancies = $this->findDiscrepancies($matched, $whmcsPayments, $gatewayTransactions);
            
            // Update run with results
            $this->completeReconciliationRun($runId, [
                'total' => count($whmcsPayments),
                'matched' => count($matched),
                'discrepancies' => $discrepancies
            ]);
            
            // Alert on discrepancies
            if (count($discrepancies) > 0) {
                $this->alertDiscrepancies($discrepancies);
            }
            
            return [
                'run_id' => $runId,
                'matched' => count($matched),
                'discrepancies' => count($discrepancies),
                'details' => $discrepancies
            ];
            
        } catch (Exception $e) {
            $this->failReconciliationRun($runId, $e->getMessage());
            throw $e;
        }
    }
    
    private function getWHMCSPayments($startDate, $endDate, $gateway = null) {
        $query = Capsule::table('tblaccounts')
            ->whereBetween('date', [$startDate, $endDate])
            ->where('amount', '>', 0);
        
        if ($gateway) {
            $query->where('gateway', $gateway);
        }
        
        return $query->get()->map(function($payment) {
            return [
                'id' => $payment->id,
                'invoice_id' => $payment->invoiceid,
                'amount' => (float) $payment->amount,
                'date' => $payment->date,
                'gateway' => $payment->gateway,
                'transid' => $payment->transid
            ];
        })->toArray();
    }
    
    private function getGatewayTransactions($startDate, $endDate, $gateway = null) {
        // This would call the gateway API to get transactions
        // Example for Stripe:
        
        if ($gateway === 'stripe' || !$gateway) {
            $stripeApiKey = Capsule::table('tblpaymentgateways')
                ->where('gateway', 'stripe')
                ->where('setting', 'api_key')
                ->value('value');
            
            $ch = curl_init();
            curl_setopt_array($ch, [
                CURLOPT_URL => 'https://api.stripe.com/v1/charges?' . http_build_query([
                    'created' => [
                        'gte' => strtotime($startDate),
                        'lte' => strtotime($endDate)
                    ],
                    'limit' => 100
                ]),
                CURLOPT_RETURNTRANSFER => true,
                CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $stripeApiKey]
            ]);
            
            $response = curl_exec($ch);
            curl_close($ch);
            
            $data = json_decode($response, true);
            
            return array_map(function($charge) {
                return [
                    'gateway_id' => $charge['id'],
                    'amount' => $charge['amount'] / 100,
                    'currency' => $charge['currency'],
                    'status' => $charge['status'],
                    'created' => date('Y-m-d H:i:s', $charge['created']),
                    'metadata' => $charge['metadata'] ?? []
                ];
            }, $data['data'] ?? []);
        }
        
        return [];
    }
    
    private function matchTransactions($whmcsPayments, $gatewayTransactions) {
        $matched = [];
        
        foreach ($whmcsPayments as $whmcs) {
            foreach ($gatewayTransactions as $index => $gateway) {
                // Match by gateway transaction ID
                if ($whmcs['transid'] && $whmcs['transid'] === $gateway['gateway_id']) {
                    $matched[] = [
                        'whmcs' => $whmcs,
                        'gateway' => $gateway,
                        'difference' => abs($whmcs['amount'] - $gateway['amount']),
                        'matched' => abs($whmcs['amount'] - $gateway['amount']) <= $this->threshold
                    ];
                    unset($gatewayTransactions[$index]);
                    break;
                }
                
                // Match by amount and date
                if ($whmcs['amount'] === $gateway['amount'] && 
                    date('Y-m-d', strtotime($whmcs['date'])) === date('Y-m-d', strtotime($gateway['created']))) {
                    $matched[] = [
                        'whmcs' => $whmcs,
                        'gateway' => $gateway,
                        'difference' => 0,
                        'matched' => true
                    ];
                    unset($gatewayTransactions[$index]);
                    break;
                }
            }
        }
        
        return $matched;
    }
    
    private function findDiscrepancies($matched, $whmcsPayments, $gatewayTransactions) {
        $discrepancies = [];
        
        // Amount mismatches in matched transactions
        foreach ($matched as $match) {
            if (!$match['matched']) {
                $discrepancies[] = [
                    'type' => 'amount_mismatch',
                    'whmcs_id' => $match['whmcs']['id'],
                    'gateway_id' => $match['gateway']['gateway_id'],
                    'whmcs_amount' => $match['whmcs']['amount'],
                    'gateway_amount' => $match['gateway']['amount'],
                    'difference' => $match['difference']
                ];
            }
        }
        
        // Transactions in WHMCS but not in gateway
        $matchedWhmcsIds = array_column($matched, 'whmcs');
        foreach ($whmcsPayments as $whmcs) {
            $found = false;
            foreach ($matchedWhmcsIds as $matched) {
                if ($matched['id'] === $whmcs['id']) {
                    $found = true;
                    break;
                }
            }
            if (!$found) {
                $discrepancies[] = [
                    'type' => 'missing_in_gateway',
                    'whmcs_id' => $whmcs['id'],
                    'whmcs_amount' => $whmcs['amount']
                ];
            }
        }
        
        // Transactions in gateway but not in WHMCS
        foreach ($gatewayTransactions as $gateway) {
            $discrepancies[] = [
                'type' => 'missing_in_whmcs',
                'gateway_id' => $gateway['gateway_id'],
                'gateway_amount' => $gateway['amount']
            ];
        }
        
        // Store discrepancies in database
        foreach ($discrepancies as $discrepancy) {
            $this->recordDiscrepancy($discrepancy);
        }
        
        return $discrepancies;
    }
    
    private function recordDiscrepancy($discrepancy) {
        Capsule::table('mod_reconciliation_discrepancies')->insert([
            'run_id' => $this->currentRunId,
            'type' => $discrepancy['type'],
            'whmcs_transaction_id' => $discrepancy['whmcs_id'] ?? null,
            'gateway_transaction_id' => $discrepancy['gateway_id'] ?? null,
            'whmcs_amount' => $discrepancy['whmcs_amount'] ?? null,
            'gateway_amount' => $discrepancy['gateway_amount'] ?? null,
            'difference' => $discrepancy['difference'] ?? null,
            'status' => 'pending'
        ]);
    }
    
    private function startReconciliationRun($startDate, $endDate) {
        return Capsule::table('mod_reconciliation_runs')->insertGetId([
            'reconciliation_date' => date('Y-m-d'),
            'status' => 'running',
            'details' => json_encode(['start' => $startDate, 'end' => $endDate])
        ]);
    }
    
    private function completeReconciliationRun($runId, $results) {
        Capsule::table('mod_reconciliation_runs')
            ->where('id', $runId)
            ->update([
                'status' => 'completed',
                'total_transactions' => $results['total'],
                'matched_count' => $results['matched'],
                'discrepancy_count' => $results['discrepancies'],
                'updated_at' => Capsule::raw('NOW()')
            ]);
    }
    
    private function alertDiscrepancies($discrepancies) {
        $alertEmail = Capsule::table('tblconfiguration')
            ->where('setting', 'DiscrepancyAlertEmail')
            ->value('value');
        
        if ($alertEmail) {
            $message = "Payment Reconciliation Discrepancies Found\n\n";
            $message .= "Total discrepancies: " . count($discrepancies) . "\n\n";
            
            foreach ($discrepancies as $d) {
                $message .= "- Type: {$d['type']}\n";
                if (isset($d['difference'])) {
                    $message .= "  Difference: $" . number_format($d['difference'], 2) . "\n";
                }
                $message .= "\n";
            }
            
            sendAdminNotification('email', 'Payment Reconciliation Alert', $message);
        }
    }
}
```

### Step 3: Create Automated Reconciliation Hook

Schedule reconciliation:

```php
// File: /includes/hooks/reconciliation_automation.php

add_hook('DailyCronJob', 1, function($vars) {
    $autoReconcile = Capsule::table('tblconfiguration')
        ->where('setting', 'AutoReconciliation')
        ->value('value');
    
    if ($autoReconcile !== 'on') {
        return;
    }
    
    $yesterday = date('Y-m-d', strtotime('-1 day'));
    
    $reconciliation = new \WHMCS\Billing\PaymentReconciliation();
    
    try {
        $result = $reconciliation->runReconciliation(
            $yesterday . ' 00:00:00',
            $yesterday . ' 23:59:59'
        );
        
        logActivity("Reconciliation completed: {$result['matched']} matched, {$result['discrepancies']} discrepancies");
        
        if ($result['discrepancies'] > 0) {
            // Send summary to accounting
            sendTemplatedEmail('ReconciliationComplete', 0, [
                'date' => $yesterday,
                'matched' => $result['matched'],
                'discrepancies' => $result['discrepancies']
            ]);
        }
        
    } catch (Exception $e) {
        logActivity("Reconciliation failed: " . $e->getMessage());
        sendAdminNotification('email', 'Reconciliation Failed', $e->getMessage());
    }
});
```

### Step 4: Create Discrepancy Resolution Functions

Handle reconciliation exceptions:

```php
// File: /includes/classes/DiscrepancyResolver.php

namespace WHMCS\Billing;

class DiscrepancyResolver {
    
    public function resolveDiscrepancy($discrepancyId, $resolution, $notes = '') {
        $discrepancy = Capsule::table('mod_reconciliation_discrepancies')
            ->where('id', $discrepancyId)
            ->first();
        
        switch ($resolution) {
            case 'adjust_whmcs':
                // Adjust WHMCS to match gateway
                $this->adjustWHMCSTransaction($discrepancy);
                break;
                
            case 'adjust_gateway':
                // Record adjustment for gateway discrepancy
                $this->recordGatewayAdjustment($discrepancy);
                break;
                
            case 'ignore':
                // Log and ignore (documented decision)
                $this->logIgnoredDiscrepancy($discrepancy, $notes);
                break;
                
            case 'create_dispute':
                // Flag for dispute investigation
                $this->flagForDispute($discrepancy);
                break;
        }
        
        Capsule::table('mod_reconciliation_discrepancies')
            ->where('id', $discrepancyId)
            ->update([
                'status' => 'resolved',
                'notes' => $notes,
                'resolved_at' => Capsule::raw('NOW()')
            ]);
        
        logActivity("Reconciliation discrepancy #{$discrepancyId} resolved: {$resolution}");
    }
    
    private function adjustWHMCSTransaction($discrepancy) {
        if ($discrepancy->whmcs_transaction_id && $discrepancy->gateway_amount) {
            // Create adjustment transaction
            Capsule::table('tblaccounts')->insert([
                'userid' => $this->getUserIdFromInvoice($discrepancy->whmcs_transaction_id),
                'invoiceid' => $discrepancy->whmcs_transaction_id,
                'description' => 'Reconciliation adjustment',
                'amount' => $discrepancy->gateway_amount - $discrepancy->whmcs_amount,
                'date' => date('Y-m-d H:i:s'),
                'gateway' => 'adjustment'
            ]);
        }
    }
    
    private function recordGatewayAdjustment($discrepancy) {
        Capsule::table('mod_reconciliation_adjustments')->insert([
            'discrepancy_id' => $discrepancy->id,
            'type' => 'gateway_adjustment',
            'amount' => $discrepancy->difference,
            'created_at' => Capsule::raw('NOW()')
        ]);
    }
    
    private function logIgnoredDiscrepancy($discrepancy, $notes) {
        Capsule::table('mod_reconciliation_log')->insert([
            'discrepancy_id' => $discrepancy->id,
            'action' => 'ignored',
            'reason' => $notes,
            'admin_id' => $_SESSION['adminid'],
            'created_at' => Capsule::raw('NOW()')
        ]);
    }
    
    private function flagForDispute($discrepancy) {
        Capsule::table('mod_reconciliation_disputes')->insert([
            'discrepancy_id' => $discrepancy->id,
            'status' => 'open',
            'created_at' => Capsule::raw('NOW()')
        ]);
    }
}
```

### Step 5: Generate Reconciliation Reports

Create reconciliation summaries:

```php
// File: /includes/hooks/reconciliation_reporting.php

add_hook('MonthlyCronJob', 1, function($vars) {
    // Generate monthly reconciliation summary
    $startDate = date('Y-m-01', strtotime('-1 month'));
    $endDate = date('Y-m-t', strtotime('-1 month'));
    
    $runs = Capsule::table('mod_reconciliation_runs')
        ->whereBetween('reconciliation_date', [$startDate, $endDate])
        ->get();
    
    $summary = [
        'period' => ['start' => $startDate, 'end' => $endDate],
        'total_runs' => count($runs),
        'total_transactions' => 0,
        'total_matched' => 0,
        'total_discrepancies' => 0,
        'total_discrepancy_amount' => 0,
        'resolved_discrepancies' => 0,
        'pending_discrepancies' => 0
    ];
    
    foreach ($runs as $run) {
        $summary['total_transactions'] += $run->total_transactions;
        $summary['total_matched'] += $run->matched_count;
        $summary['total_discrepancies'] += $run->discrepancy_count;
        $summary['total_discrepancy_amount'] += $run->discrepancy_amount;
    }
    
    $resolved = Capsule::table('mod_reconciliation_discrepancies')
        ->join('mod_reconciliation_runs', 'mod_reconciliation_discrepancies.run_id', '=', 'mod_reconciliation_runs.id')
        ->where('mod_reconciliation_discrepancies.status', 'resolved')
        ->whereBetween('mod_reconciliation_runs.reconciliation_date', [$startDate, $endDate])
        ->count();
    
    $pending = Capsule::table('mod_reconciliation_discrepancies')
        ->join('mod_reconciliation_runs', 'mod_reconciliation_discrepancies.run_id', '=', 'mod_reconciliation_runs.id')
        ->where('mod_reconciliation_discrepancies.status', 'pending')
        ->whereBetween('mod_reconciliation_runs.reconciliation_date', [$startDate, $endDate])
        ->count();
    
    $summary['resolved_discrepancies'] = $resolved;
    $summary['pending_discrepancies'] = $pending;
    
    // Save report
    file_put_contents(
        __DIR__ . "/../storage/logs/reconciliation/monthly_{$startDate}.json",
        json_encode($summary, JSON_PRETTY_PRINT)
    );
    
    // Email to accounting
    $reportText = "Monthly Reconciliation Summary\n";
    $reportText .= "Period: {$startDate} to {$endDate}\n\n";
    $reportText .= "Total Runs: {$summary['total_runs']}\n";
    $reportText .= "Transactions Processed: {$summary['total_transactions']}\n";
    $reportText .= "Matched: {$summary['total_matched']}\n";
    $reportText .= "Discrepancies: {$summary['total_discrepancies']}\n";
    $reportText .= "Discrepancy Amount: $" . number_format($summary['total_discrepancy_amount'], 2) . "\n";
    $reportText .= "Resolved: {$summary['resolved_discrepancies']}\n";
    $reportText .= "Pending: {$summary['pending_discrepancies']}\n";
    
    sendAdminNotification('email', 'Monthly Reconciliation Report', $reportText);
});
```

## Verification Checklist

- [ ] Reconciliation service configured
- [ ] Daily reconciliation running on schedule
- [ ] Transactions matched correctly
- [ ] Discrepancies identified and recorded
- [ ] Alerts sent for discrepancies
- [ ] Resolution functions working
- [ ] Monthly reports generated
- [ ] Reports emailed to accounting
- [ ] Historical data accessible
- [ ] Resolution documented

## Related Skills and Documentation

- [WHMCS Payment Processing](whmcs-payment-processing-workflow.md)
- [WHMCS Billing Reporting](whmcs-billing-reporting-workflow.md)
- [WHMCS Billing Audit](whmcs-billing-audit-workflow.md)
- WHMCS Documentation: Transaction Management
- Payment Gateway Documentation: Transaction APIs

## Notes

- Run reconciliation daily for accurate records
- Investigate all discrepancies promptly
- Keep resolution documentation for audit
- Set appropriate threshold to avoid false positives
- Consider timezone differences in matching
- Automate as much as possible for consistency
- Review reconciliation reports monthly