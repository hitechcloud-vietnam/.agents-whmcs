# WHMCS Subscription Management Workflow

## Overview
This workflow covers subscription lifecycle management including upgrades, downgrades, pauses, and cancellations.

## Step 1: Subscription Management Service

```php
<?php
// src/Service/SubscriptionService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class SubscriptionService
{
    public function upgradeSubscription(int $serviceId, int $newProductId): array
    {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
        if (!$service) {
            return ['success' => false, 'error' => 'Service not found'];
        }

        $currentProduct = Capsule::table('tblproducts')->where('id', $service->packageid)->first();
        $newProduct = Capsule::table('tblproducts')->where('id', $newProductId)->first();

        if (!$newProduct) {
            return ['success' => false, 'error' => 'New product not found'];
        }

        // Calculate prorated difference
        $proration = $this->calculateUpgradeProration($service, $currentProduct, $newProduct);

        // Create proration invoice
        $invoiceId = $this->createProrationInvoice($service, $proration);

        // Update service
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update([
                'packageid' => $newProductId,
                'last_updated' => date('Y-m-d H:i:s')
            ]);

        // Update server if needed
        $this->processServerUpgrade($service, $newProduct);

        return [
            'success' => true,
            'invoice_id' => $invoiceId,
            'proration_amount' => $proration['amount'],
            'new_product' => $newProduct->name
        ];
    }

    public function downgradeSubscription(int $serviceId, int $newProductId): array
    {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

        // Calculate credit for unused time
        $credit = $this->calculateDowngradeCredit($service, $newProductId);

        // Update service
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update([
                'packageid' => $newProductId,
                'last_updated' => date('Y-m-d H:i:s')
            ]);

        // Apply credit to account
        $this->applyAccountCredit($service->userid, $credit);

        return [
            'success' => true,
            'credit_amount' => $credit,
            'new_product_id' => $newProductId
        ];
    }

    public function pauseSubscription(int $serviceId, int $pauseDays): array
    {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
        $resumeDate = date('Y-m-d', strtotime("+{$pauseDays} days"));

        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update([
                'domainstatus' => 'Paused',
                'pause_date' => date('Y-m-d'),
                'resume_date' => $resumeDate
            ]);

        // Log pause event
        Capsule::table('mod_subscription_events')->insert([
            'service_id' => $serviceId,
            'event_type' => 'pause',
            'pause_days' => $pauseDays,
            'resume_date' => $resumeDate,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return [
            'success' => true,
            'resume_date' => $resumeDate
        ];
    }

    public function resumeSubscription(int $serviceId): array
    {
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update([
                'domainstatus' => 'Active',
                'pause_date' => null,
                'resume_date' => null
            ]);

        return ['success' => true];
    }

    public function cancelSubscription(int $serviceId, string $reason, bool $terminateImmediately = false): array
    {
        $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

        $endDate = $terminateImmediately
            ? date('Y-m-d')
            : $service->nextduedate;

        // Update cancellation status
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update([
                'domainstatus' => 'Cancelled',
                'termination_date' => $endDate,
                'cancellation_reason' => $reason
            ]);

        // Log cancellation
        Capsule::table('mod_subscription_events')->insert([
            'service_id' => $serviceId,
            'event_type' => 'cancel',
            'reason' => $reason,
            'end_date' => $endDate,
            'immediate' => $terminateImmediately,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        // Send cancellation confirmation
        $client = Capsule::table('tblclients')->where('id', $service->userid)->first();
        send_email('Subscription Cancellation', $client->email, [
            'service_id' => $serviceId,
            'end_date' => $endDate,
            'reason' => $reason
        ]);

        if ($terminateImmediately) {
            $this->terminateService($serviceId);
        }

        return [
            'success' => true,
            'end_date' => $endDate
        ];
    }

    private function calculateUpgradeProration($service, $currentProduct, $newProduct): array
    {
        $currentPrice = $this->getProductPrice($currentProduct, $service->billingcycle);
        $newPrice = $this->getProductPrice($newProduct, $service->billingcycle);

        $daysRemaining = $this->getDaysRemainingInCycle($service->nextduedate);
        $daysInCycle = $this->getDaysInCycle($service->billingcycle);

        $unusedDays = $currentPrice * ($daysRemaining / $daysInCycle);
        $newDays = $newPrice * ($daysRemaining / $daysInCycle);

        return [
            'current_daily_rate' => $currentPrice / $daysInCycle,
            'new_daily_rate' => $newPrice / $daysInCycle,
            'days_remaining' => $daysRemaining,
            'credit' => round($unusedDays, 2),
            'charge' => round($newDays, 2),
            'amount' => round($newDays - $unusedDays, 2)
        ];
    }

    private function calculateDowngradeCredit($service, int $newProductId): float
    {
        $currentProduct = Capsule::table('tblproducts')->where('id', $service->packageid)->first();
        $newProduct = Capsule::table('tblproducts')->where('id', $newProductId)->first();

        $currentPrice = $this->getProductPrice($currentProduct, $service->billingcycle);
        $newPrice = $this->getProductPrice($newProduct, $service->billingcycle);

        $daysRemaining = $this->getDaysRemainingInCycle($service->nextduedate);
        $daysInCycle = $this->getDaysInCycle($service->billingcycle);

        $dailyDiff = ($currentPrice - $newPrice) / $daysInCycle;
        return round($dailyDiff * $daysRemaining, 2);
    }

    private function getProductPrice($product, string $billingCycle): float
    {
        $pricing = Capsule::table('tblpricing')
            ->where('type', 'product')
            ->where('relid', $product->id)
            ->first();

        $cycle = strtolower($billingCycle);
        return (float)($pricing->{$cycle} ?? $pricing->monthly ?? 0);
    }

    private function getDaysRemainingInCycle(string $dueDate): int
    {
        $due = new \DateTime($dueDate);
        $today = new \DateTime();
        return max(0, $due->diff($today)->days);
    }

    private function getDaysInCycle(string $billingCycle): int
    {
        switch (strtolower($billingCycle)) {
            case 'monthly': return 30;
            case 'quarterly': return 90;
            case 'semiannually': return 180;
            case 'annually': return 365;
            default: return 30;
        }
    }

    private function createProrationInvoice($service, array $proration): int
    {
        if ($proration['amount'] <= 0) {
            return 0;
        }

        $invoiceId = Capsule::table('tblinvoices')->insertGetId([
            'userid' => $service->userid,
            'invoicenum' => 'PR-' . time(),
            'date' => date('Y-m-d'),
            'duedate' => date('Y-m-d'),
            'status' => 'Unpaid',
            'subtotal' => $proration['amount'],
            'tax' => 0,
            'total' => $proration['amount'],
            'paymentmethod' => $service->paymentmethod
        ]);

        Capsule::table('tblinvoiceitems')->insert([
            'invoiceid' => $invoiceId,
            'userid' => $service->userid,
            'type' => 'Upgrade Proration',
            'relid' => $service->id,
            'description' => 'Prorated upgrade charge',
            'amount' => $proration['amount'],
            'taxed' => 1
        ]);

        return $invoiceId;
    }

    private function applyAccountCredit(int $clientId, float $amount): void
    {
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->increment('credit', $amount);

        Capsule::table('mod_subscription_credit_log')->insert([
            'client_id' => $clientId,
            'amount' => $amount,
            'type' => 'downgrade_credit',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    private function processServerUpgrade($service, $newProduct): void
    {
        if ($service->serverid && $newProduct->servergroup !== null) {
            $newServerId = $this->selectServer($newProduct);
            if ($newServerId !== $service->serverid) {
                Capsule::table('tblhosting')
                    ->where('id', $service->id)
                    ->update(['serverid' => $newServerId]);
            }
        }
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
            $module->call('TerminateAccount', ['serviceid' => $serviceId]);
        }

        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update(['domainstatus' => 'Terminated']);
    }
}
```

## Step 2: Subscription Change Hooks

```php
<?php
// includes/hooks/subscription_hook.php

add_hook('SubscriptionUpgrade', 1, function($params) {
    logActivity("Subscription upgraded: Service {$params['service_id']} to product {$params['new_product_id']}");
    return $params;
});

add_hook('SubscriptionDowngrade', 1, function($params) {
    $creditAmount = $params['credit_amount'] ?? 0;
    logActivity("Subscription downgraded: Service {$params['service_id']}. Credit applied: {$creditAmount}");
    return $params;
});

add_hook('SubscriptionPaused', 1, function($params) {
    $client = Capsule::table('tblclients')->where('id', $params['user_id'])->first();
    send_email('Subscription Paused', $client->email, [
        'service_id' => $params['service_id'],
        'resume_date' => $params['resume_date']
    ]);
    return $params;
});

add_hook('SubscriptionCancelled', 1, function($params) {
    logActivity("Subscription cancelled: Service {$params['service_id']}. End date: {$params['end_date']}");
    return $params;
});
```

## Verification Checklist

- [ ] Subscription service implemented
- [ ] Upgrade logic working
- [ ] Downgrade logic working
- [ ] Proration calculation accurate
- [ ] Pause/resume functionality working
- [ ] Cancellation workflow complete
- [ ] Webhooks firing correctly
- [ ] Credit handling correct
- [ ] Test upgrade completed
- [ ] Test downgrade completed
