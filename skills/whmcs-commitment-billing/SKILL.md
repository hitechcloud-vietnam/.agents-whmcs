# WHMCS Commitment Billing Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Implement committed billing plans with usage commitments and true-up billing.

## Database Schema

```php
<?php
// modules/addons/commitment_billing/commitment_billing.php

use WHMCS\Database\Capsule;

function commitment_billing_config(): array {
    return [
        'name' => 'Commitment Billing',
        'description' => 'Committed usage plans with true-up billing',
        'version' => '1.0',
    ];
}

function commitment_billing_activate(): array {
    Capsule::schema()->create('mod_commitment_plans', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('tier', 50)->nullable();
        $t->text('description')->nullable();
        $t->decimal('monthly_commitment', 10, 2);
        $t->decimal('annual_commitment', 10, 2)->nullable();
        $t->string('billing_cycle', 20)->default('monthly');
        $t->decimal('discount_percentage', 5, 2)->default(0);
        $t->string('commitment_type', 30)->default('spend');
        $t->integer('min_term_months')->default(1);
        $t->boolean('auto_renew')->default(true);
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_commitment_included', function($t) {
        $t->increments('id');
        $t->integer('plan_id')->unsigned();
        $t->string('resource', 100);
        $t->decimal('included_units', 12, 2)->default(0);
        $t->string('unit', 30)->nullable();
    });

    Capsule::schema()->create('mod_commitment_contracts', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->integer('plan_id')->unsigned();
        $t->string('contract_id', 50)->unique();
        $t->decimal('committed_amount', 10, 2);
        $t->decimal('actual_spend', 10, 2)->default(0);
        $t->decimal('true_up_amount', 10, 2)->default(0);
        $t->date('start_date');
        $t->date('end_date');
        $t->string('status', 20)->default('active');
        $t->text('notes')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_commitment_credits', function($t) {
        $t->increments('id');
        $t->integer('contract_id')->unsigned();
        $t->decimal('credit_amount', 10, 2);
        $t->string('reason', 255);
        $t->timestamp('credited_at')->useCurrent();
    });

    Capsule::schema()->create('mod_commitment_spend', function($t) {
        $t->increments('id');
        $t->integer('contract_id')->unsigned();
        $t->string('resource', 100);
        $t->string('period', 20);
        $t->decimal('amount', 10, 2);
        $t->timestamp('recorded_at')->useCurrent();
    });

    Capsule::schema()->create('mod_commitment_trueup', function($t) {
        $t->increments('id');
        $t->integer('contract_id')->unsigned();
        $t->string('period', 20);
        $t->decimal('committed_amount', 10, 2);
        $t->decimal('actual_spend', 10, 2);
        $t->decimal('true_up_charge', 10, 2)->default(0);
        $t->decimal('carry_forward', 10, 2)->default(0);
        $t->string('status', 20)->default('pending');
        $t->integer('invoice_id')->unsigned()->nullable();
        $t->timestamp('processed_at')->nullable();
    });

    return ['status' => 'success'];
}

function commitment_billing_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_commitment_trueup');
    Capsule::schema()->dropIfExists('mod_commitment_spend');
    Capsule::schema()->dropIfExists('mod_commitment_credits');
    Capsule::schema()->dropIfExists('mod_commitment_contracts');
    Capsule::schema()->dropIfExists('mod_commitment_included');
    Capsule::schema()->dropIfExists('mod_commitment_plans');
    return ['status' => 'success'];
}
```

## Commitment Billing Manager

```php
<?php
class CommitmentBillingManager {
    public function createContract(int $userId, int $planId, ?DateTime $startDate = null): array {
        $plan = Capsule::table('mod_commitment_plans')->find($planId);

        if (!$plan || !$plan->is_active) {
            return ['success' => false, 'error' => 'Invalid plan'];
        }

        $start = $startDate ?? new DateTime();
        $end = clone $start;
        $end->modify("+{$plan->min_term_months} months");

        $contractId = 'CN-' . strtoupper(substr(md5($userId . $planId . time()), 0, 8));

        $contractId = Capsule::table('mod_commitment_contracts')->insertGetId([
            'user_id' => $userId,
            'plan_id' => $planId,
            'contract_id' => $contractId,
            'committed_amount' => $plan->monthly_commitment,
            'start_date' => $start->format('Y-m-d'),
            'end_date' => $end->format('Y-m-d'),
            'status' => 'active',
        ]);

        // Create first billing period
        $this->initializeBillingPeriod($contractId);

        return [
            'success' => true,
            'contract_id' => $contractId,
            'contract_ref' => $contractId,
            'start_date' => $start->format('Y-m-d'),
            'end_date' => $end->format('Y-m-d'),
            'monthly_commitment' => $plan->monthly_commitment,
        ];
    }

    private function initializeBillingPeriod(int $contractId): void {
        $contract = Capsule::table('mod_commitment_contracts')->find($contractId);

        Capsule::table('mod_commitment_trueup')->insert([
            'contract_id' => $contractId,
            'period' => date('Y-m'),
            'committed_amount' => $contract->committed_amount,
            'actual_spend' => 0,
            'status' => 'active',
        ]);
    }

    public function recordSpend(int $contractId, string $resource, float $amount): void {
        $contract = Capsule::table('mod_commitment_contracts')->find($contractId);

        if (!$contract || $contract->status !== 'active') {
            return;
        }

        $period = date('Y-m');

        // Check for existing spend record
        $existing = Capsule::table('mod_commitment_spend')
            ->where('contract_id', $contractId)
            ->where('resource', $resource)
            ->where('period', $period)
            ->first();

        if ($existing) {
            Capsule::table('mod_commitment_spend')
                ->where('id', $existing->id)
                ->update(['amount' => $existing->amount + $amount]);
        } else {
            Capsule::table('mod_commitment_spend')->insert([
                'contract_id' => $contractId,
                'resource' => $resource,
                'period' => $period,
                'amount' => $amount,
            ]);
        }

        // Update contract actual spend
        Capsule::table('mod_commitment_contracts')
            ->where('id', $contractId)
            ->update(['actual_spend' => Capsule::raw('actual_spend + ' . $amount)]);

        // Update current period trueup
        $this->updateTrueup($contractId, $period);
    }

    private function updateTrueup(int $contractId, string $period): void {
        $contract = Capsule::table('mod_commitment_contracts')->find($contractId);

        $periodSpend = Capsule::table('mod_commitment_spend')
            ->where('contract_id', $contractId)
            ->where('period', $period)
            ->sum('amount');

        $credits = Capsule::table('mod_commitment_credits')
            ->where('contract_id', $contractId)
            ->sum('credit_amount');

        $trueUpRecord = Capsule::table('mod_commitment_trueup')
            ->where('contract_id', $contractId)
            ->where('period', $period)
            ->first();

        if ($trueUpRecord) {
            $actualSpend = $periodSpend + $credits;
            $committedAmount = $trueUpRecord->committed_amount;

            // Check for carry forward from previous period
            $prevPeriod = date('Y-m', strtotime($period . ' -1 month'));
            $prevTrueup = Capsule::table('mod_commitment_trueup')
                ->where('contract_id', $contractId)
                ->where('period', $prevPeriod)
                ->first();

            $carryForward = $prevTrueup ? (float)$prevTrueup->carry_forward : 0;
            $adjustedCommitment = $committedAmount - $carryForward;

            $trueUpAmount = max(0, $adjustedCommitment - $actualSpend);
            $newCarryForward = max(0, $actualSpend - $adjustedCommitment);

            Capsule::table('mod_commitment_trueup')
                ->where('id', $trueUpRecord->id)
                ->update([
                    'actual_spend' => $actualSpend,
                    'true_up_charge' => 0, // No charge, just carry forward
                    'carry_forward' => $newCarryForward,
                ]);
        }
    }

    public function processMonthlyTrueUp(int $contractId): array {
        $contract = Capsule::table('mod_commitment_contracts')->find($contractId);

        if (!$contract || $contract->status !== 'active') {
            return ['success' => false, 'error' => 'Invalid contract'];
        }

        $currentPeriod = date('Y-m');
        $periodEnd = date('Y-m-t');

        if (date('d') < (int)date('t')) {
            return ['success' => false, 'error' => 'Period not complete'];
        }

        $trueUpRecord = Capsule::table('mod_commitment_trueup')
            ->where('contract_id', $contractId)
            ->where('period', $currentPeriod)
            ->first();

        if (!$trueUpRecord) {
            return ['success' => false, 'error' => 'No trueup record found'];
        }

        $plan = Capsule::table('mod_commitment_plans')->find($contract->plan_id);

        // Check if under committed
        if ($trueUpRecord->actual_spend < $trueUpRecord->committed_amount) {
            $shortfall = $trueUpRecord->committed_amount - $trueUpRecord->actual_spend;

            // True-up charge for not meeting commitment
            $trueUpCharge = $shortfall * ($plan->discount_percentage / 100);

            Capsule::table('mod_commitment_trueup')
                ->where('id', $trueUpRecord->id)
                ->update([
                    'true_up_charge' => $trueUpCharge,
                    'status' => 'invoiced',
                ]);

            if ($trueUpCharge > 0) {
                $invoiceResult = $this->createTrueUpInvoice($contract->user_id, $contractId, $currentPeriod, $trueUpCharge);

                if ($invoiceResult['success']) {
                    Capsule::table('mod_commitment_trueup')
                        ->where('id', $trueUpRecord->id)
                        ->update([
                            'invoice_id' => $invoiceResult['invoice_id'],
                            'processed_at' => date('Y-m-d H:i:s'),
                        ]);
                }
            }
        } else {
            // Met or exceeded commitment - carry forward excess
            Capsule::table('mod_commitment_trueup')
                ->where('id', $trueUpRecord->id)
                ->update(['status' => 'settled']);
        }

        // Create next period record
        $nextPeriod = date('Y-m', strtotime('+1 month'));
        $existingNext = Capsule::table('mod_commitment_trueup')
            ->where('contract_id', $contractId)
            ->where('period', $nextPeriod)
            ->first();

        if (!$existingNext) {
            Capsule::table('mod_commitment_trueup')->insert([
                'contract_id' => $contractId,
                'period' => $nextPeriod,
                'committed_amount' => $contract->committed_amount,
                'actual_spend' => 0,
                'carry_forward' => $trueUpRecord->carry_forward,
            ]);
        }

        return [
            'success' => true,
            'period' => $currentPeriod,
            'actual_spend' => $trueUpRecord->actual_spend,
            'committed' => $trueUpRecord->committed_amount,
            'true_up_charge' => $trueUpRecord->true_up_charge,
        ];
    }

    private function createTrueUpInvoice(int $userId, int $contractId, string $period, float $amount): array {
        $contract = Capsule::table('mod_commitment_contracts')->find($contractId);

        $params = [
            'userid' => $userId,
            'date' => date('Y-m-d'),
            'duedate' => date('Y-m-d', strtotime('+7 days')),
            'itemdescription' => ["Committed Spend True-Up ({$period}) - Contract {$contract->contract_id}"],
            'itemamount' => [$amount],
        ];

        return localAPI('CreateInvoice', $params);
    }

    public function addCredits(int $contractId, float $amount, string $reason): array {
        Capsule::table('mod_commitment_credits')->insert([
            'contract_id' => $contractId,
            'credit_amount' => $amount,
            'reason' => $reason,
        ]);

        // Update current period
        $period = date('Y-m');
        $this->updateTrueup($contractId, $period);

        return [
            'success' => true,
            'credit_amount' => $amount,
            'new_balance' => $this->getContractCredits($contractId),
        ];
    }

    private function getContractCredits(int $contractId): float {
        return Capsule::table('mod_commitment_credits')
            ->where('contract_id', $contractId)
            ->sum('credit_amount') ?? 0;
    }

    public function getContractStatus(int $contractId): array {
        $contract = Capsule::table('mod_commitment_contracts')->find($contractId);

        if (!$contract) {
            return ['success' => false, 'error' => 'Contract not found'];
        }

        $currentPeriod = date('Y-m');
        $trueUp = Capsule::table('mod_commitment_trueup')
            ->where('contract_id', $contractId)
            ->where('period', $currentPeriod)
            ->first();

        $plan = Capsule::table('mod_commitment_plans')->find($contract->plan_id);

        return [
            'success' => true,
            'contract' => $contract,
            'plan' => $plan,
            'current_period' => [
                'committed' => $trueUp->committed_amount ?? 0,
                'actual_spend' => $trueUp->actual_spend ?? 0,
                'carry_forward' => $trueUp->carry_forward ?? 0,
                'remaining' => ($trueUp->committed_amount ?? 0) + ($trueUp->carry_forward ?? 0) - ($trueUp->actual_spend ?? 0),
                'on_track' => ($trueUp->actual_spend ?? 0) >= ($trueUp->committed_amount ?? 0),
            ],
        ];
    }

    public function renewContract(int $contractId): array {
        $contract = Capsule::table('mod_commitment_contracts')->find($contractId);

        if (!$contract || $contract->status !== 'active') {
            return ['success' => false, 'error' => 'Invalid contract'];
        }

        $plan = Capsule::table('mod_commitment_plans')->find($contract->plan_id);

        $newStart = new DateTime($contract->end_date);
        $newEnd = clone $newStart;
        $newEnd->modify("+{$plan->min_term_months} months");

        Capsule::table('mod_commitment_contracts')
            ->where('id', $contractId)
            ->update(['status' => 'expired']);

        $newContractId = Capsule::table('mod_commitment_contracts')->insertGetId([
            'user_id' => $contract->user_id,
            'plan_id' => $contract->plan_id,
            'contract_id' => 'CN-' . strtoupper(substr(md5($contractId . time()), 0, 8)),
            'committed_amount' => $contract->committed_amount,
            'start_date' => $newStart->format('Y-m-d'),
            'end_date' => $newEnd->format('Y-m-d'),
            'status' => 'active',
            'notes' => "Renewed from contract {$contract->id}",
        ]);

        return [
            'success' => true,
            'new_contract_id' => $newContractId,
            'start_date' => $newStart->format('Y-m-d'),
            'end_date' => $newEnd->format('Y-m-d'),
        ];
    }
}
```

## Client Area

```php
<?php
function commitment_billing_clientarea(array $vars): array {
    $userId = $_SESSION['uid'];
    $manager = new CommitmentBillingManager();

    $contracts = Capsule::table('mod_commitment_contracts')
        ->join('mod_commitment_plans', 'mod_commitment_contracts.plan_id', '=', 'mod_commitment_plans.id')
        ->where('mod_commitment_contracts.user_id', $userId)
        ->where('mod_commitment_contracts.status', 'active')
        ->get();

    $statuses = [];
    foreach ($contracts as $contract) {
        $statuses[$contract->id] = $manager->getContractStatus($contract->id);
    }

    return [
        'pagetitle' => 'My Commitment',
        'templatefile' => 'commitment',
        'vars' => [
            'contracts' => $contracts,
            'statuses' => $statuses,
        ],
    ];
}
```

---

**Related Skills:**
- whmcs-overage-billing
- whmcs-usage-tracking
- whmcs-recurring-billing