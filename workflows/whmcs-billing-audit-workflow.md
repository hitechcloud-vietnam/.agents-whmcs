# WHMCS Billing Audit Workflow

## Purpose

Conduct comprehensive billing audits, verify financial accuracy, identify discrepancies, ensure compliance with accounting standards, and maintain audit trails for regulatory requirements.

## Prerequisites

- WHMCS admin access with billing permissions
- Access to financial records and reports
- Audit schedule defined
- Understanding of accounting principles
- Documentation of billing policies

## Workflow Steps

### Step 1: Prepare Audit Documentation

Set up audit infrastructure:

```php
// Database configuration for billing audit
INSERT INTO tblconfiguration (setting, value) VALUES 
('BillingAuditEnabled', 'on'),
('AuditRetentionDays', '2555'),
('AuditReportEmail', 'audit@example.com,accounting@example.com'),
('AuditorAccessLevel', 'full'); // full, read_only

// Create billing audit tables
Capsule::schema()->create('mod_billing_audit_log', function($t) {
    $t->increments('id');
    $t->string('audit_type'); // invoice, payment, refund, credit
    $t->integer('reference_id'); // invoice_id, payment_id, etc.
    $t->string('action');
    $t->string('old_value')->nullable();
    $t->string('new_value')->nullable();
    $t->integer('performed_by');
    $t->string('ip_address');
    $t->text('notes')->nullable();
    $t->timestamp('created_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});

Capsule::schema()->create('mod_audit_schedules', function($t) {
    $t->increments('id');
    $t->string('audit_name');
    $t->string('audit_type'); // daily, weekly, monthly, quarterly, annual
    $t->date('last_run');
    $t->date('next_run');
    $t->string('status'); // active, paused, completed
    $t->json('parameters');
    $t->timestamp('created_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});

Capsule::schema()->create('mod_audit_findings', function($t) {
    $t->increments('id');
    $t->integer('audit_schedule_id');
    $t->string('severity'); // critical, high, medium, low
    $t->string('category');
    $t->text('description');
    $t->text('recommendation');
    $t->string('status'); // open, in_progress, resolved, dismissed
    $t->integer('resolved_by');
    $t->timestamp('resolved_at')->nullable();
    $t->text('resolution_notes')->nullable();
    $t->timestamp('created_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});
```

### Step 2: Create Audit Logging System

Implement comprehensive audit logging:

```php
// File: /includes/classes/BillingAuditLogger.php

namespace WHMCS\Audit;

class BillingAuditLogger {
    
    public function logInvoiceCreation($invoiceId, $adminId = null) {
        $invoice = Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->first();
        
        Capsule::table('mod_billing_audit_log')->insert([
            'audit_type' => 'invoice',
            'reference_id' => $invoiceId,
            'action' => 'created',
            'new_value' => json_encode([
                'user_id' => $invoice->userid,
                'total' => $invoice->total,
                'status' => $invoice->status
            ]),
            'performed_by' => $adminId ?? $_SESSION['adminid'] ?? 0,
            'ip_address' => $this->getClientIP()
        ]);
    }
    
    public function logInvoiceModification($invoiceId, $changes, $adminId = null) {
        foreach ($changes as $field => $oldValue) {
            Capsule::table('mod_billing_audit_log')->insert([
                'audit_type' => 'invoice',
                'reference_id' => $invoiceId,
                'action' => 'modified',
                'old_value' => $oldValue,
                'new_value' => $changes[$field],
                'performed_by' => $adminId ?? $_SESSION['adminid'] ?? 0,
                'ip_address' => $this->getClientIP(),
                'notes' => "Field: {$field}"
            ]);
        }
    }
    
    public function logPayment($invoiceId, $paymentData, $adminId = null) {
        Capsule::table('mod_billing_audit_log')->insert([
            'audit_type' => 'payment',
            'reference_id' => $invoiceId,
            'action' => 'payment_received',
            'new_value' => json_encode($paymentData),
            'performed_by' => $adminId ?? $_SESSION['adminid'] ?? 0,
            'ip_address' => $this->getClientIP()
        ]);
        
        // Log gateway transaction
        Capsule::table('mod_billing_audit_log')->insert([
            'audit_type' => 'payment',
            'reference_id' => $invoiceId,
            'action' => 'gateway_transaction',
            'new_value' => json_encode([
                'gateway' => $paymentData['gateway'],
                'transaction_id' => $paymentData['transid']
            ]),
            'performed_by' => 0, // System action
            'ip_address' => 'system'
        ]);
    }
    
    public function logRefund($invoiceId, $refundData, $adminId = null) {
        Capsule::table('mod_billing_audit_log')->insert([
            'audit_type' => 'refund',
            'reference_id' => $invoiceId,
            'action' => 'refund_processed',
            'old_value' => json_encode(['invoice_total' => $refundData['invoice_total']]),
            'new_value' => json_encode($refundData),
            'performed_by' => $adminId ?? $_SESSION['adminid'] ?? 0,
            'ip_address' => $this->getClientIP(),
            'notes' => "Reason: {$refundData['reason']}"
        ]);
    }
    
    public function logCreditAdjustment($clientId, $creditData, $adminId = null) {
        Capsule::table('mod_billing_audit_log')->insert([
            'audit_type' => 'credit',
            'reference_id' => $clientId,
            'action' => $creditData['action'], // add, remove, apply
            'old_value' => $creditData['old_balance'] ?? null,
            'new_value' => $creditData['new_balance'] ?? null,
            'performed_by' => $adminId ?? $_SESSION['adminid'] ?? 0,
            'ip_address' => $this->getClientIP(),
            'notes' => $creditData['description'] ?? ''
        ]);
    }
    
    public function logInvoiceDeletion($invoiceId, $reason, $adminId = null) {
        $invoice = Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->first();
        
        Capsule::table('mod_billing_audit_log')->insert([
            'audit_type' => 'invoice',
            'reference_id' => $invoiceId,
            'action' => 'deleted',
            'old_value' => json_encode([
                'user_id' => $invoice->userid,
                'total' => $invoice->total,
                'status' => $invoice->status
            ]),
            'performed_by' => $adminId ?? $_SESSION['adminid'] ?? 0,
            'ip_address' => $this->getClientIP(),
            'notes' => "Reason: {$reason}"
        ]);
    }
    
    private function getClientIP() {
        return $_SERVER['REMOTE_ADDR'] ?? 'unknown';
    }
}
```

### Step 3: Create Audit Hooks

Automate audit logging:

```php
// File: /includes/hooks/billing_audit_automation.php

add_hook('InvoiceCreated', 1, function($vars) {
    $logger = new \WHMCS\Audit\BillingAuditLogger();
    $logger->logInvoiceCreation($vars['invoiceid']);
});

add_hook('InvoicePaid', 1, function($vars) {
    $logger = new \WHMCS\Audit\BillingAuditLogger();
    
    $payment = Capsule::table('tblaccounts')
        ->where('invoiceid', $vars['invoiceid'])
        ->orderBy('id', 'desc')
        ->first();
    
    $logger->logPayment($vars['invoiceid'], [
        'gateway' => $vars['paymentmethod'] ?? 'unknown',
        'transid' => $vars['transactionid'] ?? '',
        'amount' => $vars['amount'] ?? 0,
        'date' => date('Y-m-d H:i:s')
    ]);
});

add_hook('InvoiceRefunded', 1, function($vars) {
    $logger = new \WHMCS\Audit\BillingAuditLogger();
    
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $vars['invoiceid'])
        ->first();
    
    $logger->logRefund($vars['invoiceid'], [
        'amount' => $vars['amount'],
        'reason' => 'Customer refund request',
        'invoice_total' => $invoice->total
    ]);
});

add_hook('CreditApplied', 1, function($vars) {
    $logger = new \WHMCS\Audit\BillingAuditLogger();
    
    $logger->logCreditAdjustment($vars['client_id'], [
        'action' => 'applied',
        'amount' => $vars['amount'],
        'invoice_id' => $vars['invoice_id'],
        'new_balance' => $vars['new_balance'] ?? 0
    ]);
});
```

### Step 4: Create Audit Report Generator

Build comprehensive audit reports:

```php
// File: /includes/classes/AuditReportGenerator.php

namespace WHMCS\Audit;

class AuditReportGenerator {
    
    public function generateInvoiceAuditReport($startDate, $endDate) {
        $invoices = Capsule::table('tblinvoices')
            ->whereBetween('date', [$startDate, $endDate])
            ->get();
        
        $report = [
            'period' => ['start' => $startDate, 'end' => $endDate],
            'summary' => [
                'total_invoices' => 0,
                'total_amount' => 0,
                'by_status' => []
            ],
            'deleted_invoices' => [],
            'modified_invoices' => [],
            'discrepancies' => []
        ];
        
        foreach ($invoices as $invoice) {
            $report['summary']['total_invoices']++;
            $report['summary']['total_amount'] += $invoice->total;
            
            $status = $invoice->status;
            if (!isset($report['summary']['by_status'][$status])) {
                $report['summary']['by_status'][$status] = ['count' => 0, 'amount' => 0];
            }
            $report['summary']['by_status'][$status]['count']++;
            $report['summary']['by_status'][$status]['amount'] += $invoice->total;
        }
        
        // Get deleted invoices from audit log
        $deleted = Capsule::table('mod_billing_audit_log')
            ->where('audit_type', 'invoice')
            ->where('action', 'deleted')
            ->whereBetween('created_at', [$startDate, $endDate])
            ->get();
        
        foreach ($deleted as $d) {
            $report['deleted_invoices'][] = [
                'invoice_id' => $d->reference_id,
                'deleted_at' => $d->created_at,
                'performed_by' => $d->performed_by
            ];
        }
        
        // Get modifications
        $modified = Capsule::table('mod_billing_audit_log')
            ->where('audit_type', 'invoice')
            ->where('action', 'modified')
            ->whereBetween('created_at', [$startDate, $endDate])
            ->get();
        
        foreach ($modified as $m) {
            $report['modified_invoices'][] = [
                'invoice_id' => $m->reference_id,
                'field' => $m->notes,
                'old_value' => $m->old_value,
                'new_value' => $m->new_value,
                'modified_at' => $m->created_at,
                'performed_by' => $m->performed_by
            ];
        }
        
        return $report;
    }
    
    public function generatePaymentAuditReport($startDate, $endDate) {
        $payments = Capsule::table('tblaccounts')
            ->whereBetween('date', [$startDate, $endDate])
            ->get();
        
        $report = [
            'period' => ['start' => $startDate, 'end' => $endDate],
            'summary' => [
                'total_payments' => 0,
                'total_amount' => 0,
                'by_gateway' => []
            ],
            'unmatched_payments' => [],
            'reconciled_payments' => 0
        ];
        
        foreach ($payments as $payment) {
            $report['summary']['total_payments']++;
            $report['summary']['total_amount'] += $payment->amount;
            
            $gateway = $payment->gateway ?? 'unknown';
            if (!isset($report['summary']['by_gateway'][$gateway])) {
                $report['summary']['by_gateway'][$gateway] = ['count' => 0, 'amount' => 0];
            }
            $report['summary']['by_gateway'][$gateway]['count']++;
            $report['summary']['by_gateway'][$gateway]['amount'] += $payment->amount;
            
            // Check if payment has matching gateway transaction
            if (empty($payment->transid)) {
                $report['unmatched_payments'][] = [
                    'payment_id' => $payment->id,
                    'invoice_id' => $payment->invoiceid,
                    'amount' => $payment->amount,
                    'date' => $payment->date
                ];
            } else {
                $report['reconciled_payments']++;
            }
        }
        
        return $report;
    }
    
    public function generateCreditAuditReport($startDate, $endDate) {
        $credits = Capsule::table('mod_billing_audit_log')
            ->where('audit_type', 'credit')
            ->whereBetween('created_at', [$startDate, $endDate])
            ->get();
        
        $report = [
            'period' => ['start' => $startDate, 'end' => $endDate],
            'total_credits' => count($credits),
            'by_action' => ['add' => 0, 'remove' => 0, 'apply' => 0],
            'suspicious_activities' => []
        ];
        
        foreach ($credits as $credit) {
            $data = json_decode($credit->new_value, true);
            $action = $credit->action;
            
            if (isset($report['by_action'][$action])) {
                $report['by_action'][$action]++;
            }
            
            // Check for suspicious activities
            // Large credit additions
            if ($action === 'add' && isset($data['new_balance']) && $data['new_balance'] > 1000) {
                $report['suspicious_activities'][] = [
                    'client_id' => $credit->reference_id,
                    'action' => 'large_credit_addition',
                    'amount' => $data['new_balance'],
                    'performed_by' => $credit->performed_by,
                    'date' => $credit->created_at
                ];
            }
        }
        
        return $report;
    }
}
```

### Step 5: Create Scheduled Audits

Implement automated audit schedules:

```php
// File: /includes/hooks/audit_scheduling.php

add_hook('DailyCronJob', 1, function($vars) {
    // Run daily billing audit
    $yesterday = date('Y-m-d', strtotime('-1 day'));
    
    $invoiceReport = generateInvoiceAuditReport($yesterday, $yesterday . ' 23:59:59');
    $paymentReport = generatePaymentAuditReport($yesterday, $yesterday . ' 23:59:59');
    
    // Log findings
    $findings = validateBillingIntegrity($invoiceReport, $paymentReport);
    
    foreach ($findings as $finding) {
        Capsule::table('mod_audit_findings')->insert($finding);
    }
    
    // Alert on critical issues
    $critical = array_filter($findings, function($f) {
        return $f['severity'] === 'critical';
    });
    
    if (count($critical) > 0) {
        sendAdminNotification(
            'email',
            'Critical Billing Audit Alert',
            "Found " . count($critical) . " critical billing issues requiring immediate attention."
        );
    }
});

add_hook('MonthlyCronJob', 1, function($vars) {
    // Generate comprehensive monthly audit report
    $startDate = date('Y-m-01', strtotime('-1 month'));
    $endDate = date('Y-m-t', strtotime('-1 month'));
    
    $auditReport = new \WHMCS\Audit\AuditReportGenerator();
    
    $invoiceReport = $auditReport->generateInvoiceAuditReport($startDate, $endDate);
    $paymentReport = $auditReport->generatePaymentAuditReport($startDate, $endDate);
    $creditReport = $auditReport->generateCreditAuditReport($startDate, $endDate);
    
    // Save comprehensive report
    $report = [
        'period' => $startDate . ' to ' . $endDate,
        'generated_at' => date('Y-m-d H:i:s'),
        'invoice_audit' => $invoiceReport,
        'payment_audit' => $paymentReport,
        'credit_audit' => $creditReport
    ];
    
    file_put_contents(
        __DIR__ . "/../storage/logs/audit_reports/monthly_{$startDate}.json",
        json_encode($report, JSON_PRETTY_PRINT)
    );
    
    // Email report to audit team
    $reportText = "Monthly Billing Audit Report\n";
    $reportText .= "Period: {$startDate} to {$endDate}\n\n";
    
    $reportText .= "INVOICE SUMMARY\n";
    $reportText .= "Total: {$invoiceReport['summary']['total_invoices']}\n";
    $reportText .= "Amount: $" . number_format($invoiceReport['summary']['total_amount'], 2) . "\n";
    $reportText .= "Deleted: " . count($invoiceReport['deleted_invoices']) . "\n";
    $reportText .= "Modified: " . count($invoiceReport['modified_invoices']) . "\n\n";
    
    $reportText .= "PAYMENT SUMMARY\n";
    $reportText .= "Total: {$paymentReport['summary']['total_payments']}\n";
    $reportText .= "Amount: $" . number_format($paymentReport['summary']['total_amount'], 2) . "\n";
    $reportText .= "Unmatched: " . count($paymentReport['unmatched_payments']) . "\n\n";
    
    $reportText .= "CREDIT SUMMARY\n";
    $reportText .= "Total Activities: {$creditReport['total_credits']}\n";
    $reportText .= "Suspicious: " . count($creditReport['suspicious_activities']) . "\n";
    
    sendAdminNotification('email', 'Monthly Billing Audit Report', $reportText);
});

function validateBillingIntegrity($invoiceReport, $paymentReport) {
    $findings = [];
    
    // Check for deleted invoices with payments
    foreach ($invoiceReport['deleted_invoices'] as $deleted) {
        // Verify if there were payments on this invoice
        $payments = Capsule::table('tblaccounts')
            ->where('invoiceid', $deleted['invoice_id'])
            ->get();
        
        if (count($payments) > 0) {
            $findings[] = [
                'severity' => 'critical',
                'category' => 'deleted_paid_invoice',
                'description' => "Invoice #{$deleted['invoice_id']} was deleted but had payments of $" . 
                    array_sum(array_column($payments, 'amount')),
                'recommendation' => 'Review deletion and consider restoration or refund investigation',
                'status' => 'open',
                'created_at' => Capsule::raw('NOW()')
            ];
        }
    }
    
    // Check for modifications after payment
    foreach ($invoiceReport['modified_invoices'] as $modified) {
        $invoice = Capsule::table('tblinvoices')
            ->where('id', $modified['invoice_id'])
            ->first();
        
        if ($invoice && $invoice->status === 'Paid' && $invoice->datepaid) {
            $modifiedDate = new \DateTime($modified['modified_at']);
            $paidDate = new \DateTime($invoice->datepaid);
            
            if ($modifiedDate > $paidDate) {
                $findings[] = [
                    'severity' => 'high',
                    'category' => 'post_payment_modification',
                    'description' => "Invoice #{$modified['invoice_id']} was modified after payment. Field: {$modified['field']}",
                    'recommendation' => 'Review post-payment changes for accuracy and compliance',
                    'status' => 'open',
                    'created_at' => Capsule::raw('NOW()')
                ];
            }
        }
    }
    
    // Check for unmatched payments
    if (count($paymentReport['unmatched_payments']) > 5) {
        $findings[] = [
            'severity' => 'medium',
            'category' => 'unmatched_payments',
            'description' => count($paymentReport['unmatched_payments']) . ' payments without transaction IDs',
            'recommendation' => 'Investigate and match payments with gateway records',
            'status' => 'open',
            'created_at' => Capsule::raw('NOW()')
        ];
    }
    
    return $findings;
}
```

### Step 6: Create Audit Dashboard

Build audit review interface:

```php
// File: /modules/addons/audit_dashboard/admin.php

function audit_dashboard_output($vars) {
    $action = $_GET['action'] ?? 'overview';
    
    echo '<div class="audit-dashboard">';
    
    switch ($action) {
        case 'findings':
            echo showAuditFindings();
            break;
        case 'reports':
            echo showAuditReports();
            break;
        case 'logs':
            echo showAuditLogs();
            break;
        default:
            echo showAuditOverview();
    }
    
    echo '</div>';
}

function showAuditOverview() {
    $recentFindings = Capsule::table('mod_audit_findings')
        ->whereIn('status', ['open', 'in_progress'])
        ->orderBy('created_at', 'desc')
        ->limit(10)
        ->get();
    
    $stats = [
        'open_findings' => Capsule::table('mod_audit_findings')
            ->where('status', 'open')
            ->count(),
        'critical_findings' => Capsule::table('mod_audit_findings')
            ->where('severity', 'critical')
            ->where('status', 'open')
            ->count(),
        'audits_this_month' => Capsule::table('mod_audit_schedules')
            ->where('last_run', '>=', date('Y-m-01'))
            ->count()
    ];
    
    $output = '<h2>Billing Audit Dashboard</h2>';
    $output .= '<div class="stats-row">';
    $output .= '<div class="stat-box">';
    $output .= '<h3>' . $stats['open_findings'] . '</h3>';
    $output .= '<p>Open Findings</p>';
    $output .= '</div>';
    $output .= '<div class="stat-box critical">';
    $output .= '<h3>' . $stats['critical_findings'] . '</h3>';
    $output .= '<p>Critical Issues</p>';
    $output .= '</div>';
    $output .= '</div>';
    
    $output .= '<h3>Recent Findings</h3>';
    $output .= '<table class="data-table">';
    $output .= '<thead><tr><th>Severity</th><th>Category</th><th>Description</th><th>Status</th><th>Date</th></tr></thead><tbody>';
    
    foreach ($recentFindings as $finding) {
        $output .= '<tr>';
        $output .= '<td><span class="severity-' . $finding->severity . '">' . strtoupper($finding->severity) . '</span></td>';
        $output .= '<td>' . $finding->category . '</td>';
        $output .= '<td>' . substr($finding->description, 0, 100) . '...</td>';
        $output .= '<td>' . $finding->status . '</td>';
        $output .= '<td>' . $finding->created_at . '</td>';
        $output .= '</tr>';
    }
    
    $output .= '</tbody></table>';
    
    return $output;
}
```

## Verification Checklist

- [ ] Audit logging configured for all billing events
- [ ] Audit logs capturing accurately
- [ ] Daily audit running on schedule
- [ ] Monthly reports generated
- [ ] Findings identified and recorded
- [ ] Critical issues alert working
- [ ] Audit dashboard accessible
- [ ] Reports email working
- [ ] Data retention configured
- [ ] Audit trail complete and accessible

## Related Skills and Documentation

- [WHMCS Billing Reporting](whmcs-billing-reporting-workflow.md)
- [WHMCS Payment Reconciliation](whmcs-payment-reconciliation-workflow.md)
- [WHMCS Credit Management](whmcs-credit-management-workflow.md)
- WHMCS Documentation: Audit Trails
- Accounting Standards: SOX Compliance

## Notes

- Maintain complete audit trail for compliance
- Review and resolve findings promptly
- Keep audit reports for required retention period
- Protect audit data from unauthorized access
- Schedule regular comprehensive audits
- Document all audit procedures
- Train staff on audit requirements
- Monitor for unauthorized access attempts
- Update audit procedures as regulations change