# WHMCS Service Cancellation Workflow

## Overview
This workflow handles the complete service cancellation process including final invoice generation, data retention, and cleanup.

## Step 1: Cancellation Service

```php
<?php
// src/Service/CancellationService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class CancellationService
{
    public function requestCancellation(int $serviceId, string $reason, string $type = 'end_of_term'): array
    {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
        if (!$service) {
            return ['success' => false, 'error' => 'Service not found'];
        }

        // Create cancellation request
        $cancellationId = Capsule::table('mod_cancellations')->insertGetId([
            'service_id' => $serviceId,
            'client_id' => $service->userid,
            'reason' => $reason,
            'type' => $type,
            'status' => 'pending',
            'requested_at' => date('Y-m-d H:i:s')
        ]);

        if ($type === 'immediate') {
            return $this->processImmediateCancellation($serviceId, $cancellationId);
        }

        // Schedule cancellation for end of term
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update([
                'domainstatus' => 'Pending Cancellation',
                'cancellation_date' => $service->nextduedate
            ]);

        return [
            'success' => true,
            'cancellation_id' => $cancellationId,
            'effective_date' => $service->nextduedate
        ];
    }

    public function processImmediateCancellation(int $serviceId, int $cancellationId): array
    {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

        // Generate final invoice for unused time
        $finalInvoice = $this->generateFinalInvoice($service);

        // Terminate service
        $this->terminateService($service);

        Capsule::table('mod_cancellations')
            ->where('id', $cancellationId)
            ->update([
                'status' => 'completed',
                'completed_at' => date('Y-m-d H:i:s')
            ]);

        return [
            'success' => true,
            'final_invoice_id' => $finalInvoice
        ];
    }

    private function generateFinalInvoice($service): int
    {
        // Calculate prorated refund
        $refund = $this->calculateProratedRefund($service);

        if ($refund <= 0) {
            return 0;
        }

        $invoiceId = Capsule::table('tblinvoices')->insertGetId([
            'userid' => $service->userid,
            'invoicenum' => 'FINAL-' . time(),
            'date' => date('Y-m-d'),
            'duedate' => date('Y-m-d'),
            'status' => 'Unpaid',
            'total' => -$refund,
            'subtotal' => -$refund
        ]);

        Capsule::table('tblinvoiceitems')->insert([
            'invoiceid' => $invoiceId,
            'userid' => $service->userid,
            'type' => 'Prorated Refund',
            'description' => 'Unused time refund',
            'amount' => -$refund
        ]);

        return $invoiceId;
    }

    private function calculateProratedRefund($service): float
    {
        $daysRemaining = $this->getDaysRemaining($service->nextduedate);
        $daysInCycle = $this->getDaysInBillingCycle($service->billingcycle);

        $pricing = Capsule::table('tblpricing')
            ->where('type', 'product')
            ->where('relid', $service->packageid)
            ->first();

        $cycle = strtolower($service->billingcycle);
        $price = (float)($pricing->{$cycle} ?? $pricing->monthly ?? 0);

        return ($price / $daysInCycle) * $daysRemaining;
    }

    private function terminateService(int $serviceId): void
    {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

        if ($service->serverid) {
            $server = Capsule::table('tblservers')->where('id', $service->serverid)->first();
            $module = new \WHMCS\Module\Server($server->type);
            $module->setServerCredential('hostname', $server->hostname);
            $module->setServerCredential('username', $server->username);
            $module->setServerCredential('password', decrypt($server->password));

            try {
                $module->call('TerminateAccount', ['serviceid' => $serviceId]);
            } catch (\Exception $e) {
                logActivity("Service termination failed: " . $e->getMessage());
            }
        }

        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update(['domainstatus' => 'Terminated']);
    }

    private function getDaysRemaining(string $dueDate): int
    {
        $due = new \DateTime($dueDate);
        $today = new \DateTime();
        return max(0, $today->diff($due)->days);
    }

    private function getDaysInBillingCycle(string $cycle): int
    {
        return match (strtolower($cycle)) {
            'monthly' => 30,
            'quarterly' => 90,
            'semiannually' => 180,
            'annually' => 365,
            default => 30
        };
    }
}
```

## Verification Checklist

- [ ] Cancellation service implemented
- [ ] End-of-term cancellation working
- [ ] Immediate cancellation working
- [ ] Final invoice generation correct
- [ ] Service termination working
- [ ] Refund calculation accurate
