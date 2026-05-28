# WHMCS Refund Processing Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build refund processing modules with approval workflows.

## Refund Module Structure

```php
<?php
/**
 * Refund Module (Addon)
 * Location: modules/addons/{module}/
 */

function {module}_config(): array {
    return [
        'name' => 'Refund Processing',
        'description' => 'Advanced refund management',
        'version' => '1.0',
    ];
}

function {module}_activate(): array {
    Capsule::schema()->create('mod_refund_requests', function($t) {
        $t->increments('id');
        $t->string('refund_number', 50)->unique();
        $t->integer('invoice_id')->unsigned();
        $t->integer('user_id')->unsigned();
        $t->string('payment_id', 100);
        $t->decimal('original_amount', 10, 2);
        $t->decimal('refund_amount', 10, 2);
        $t->string('reason', 255);
        $t->text('admin_notes')->nullable();
        $t->string('status', 20)->default('pending');
        $t->integer('requested_by')->unsigned();
        $t->integer('processed_by')->unsigned()->nullable();
        $t->timestamp('processed_at')->nullable();
        $t->timestamps();

        $t->index(['user_id', 'status']);
        $t->index('status');
    });

    Capsule::schema()->create('mod_refund_types', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('type_key', 50)->unique();
        $t->text('description')->nullable();
        $t->boolean('requires_approval')->default(true);
        $t->decimal('max_amount', 10, 2)->nullable();
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    return ['status' => 'success'];
}

function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_refund_requests');
    Capsule::schema()->dropIfExists('mod_refund_types');
    return ['status' => 'success'];
}
```

## Refund Operations

```php
<?php
class RefundManager {
    public function createRequest(int $invoiceId, float $amount, string $reason, int $requestedBy): array {
        $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();

        if (!$invoice) {
            return ['success' => false, 'error' => 'Invoice not found'];
        }

        $paidAmount = Capsule::table('tblaccounts')
            ->where('invoiceid', $invoiceId)
            ->where('amountin', '>', 0)
            ->sum('amountin');

        $refundedAmount = Capsule::table('mod_refund_requests')
            ->where('invoice_id', $invoiceId)
            ->whereIn('status', ['approved', 'processed'])
            ->sum('refund_amount');

        $availableAmount = $paidAmount - $refundedAmount;

        if ($amount > $availableAmount) {
            return ['success' => false, 'error' => 'Amount exceeds available balance'];
        }

        $refundNumber = 'RFD-' . date('Ymd') . '-' . strtoupper(substr(md5(uniqid()), 0, 6));

        $requestId = Capsule::table('mod_refund_requests')->insertGetId([
            'refund_number' => $refundNumber,
            'invoice_id' => $invoiceId,
            'user_id' => $invoice->userid,
            'payment_id' => $this->getPaymentId($invoiceId),
            'original_amount' => $paidAmount,
            'refund_amount' => $amount,
            'reason' => $reason,
            'status' => 'pending',
            'requested_by' => $requestedBy,
        ]);

        return ['success' => true, 'request_id' => $requestId, 'refund_number' => $refundNumber];
    }

    public function approveRequest(int $requestId, int $processedBy, ?string $notes = null): array {
        $request = Capsule::table('mod_refund_requests')
            ->where('id', $requestId)
            ->first();

        if (!$request || $request->status !== 'pending') {
            return ['success' => false, 'error' => 'Invalid request'];
        }

        Capsule::table('mod_refund_requests')
            ->where('id', $requestId)
            ->update([
                'status' => 'approved',
                'processed_by' => $processedBy,
                'admin_notes' => $notes,
                'processed_at' => date('Y-m-d H:i:s'),
            ]);

        return ['success' => true];
    }

    public function processRefund(int $requestId, string $method = 'original'): array {
        $request = Capsule::table('mod_refund_requests')
            ->where('id', $requestId)
            ->first();

        if (!$request || $request->status !== 'approved') {
            return ['success' => false, 'error' => 'Request not approved'];
        }

        try {
            if ($method === 'original') {
                $result = $this->refundToOriginalPayment($request);
            } elseif ($method === 'credit') {
                $result = $this->refundToCredit($request);
            } else {
                $result = $this->refundToBank($request);
            }

            if ($result['success']) {
                Capsule::table('mod_refund_requests')
                    ->where('id', $requestId)
                    ->update(['status' => 'processed']);

                $this->createCreditInvoice($request);
            }

            return $result;
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function refundToOriginalPayment(object $request): array {
        $transaction = Capsule::table('tblaccounts')
            ->where('invoiceid', $request->invoice_id)
            ->where('amountin', '>', 0)
            ->first();

        $gateway = Capsule::table('tblpaymentgateways')
            ->where('id', $transaction->gateway)
            ->first();

        $refundResult = localAPI('Refund', [
            'transactionId' => $transaction->transid,
            'amount' => $request->refund_amount,
        ]);

        return ['success' => $refundResult['result'] === 'success'];
    }

    private function refundToCredit(object $request): array {
        $creditManager = new CreditManager();
        $creditManager->addCredit(
            $request->user_id,
            $request->refund_amount,
            "Refund for Invoice #{$request->invoice_id}"
        );

        return ['success' => true];
    }
}
```

## Admin Interface

```php
function {module}_output(array $vars): void {
    $action = $_GET['action'] ?? 'list';

    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
        handleRefundAction($_POST['action'], $_POST);
    }

    echo '<div class="refund-module">';
    echo '<h1>Refund Management</h1>';

    switch ($action) {
        case 'view':
            echo renderRefundDetail($_GET['id']);
            break;
        case 'pending':
            echo renderPendingRefunds();
            break;
        default:
            echo renderRefundDashboard();
    }

    echo '</div>';
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-gateway-builder