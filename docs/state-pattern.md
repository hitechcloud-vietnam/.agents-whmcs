# State Pattern in WHMCS

The State Pattern allows an object to alter its behavior when its internal state changes. In WHMCS, this pattern is excellent for managing service lifecycle, order processing workflows, and complex business processes with multiple states.

## Overview

State pattern lets an object change its behavior without changing its class:
- Encapsulate state-specific behavior in separate classes
- Remove complex conditionals by delegating to state objects
- Make state transitions explicit
- Easy to add new states
- Simplifies context class

## Core Structure

### State Interface

```php
<?php
// includes/State/StateInterface.php

namespace CustomModule\State;

interface StateInterface
{
    public function handle(ServiceContext $context): void;
    public function getName(): string;
    public function canTransitionTo(string $stateName): bool;
}
```

### Context Class

```php
<?php
// includes/State/ServiceContext.php

namespace CustomModule\State;

class ServiceContext
{
    protected StateInterface $state;
    protected array $data = [];

    public function __construct(StateInterface $initialState)
    {
        $this->state = $initialState;
    }

    public function setState(StateInterface $state): void
    {
        $this->state = $state;
    }

    public function getState(): StateInterface
    {
        return $this->state;
    }

    public function getStateName(): string
    {
        return $this->state->getName();
    }

    public function request(): void
    {
        $this->state->handle($this);
    }

    public function setData(string $key, $value): void
    {
        $this->data[$key] = $value;
    }

    public function getData(string $key, $default = null)
    {
        return $this->data[$key] ?? $default;
    }
}
```

## Real-World WHMCS Examples

### Service Lifecycle State Machine

```php
<?php
// includes/State/ServiceStates/PendingState.php

namespace CustomModule\State\ServiceStates;

use CustomModule\State\StateInterface;
use CustomModule\State\ServiceContext;

class PendingState implements StateInterface
{
    public function handle(ServiceContext $context): void
    {
        $context->setData('entered_pending', date('Y-m-d H:i:s'));

        // Log state entry
        $this->logStateChange($context, 'pending');

        // Check if auto-activate after payment
        if ($context->getData('payment_received')) {
            $context->setState(new ActiveState());
            return;
        }

        // Send pending notification
        $this->sendPendingNotification($context);
    }

    public function getName(): string
    {
        return 'pending';
    }

    public function canTransitionTo(string $stateName): bool
    {
        return in_array($stateName, ['active', 'suspended', 'cancelled', 'fraud']);
    }

    protected function logStateChange(ServiceContext $context, string $state): void
    {
        Capsule::table('mod_service_state_log')->insert([
            'service_id' => $context->getData('service_id'),
            'state' => $state,
            'timestamp' => date('Y-m-d H:i:s'),
            'reason' => $context->getData('transition_reason') ?? 'Automatic'
        ]);
    }

    protected function sendPendingNotification(ServiceContext $context): void
    {
        // Send pending activation email
    }
}
```

```php
<?php
// includes/State/ServiceStates/ActiveState.php

namespace CustomModule\State\ServiceStates;

use CustomModule\State\StateInterface;
use CustomModule\State\ServiceContext;

class ActiveState implements StateInterface
{
    public function handle(ServiceContext $context): void
    {
        $context->setData('activated_at', date('Y-m-d H:i:s'));

        // Provision the service
        $this->provisionService($context);

        // Log state change
        $this->logStateChange($context, 'active');

        // Send welcome email
        $this->sendActivationEmail($context);
    }

    public function getName(): string
    {
        return 'active';
    }

    public function canTransitionTo(string $stateName): bool
    {
        return in_array($stateName, ['suspended', 'terminated', 'expired', 'cancelled']);
    }

    public function suspend(ServiceContext $context, string $reason): void
    {
        $context->setData('suspension_reason', $reason);
        $context->setData('suspended_at', date('Y-m-d H:i:s'));

        $this->suspendService($context);
        $this->logStateChange($context, 'suspended');

        $context->setState(new SuspendedState());
    }

    protected function provisionService(ServiceContext $context): void
    {
        // Call provisioning module
        $module = Capsule::table('tblproducts')
            ->where('id', $context->getData('product_id'))
            ->first();

        if ($module && $module->servertype) {
            // Execute provisioning
            $result = Capsule::run($module->servertype, 'CreateAccount', [
                'serviceid' => $context->getData('service_id')
            ]);

            $context->setData('provision_result', $result);
        }
    }

    protected function suspendService(ServiceContext $context): void
    {
        // Suspend on server
    }

    protected function sendActivationEmail(ServiceContext $context): void
    {
        // Send service activation email
    }

    protected function logStateChange(ServiceContext $context, string $state): void
    {
        Capsule::table('mod_service_state_log')->insert([
            'service_id' => $context->getData('service_id'),
            'state' => $state,
            'timestamp' => date('Y-m-d H:i:s')
        ]);
    }
}
```

```php
<?php
// includes/State/ServiceStates/SuspendedState.php

namespace CustomModule\State\ServiceStates;

use CustomModule\State\StateInterface;
use CustomModule\State\ServiceContext;

class SuspendedState implements StateInterface
{
    public function handle(ServiceContext $context): void
    {
        // Suspended state - waiting for payment or action
        $context->setData('is_suspended', true);

        $this->logStateChange($context, 'suspended');

        // Check if should auto-terminate
        if ($this->shouldAutoTerminate($context)) {
            $context->setState(new TerminatedState());
            return;
        }
    }

    public function getName(): string
    {
        return 'suspended';
    }

    public function canTransitionTo(string $stateName): bool
    {
        return in_array($stateName, ['active', 'terminated', 'cancelled']);
    }

    public function activate(ServiceContext $context): void
    {
        // Unsuspend and activate
        $this->unsuspendService($context);
        $this->logStateChange($context, 'active');

        $context->setState(new ActiveState());
    }

    protected function unsuspendService(ServiceContext $context): void
    {
        // Call server to unsuspend
    }

    protected function shouldAutoTerminate(ServiceContext $context): bool
    {
        $suspendedAt = $context->getData('suspended_at');
        $daysSuspended = (time() - strtotime($suspendedAt)) / 86400;

        return $daysSuspended >= 90; // Auto-terminate after 90 days
    }

    protected function logStateChange(ServiceContext $context, string $state): void
    {
        Capsule::table('mod_service_state_log')->insert([
            'service_id' => $context->getData('service_id'),
            'state' => $state,
            'timestamp' => date('Y-m-d H:i:s')
        ]);
    }
}
```

```php
<?php
// includes/State/ServiceStates/TerminatedState.php

namespace CustomModule\State\ServiceStates;

use CustomModule\State\StateInterface;
use CustomModule\State\ServiceContext;

class TerminatedState implements StateInterface
{
    public function handle(ServiceContext $context): void
    {
        $context->setData('terminated_at', date('Y-m-d H:i:s'));
        $context->setData('is_active', false);

        // Terminate on server
        $this->terminateService($context);

        // Log final state
        $this->logStateChange($context, 'terminated');

        // Send termination notification
        $this->sendTerminationEmail($context);
    }

    public function getName(): string
    {
        return 'terminated';
    }

    public function canTransitionTo(string $stateName): bool
    {
        // No transitions allowed from terminated
        return false;
    }

    protected function terminateService(ServiceContext $context): void
    {
        // Terminate service on server
    }

    protected function sendTerminationEmail(ServiceContext $context): void
    {
        // Send termination notification
    }

    protected function logStateChange(ServiceContext $context, string $state): void
    {
        Capsule::table('mod_service_state_log')->insert([
            'service_id' => $context->getData('service_id'),
            'state' => $state,
            'timestamp' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### Invoice State Machine

```php
<?php
// includes/State/InvoiceStates/DraftState.php

namespace CustomModule\State\InvoiceStates;

use CustomModule\State\StateInterface;
use CustomModule\State\InvoiceContext;

class DraftState implements StateInterface
{
    public function handle(InvoiceContext $context): void
    {
        $context->setData('is_editable', true);
        $context->setData('can_add_items', true);
        $context->setData('can_remove_items', true);
    }

    public function getName(): string
    {
        return 'draft';
    }

    public function canTransitionTo(string $stateName): bool
    {
        return in_array($stateName, ['unpaid', 'cancelled']);
    }

    public function finalize(InvoiceContext $context): void
    {
        $this->validateInvoice($context);
        $this->calculateTotals($context);

        $context->setState(new UnpaidState());
    }

    protected function validateInvoice(InvoiceContext $context): void
    {
        $items = $context->getData('items', []);

        if (empty($items)) {
            throw new \RuntimeException('Cannot finalize invoice with no items');
        }
    }

    protected function calculateTotals(InvoiceContext $context): void
    {
        $items = $context->getData('items', []);

        $subtotal = 0;
        $tax = 0;

        foreach ($items as $item) {
            $subtotal += $item['amount'] * $item['quantity'];
            if ($item['taxed'] ?? false) {
                $tax += $item['amount'] * $item['quantity'] * ($item['taxrate'] / 100);
            }
        }

        $context->setData('subtotal', $subtotal);
        $context->setData('tax', $tax);
        $context->setData('total', $subtotal + $tax);
    }
}
```

```php
<?php
// includes/State/InvoiceStates/PaidState.php

namespace CustomModule\State\InvoiceStates;

use CustomModule\State\StateInterface;
use CustomModule\State\InvoiceContext;

class PaidState implements StateInterface
{
    public function handle(InvoiceContext $context): void
    {
        $context->setData('is_editable', false);
        $context->setData('paid_at', date('Y-m-d H:i:s'));
        $context->setData('status', 'Paid');

        // Activate related services
        $this->activateServices($context);

        // Send receipt
        $this->sendReceipt($context);

        // Log payment
        $this->logPayment($context);
    }

    public function getName(): string
    {
        return 'paid';
    }

    public function canTransitionTo(string $stateName): bool
    {
        return in_array($stateName, ['refunded', 'collections']);
    }

    public function refund(InvoiceContext $context, float $amount, string $reason): void
    {
        $context->setData('refund_amount', $amount);
        $context->setData('refund_reason', $reason);
        $context->setData('refunded_at', date('Y-m-d H:i:s'));

        $context->setState(new RefundedState());
    }

    protected function activateServices(InvoiceContext $context): void
    {
        $invoiceId = $context->getData('invoice_id');

        $items = Capsule::table('tblinvoiceitems')
            ->where('invoiceid', $invoiceId)
            ->where('type', 'Hosting')
            ->get();

        foreach ($items as $item) {
            Capsule::table('tblhosting')
                ->where('id', $item->relid)
                ->update(['domainstatus' => 'Active']);
        }
    }

    protected function sendReceipt(InvoiceContext $context): void
    {
        // Send payment receipt email
    }

    protected function logPayment(InvoiceContext $context): void
    {
        Capsule::table('tblaccounts')->insert([
            'invoiceid' => $context->getData('invoice_id'),
            'userid' => $context->getData('client_id'),
            'description' => 'Payment received',
            'amount' => $context->getData('total'),
            'date' => date('Y-m-d H:i:s'),
            'paymentmethod' => $context->getData('payment_method')
        ]);
    }
}
```

### Usage Example

```php
<?php
// Service lifecycle management with state pattern

// Create initial state
$context = new ServiceContext(new PendingState());
$context->setData('service_id', 12345);
$context->setData('product_id', 100);
$context->setData('client_id', 500);

// Process current state
$context->request();

// Handle payment received
$context->setData('payment_received', true);
$context->setState(new ActiveState());
$context->request();

// Later, suspend for non-payment
$activeState = $context->getState();
if ($activeState instanceof ActiveState) {
    $activeState->suspend($context, 'Non-payment - Invoice #12345 overdue');

    // Now in suspended state
    $context->request();

    // When payment received, activate again
    $suspendedState = $context->getState();
    if ($suspendedState instanceof SuspendedState) {
        $suspendedState->activate($context);
        $context->request();
    }
}
```

### State Factory

```php
<?php
// includes/State/StateFactory.php

namespace CustomModule\State;

class StateFactory
{
    protected static array $stateClasses = [
        'service' => [
            'pending' => ServiceStates\PendingState::class,
            'active' => ServiceStates\ActiveState::class,
            'suspended' => ServiceStates\SuspendedState::class,
            'terminated' => ServiceStates\TerminatedState::class
        ],
        'invoice' => [
            'draft' => InvoiceStates\DraftState::class,
            'unpaid' => InvoiceStates\UnpaidState::class,
            'paid' => InvoiceStates\PaidState::class,
            'refunded' => InvoiceStates\RefundedState::class
        ]
    ];

    public static function create(string $entityType, string $stateName): StateInterface
    {
        if (!isset(self::$stateClasses[$entityType][$stateName])) {
            throw new \InvalidArgumentException(
                "Unknown state: {$stateName} for entity: {$entityType}"
            );
        }

        $class = self::$stateClasses[$entityType][$stateName];
        return new $class();
    }

    public static function registerState(string $entityType, string $stateName, string $class): void
    {
        self::$stateClasses[$entityType][$stateName] = $class;
    }

    public static function getAvailableStates(string $entityType): array
    {
        return array_keys(self::$stateClasses[$entityType] ?? []);
    }
}
```

## Pros

- **Localizes State Logic**: Each state has its own class
- **Eliminates Conditionals**: Removes large switch statements
- **Explicit Transitions**: State changes are explicit method calls
- **Extensibility**: Easy to add new states
- **Testability**: Test each state independently

## Cons

- **Complexity**: More classes for simple state machines
- **Overhead**: Each state transition involves multiple objects
- **State Explosion**: Many states can lead to many classes

## Best Practices

1. Keep state classes focused on one responsibility
2. Use factory for state creation
3. Implement canTransitionTo for validation
4. Log all state transitions for debugging
5. Consider using events for state changes