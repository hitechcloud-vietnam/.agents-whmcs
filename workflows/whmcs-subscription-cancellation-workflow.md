# WHMCS Subscription Cancellation Workflow

## Purpose

Handle subscription and service cancellations, process refund calculations for unused time, maintain proper audit trails, and manage the post-cancellation experience.

## Prerequisites

- WHMCS with product/service management
- Cancellation policy documented
- Refund calculation rules defined
- Proper authorization levels for cancellations

## Workflow Steps

### Step 1: Configure Cancellation Settings

Set up cancellation configuration:

```php
// Database configuration for cancellation
INSERT INTO tblconfiguration (setting, value) VALUES 
('CancellationEnabled', 'on'),
('CancellationRequestMethod', 'both'), // client, admin, both
('CancellationNoticeDays', '30'),
('RefundUnusedTime', 'on'),
('CancellationFee', '0.00'),
('ProrateRefunds', 'on');

// Create cancellation tracking tables
Capsule::schema()->create('mod_cancellation_requests', function($t) {
    $t->increments('id');
    $t->integer('hosting_id');
    $t->integer('client_id');
    $t->string('reason');
    $t->text('feedback')->nullable();
    $t->date('requested_date');
    $t->date('cancellation_date');
    $t->decimal('refund_amount', 10, 2)->nullable();
    $t->decimal('cancellation_fee', 10, 2)->default(0);
    $t->string('status'); // pending, approved, processing, completed, rejected
    $t->integer('processed_by');
    $t->text('admin_notes')->nullable();
    $t->timestamp('created_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
    $t->timestamp('processed_at')->nullable();
});

Capsule::schema()->create('mod_cancellation_audit', function($t) {
    $t->increments('id');
    $t->integer('cancellation_id');
    $t->string('action');
    $t->integer('performed_by');
    $t->text('details');
    $t->timestamp('created_at')->default(Capsule::raw('CURRENT_TIMESTAMP'));
});
```

### Step 2: Create Cancellation Request Handler

Build cancellation functionality:

```php
// File: /includes/classes/CancellationManager.php

namespace WHMCS\Service;

class CancellationManager {
    
    public function createCancellationRequest($hostingId, $reason, $feedback = '', $immediate = false) {
        $hosting = Capsule::table('tblhosting')
            ->where('id', $hostingId)
            ->first();
        
        // Check for existing pending request
        $existing = Capsule::table('mod_cancellation_requests')
            ->where('hosting_id', $hostingId)
            ->whereIn('status', ['pending', 'approved', 'processing'])
            ->first();
        
        if ($existing) {
            throw new Exception('A cancellation request already exists for this service');
        }
        
        // Calculate cancellation date
        $noticeDays = (int) Capsule::table('tblconfiguration')
            ->where('setting', 'CancellationNoticeDays')
            ->value('value') ?? 30;
        
        $cancellationDate = $immediate 
            ? date('Y-m-d') 
            : date('Y-m-d', strtotime('+' . $noticeDays . ' days'));
        
        // Calculate refund if applicable
        $refundAmount = null;
        if ($this->shouldRefundUnusedTime()) {
            $refundAmount = $this->calculateRefund($hosting);
        }
        
        $requestId = Capsule::table('mod_cancellation_requests')->insertGetId([
            'hosting_id' => $hostingId,
            'client_id' => $hosting->userid,
            'reason' => $reason,
            'feedback' => $feedback,
            'requested_date' => date('Y-m-d'),
            'cancellation_date' => $cancellationDate,
            'refund_amount' => $refundAmount,
            'status' => 'pending',
            'processed_by' => $_SESSION['adminid'] ?? 0
        ]);
        
        // Log the request
        $this->logAudit($requestId, 'request_created', [
            'reason' => $reason,
            'cancellation_date' => $cancellationDate,
            'refund_amount' => $refundAmount
        ]);
        
        // Send confirmation email
        sendTemplatedEmail('CancellationRequestReceived', $hosting->userid, [
            'service_name' => $this->getServiceName($hosting),
            'cancellation_date' => $cancellationDate,
            'refund_amount' => $refundAmount
        ]);
        
        return $requestId;
    }
    
    private function shouldRefundUnusedTime() {
        $setting = Capsule::table('tblconfiguration')
            ->where('setting', 'RefundUnusedTime')
            ->value('value');
        
        return $setting === 'on';
    }
    
    private function calculateRefund($hosting) {
        $invoice = Capsule::table('tblinvoices')
            ->join('tblinvoiceitems', 'tblinvoices.id', '=', 'tblinvoiceitems.invoiceid')
            ->where('tblinvoiceitems.relid', $hosting->id)
            ->where('tblinvoices.status', 'Paid')
            ->orderBy('tblinvoices.datepaid', 'desc')
            ->first();
        
        if (!$invoice) {
            return 0;
        }
        
        $billingDays = $this->getBillingCycleDays($hosting->billingcycle);
        $cycleStart = new \DateTime($invoice->datepaid);
        $cancellationDate = new \DateTime();
        
        $daysUsed = $cycleStart->diff($cancellationDate)->days;
        $daysRemaining = max(0, $billingDays - $daysUsed);
        
        $dailyRate = $invoice->amount / $billingDays;
        $refundAmount = $dailyRate * $daysRemaining;
        
        // Apply cancellation fee if any
        $cancellationFee = (float) Capsule::table('tblconfiguration')
            ->where('setting', 'CancellationFee')
            ->value('value') ?? 0;
        
        $refundAmount = max(0, $refundAmount - $cancellationFee);
        
        return round($refundAmount, 2);
    }
    
    private function getBillingCycleDays($cycle) {
        $cycles = [
            'Monthly' => 30,
            'Quarterly' => 90,
            'SemiAnnually' => 180,
            'Annually' => 365
        ];
        return $cycles[$cycle] ?? 30;
    }
    
    private function getServiceName($hosting) {
        $product = Capsule::table('tblproducts')
            ->where('id', $hosting->packageid)
            ->first();
        
        return $product->name ?? 'Service';
    }
    
    public function approveCancellation($requestId, $adminId = null) {
        $request = Capsule::table('mod_cancellation_requests')
            ->where('id', $requestId)
            ->first();
        
        if ($request->status !== 'pending') {
            throw new Exception('Cancellation request is not pending');
        }
        
        Capsule::table('mod_cancellation_requests')
            ->where('id', $requestId)
            ->update([
                'status' => 'approved',
                'processed_by' => $adminId ?? $_SESSION['adminid'] ?? 0,
                'processed_at' => Capsule::raw('NOW()')
            ]);
        
        $this->logAudit($requestId, 'request_approved', [
            'approved_by' => $adminId ?? $_SESSION['adminid'] ?? 0
        ]);
        
        // Schedule for processing
        $this->scheduleCancellation($request);
    }
    
    private function scheduleCancellation($request) {
        // Schedule the actual cancellation for the requested date
        // This could be handled by the cron job check
        Capsule::table('mod_cancellation_scheduled')->insert([
            'cancellation_id' => $request->id,
            'hosting_id' => $request->hosting_id,
            'scheduled_date' => $request->cancellation_date,
            'status' => 'scheduled'
        ]);
        
        sendTemplatedEmail('CancellationApproved', $request->client_id, [
            'cancellation_date' => $request->cancellation_date,
            'refund_amount' => $request->refund_amount
        ]);
    }
}
```

### Step 3: Create Cancellation Processing

Handle the actual cancellation:

```php
// File: /includes/hooks/cancellation_processing.php

add_hook('DailyCronJob', 1, function($vars) {
    $today = date('Y-m-d');
    
    // Find scheduled cancellations due today
    $scheduled = Capsule::table('mod_cancellation_scheduled')
        ->where('scheduled_date', '<=', $today)
        ->where('status', 'scheduled')
        ->get();
    
    foreach ($scheduled as $cancellation) {
        $request = Capsule::table('mod_cancellation_requests')
            ->where('id', $cancellation->cancellation_id)
            ->first();
        
        processCancellation($request);
        
        Capsule::table('mod_cancellation_scheduled')
            ->where('id', $cancellation->id)
            ->update(['status' => 'completed']);
    }
});

function processCancellation($request) {
    $hosting = Capsule::table('tblhosting')
        ->where('id', $request->hosting_id)
        ->first();
    
    // Update request status
    Capsule::table('mod_cancellation_requests')
        ->where('id', $request->id)
        ->update([
            'status' => 'processing',
            'processed_at' => Capsule::raw('NOW()')
        ]);
    
    // Process refund if applicable
    if ($request->refund_amount > 0) {
        processRefund($request);
    }
    
    // Terminate service with provider
    $product = Capsule::table('tblproducts')
        ->where('id', $hosting->packageid)
        ->first();
    
    if ($product->servertype) {
        $params = [
            'accountid' => $hosting->id,
            'username' => $hosting->username ?? '',
            'server' => getServerParams($hosting->server),
            'serverid' => $hosting->server
        ];
        
        $result = Capsule::moduleFactory($product->servertype)
            ->call('TerminateAccount', $params);
        
        logActivity("Service termination via module: {$result}");
    }
    
    // Update service status
    Capsule::table('tblhosting')
        ->where('id', $request->hosting_id)
        ->update([
            'domainstatus' => 'Terminated',
            'termination_date' => date('Y-m-d H:i:s')
        ]);
    
    // Complete cancellation request
    Capsule::table('mod_cancellation_requests')
        ->where('id', $request->id)
        ->update(['status' => 'completed']);
    
    // Log audit
    logActivity("Service #{$request->hosting_id} cancelled. Refund: {$request->refund_amount}");
    
    // Send confirmation email
    sendTemplatedEmail('CancellationCompleted', $request->client_id, [
        'service_name' => $product->name ?? 'Service',
        'cancellation_date' => date('Y-m-d'),
        'refund_amount' => $request->refund_amount
    ]);
    
    // If refund was processed, notify accounting
    if ($request->refund_amount > 0) {
        sendTemplatedEmail('CancellationRefundProcessed', 0, [
            'client_id' => $request->client_id,
            'amount' => $request->refund_amount,
            'invoice_id' => getRelatedInvoiceId($request->hosting_id)
        ]);
    }
}

function processRefund($request) {
    // Get original payment transaction
    $originalPayment = Capsule::table('tblaccounts')
        ->where('invoiceid', getRelatedInvoiceId($request->hosting_id))
        ->where('transid', '!=', '')
        ->orderBy('id', 'desc')
        ->first();
    
    if ($originalPayment) {
        // Process refund through gateway
        $refundResult = processGatewayRefund(
            $originalPayment->transid,
            $request->refund_amount,
            $originalPayment->gateway
        );
        
        if ($refundResult['success']) {
            // Record refund transaction
            Capsule::table('tblaccounts')->insert([
                'userid' => $request->client_id,
                'description' => 'Cancellation refund',
                'amount' => -$request->refund_amount,
                'date' => date('Y-m-d H:i:s'),
                'gateway' => $originalPayment->gateway
            ]);
            
            // Update client credit or apply to balance
            $client = Capsule::table('tblclients')
                ->where('id', $request->client_id)
                ->first();
            
            // Send refund to original payment method
            // ... (gateway-specific implementation)
            
            logActivity("Refund of {$request->refund_amount} processed for cancellation #{$request->id}");
        }
    }
}
```

### Step 4: Create Client-Facing Cancellation

Add client area cancellation functionality:

```php
// File: /includes/hooks/client_cancellation.php

add_hook('ClientAreaPage', 1, function($vars) {
    // Add cancellation option to service details
    if ($vars['filename'] === 'clientarea' && isset($_GET['action']) && $_GET['action'] === 'details') {
        $serviceId = (int)($_GET['id'] ?? 0);
        
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->where('userid', $_SESSION['uid'])
            ->first();
        
        if ($service && $service->domainstatus === 'Active') {
            return [
                'show_cancellation' => true,
                'cancellation_url' => 'cancel.php?id=' . $serviceId
            ];
        }
    }
});

// Handle cancellation form submission
add_hook('ClientAreaRequest', 1, function($vars) {
    if ($_GET['action'] === 'request_cancellation') {
        check_token();
        
        $serviceId = (int)($_POST['service_id'] ?? 0);
        $reason = sanitize($_POST['reason'] ?? '');
        $feedback = sanitize($_POST['feedback'] ?? '');
        
        // Verify ownership
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->where('userid', $_SESSION['uid'])
            ->first();
        
        if (!$service) {
            return ['error' => 'Service not found'];
        }
        
        try {
            $manager = new \WHMCS\Service\CancellationManager();
            $requestId = $manager->createCancellationRequest(
                $serviceId,
                $reason,
                $feedback,
                isset($_POST['immediate'])
            );
            
            return [
                'success' => true,
                'message' => 'Cancellation request submitted successfully',
                'request_id' => $requestId
            ];
        } catch (Exception $e) {
            return ['error' => $e->getMessage()];
        }
    }
});
```

### Step 5: Create Cancellation Analytics

Track and report on cancellations:

```php
// File: /includes/hooks/cancellation_reporting.php

function getCancellationMetrics($startDate, $endDate) {
    $requests = Capsule::table('mod_cancellation_requests')
        ->whereBetween('requested_date', [$startDate, $endDate])
        ->get();
    
    $metrics = [
        'total_requests' => count($requests),
        'by_status' => ['pending' => 0, 'approved' => 0, 'completed' => 0, 'rejected' => 0],
        'by_reason' => [],
        'total_refunds' => 0,
        'average_refund' => 0
    ];
    
    foreach ($requests as $request) {
        $metrics['by_status'][$request->status]++;
        $metrics['by_reason'][$request->reason] = ($metrics['by_reason'][$request->reason] ?? 0) + 1;
        $metrics['total_refunds'] += $request->refund_amount ?? 0;
    }
    
    $completed = array_filter($requests, function($r) {
        return $r->status === 'completed';
    });
    
    $metrics['average_refund'] = count($completed) > 0 
        ? round($metrics['total_refunds'] / count($completed), 2) 
        : 0;
    
    return $metrics;
}

add_hook('MonthlyCronJob', 1, function($vars) {
    $startDate = date('Y-m-01', strtotime('-1 month'));
    $endDate = date('Y-m-t', strtotime('-1 month'));
    
    $metrics = getCancellationMetrics($startDate, $endDate);
    
    $report = "Cancellation Monthly Report\n";
    $report .= "Period: {$startDate} to {$endDate}\n\n";
    $report .= "Total Requests: {$metrics['total_requests']}\n";
    $report .= "Completed: {$metrics['by_status']['completed']}\n";
    $report .= "Pending: {$metrics['by_status']['pending']}\n";
    $report .= "Rejected: {$metrics['by_status']['rejected']}\n\n";
    $report .= "Total Refunds: $" . number_format($metrics['total_refunds'], 2) . "\n";
    $report .= "Average Refund: $" . number_format($metrics['average_refund'], 2) . "\n\n";
    $report .= "By Reason:\n";
    foreach ($metrics['by_reason'] as $reason => $count) {
        $report .= "- {$reason}: {$count}\n";
    }
    
    sendAdminNotification('email', 'Cancellation Report', $report);
});
```

## Verification Checklist

- [ ] Cancellation request form working
- [ ] Request approval workflow functioning
- [ ] Scheduled cancellations processing
- [ ] Service termination working
- [ ] Refund calculation accurate
- [ ] Refund processing functional
- [ ] Client notification emails sent
- [ ] Admin notification emails sent
- [ ] Audit trail complete
- [ ] Reports generating correctly

## Related Skills and Documentation

- [WHMCS Subscription Workflow](whmcs-subscription-workflow.md)
- [WHMCS Refund Workflow](whmcs-refund-workflow.md)
- [WHMCS Prorating](whmcs-prorating-workflow.md)
- WHMCS Documentation: Cancellation Requests
- WHMCS Documentation: Service Termination

## Notes

- Clearly communicate cancellation policies to clients
- Make the process transparent and easy to understand
- Provide clear refund calculations before confirmation
- Keep detailed audit trail for compliance
- Monitor cancellation reasons for service improvements
- Review cancellation rates monthly
- Consider exit surveys for insights
- Balance cancellation ease with retention efforts