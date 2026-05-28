# WHMCS Subscription Billing Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing subscription and recurring billing in WHMCS.

## When to Use

- Creating subscription-based services
- Managing recurring payments
- Handling trial periods

## Subscription Patterns

### Subscription Manager

```php
class SubscriptionManager {
    public function createSubscription(int $clientId, array $params): int {
        $subscriptionId = Capsule::table('mod_{module}_subscriptions')->insertGetId([
            'client_id' => $clientId,
            'plan_id' => $params['plan_id'],
            'token' => $params['token'],
            'status' => 'active',
            'trial_end' => $params['trial_days'] ? date('Y-m-d', strtotime('+' . $params['trial_days'] . ' days')) : null,
            'next_billing' => date('Y-m-d', strtotime('+' . $params['interval_days'] . ' days')),
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        // Create first invoice
        $this->createSubscriptionInvoice($subscriptionId);

        return $subscriptionId;
    }

    public function processBilling(): void {
        $dueSubscriptions = Capsule::table('mod_{module}_subscriptions')
            ->where('status', 'active')
            ->where('next_billing', '<=', date('Y-m-d'))
            ->get();

        foreach ($dueSubscriptions as $sub) {
            $this->chargeSubscription($sub);
        }
    }

    private function chargeSubscription(object $subscription): void {
        try {
            $result = chargeToken($subscription->token, $subscription->amount * 100);

            $this->recordPayment($subscription->id, $result);

            // Update next billing
            Capsule::table('mod_{module}_subscriptions')
                ->where('id', $subscription->id)
                ->update(['next_billing' => calculateNextDate($subscription->interval_days)]);

        } catch (\Exception $e) {
            $this->handleFailedPayment($subscription, $e);
        }
    }
}
```

---

**Related Skills:**
- whmcs-tokenization-gateway
- whmcs-gateway-builder
- whmcs-cron-automation