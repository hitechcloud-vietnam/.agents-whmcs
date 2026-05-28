# WHMCS Payment Gateway Retry Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build payment retry logic for failed transactions.

## Retry Module

```php
<?php
class PaymentRetryManager {
    private array $retrySchedule = [
        ['delay' => 3600, 'description' => '1 hour'],      // First retry: 1 hour
        ['delay' => 14400, 'description' => '4 hours'],      // Second retry: 4 hours
        ['delay' => 86400, 'description' => '1 day'],       // Third retry: 1 day
        ['delay' => 259200, 'description' => '3 days'],     // Fourth retry: 3 days
        ['delay' => 604800, 'description' => '1 week'],     // Fifth retry: 1 week
    ];

    public function scheduleRetry(int $paymentId, string $failureReason): void {
        $attempt = $this->getAttemptCount($paymentId) + 1;

        if ($attempt > count($this->retrySchedule)) {
            $this->markExhausted($paymentId);
            $this->notifyExhausted($paymentId);
            return;
        }

        $schedule = $this->retrySchedule[$attempt - 1];
        $nextRetry = date('Y-m-d H:i:s', time() + $schedule['delay']);

        Capsule::table('mod_payment_retries')->insert([
            'payment_id' => $paymentId,
            'attempt' => $attempt,
            'failure_reason' => $failureReason,
            'next_retry' => $nextRetry,
            'status' => 'scheduled',
        ]);

        logActivity("Payment retry #{$attempt} scheduled for payment {$paymentId}");
    }

    public function processRetries(): array {
        $pending = Capsule::table('mod_payment_retries')
            ->where('status', 'scheduled')
            ->where('next_retry', '<=', date('Y-m-d H:i:s'))
            ->get();

        $results = ['success' => 0, 'failed' => 0, 'exhausted' => 0];

        foreach ($pending as $retry) {
            $result = $this->attemptRetry($retry);

            if ($result['success']) {
                $results['success']++;
            } else {
                $results['failed']++;
                $this->scheduleRetry($retry->payment_id, $result['error']);
            }
        }

        return $results;
    }

    private function attemptRetry(object $retry): array {
        $payment = $this->getPayment($retry->payment_id);

        try {
            $result = localAPI('CapturePayment', [
                'invoiceid' => $payment->invoice_id,
                'gateway' => $payment->gateway,
            ]);

            if ($result['result'] === 'success') {
                $this->markSuccess($retry->id);
                return ['success' => true];
            }

            return ['success' => false, 'error' => $result['message'] ?? 'Payment failed'];
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }
}
```

---

**Related Skills:**
- whmcs-gateway-builder
- whmcs-auto-bill