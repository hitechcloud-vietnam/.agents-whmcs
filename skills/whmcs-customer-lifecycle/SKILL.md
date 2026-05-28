# WHMCS Customer Lifecycle Management Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Track and manage customer lifecycle stages with automated transitions.

## Lifecycle Stages

```
PROSPECT → LEAD → CUSTOMER → VIP → CHURNED → REACTIVATED
```

## Database Schema

```php
<?php
// modules/addons/customer_lifecycle/customer_lifecycle.php

use WHMCS\Database\Capsule;

function customer_lifecycle_config(): array {
    return [
        'name' => 'Customer Lifecycle',
        'description' => 'Track and manage customer lifecycle stages',
        'version' => '1.0',
    ];
}

function customer_lifecycle_activate(): array {
    Capsule::schema()->create('mod_lifecycle_stages', function($t) {
        $t->increments('id');
        $t->string('name', 50)->unique();
        $t->string('display_name', 100);
        $t->text('description')->nullable();
        $t->integer('order')->unsigned();
        $t->string('color', 7)->default('#000000');
        $t->boolean('is_active')->default(true);
    });

    Capsule::schema()->create('mod_lifecycle_transitions', function($t) {
        $t->increments('id');
        $t->integer('from_stage_id')->unsigned();
        $t->integer('to_stage_id')->unsigned();
        $t->string('trigger_type', 50);
        $t->text('trigger_conditions')->nullable();
        $t->string('action', 50)->nullable();
        $t->boolean('is_active')->default(true);
    });

    Capsule::schema()->create('mod_lifecycle_user_progress', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->integer('stage_id')->unsigned();
        $t->timestamp('entered_at')->useCurrent();
        $t->timestamp('exited_at')->nullable();
        $t->string('transition_reason', 255)->nullable();
        $t->json('metadata')->nullable();
    });

    Capsule::schema()->create('mod_lifecycle_activities', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->string('activity_type', 50);
        $t->integer('points')->default(0);
        $t->text('description')->nullable();
        $t->timestamp('created_at')->useCurrent();
    });

    // Insert default stages
    $stages = [
        ['name' => 'prospect', 'display_name' => 'Prospect', 'order' => 1, 'color' => '#808080'],
        ['name' => 'lead', 'display_name' => 'Lead', 'order' => 2, 'color' => '#FFA500'],
        ['name' => 'customer', 'display_name' => 'Customer', 'order' => 3, 'color' => '#008000'],
        ['name' => 'vip', 'display_name' => 'VIP', 'order' => 4, 'color' => '#FFD700'],
        ['name' => 'churned', 'display_name' => 'Churned', 'order' => 5, 'color' => '#FF0000'],
        ['name' => 'reactivated', 'display_name' => 'Reactivated', 'order' => 6, 'color' => '#00CED1'],
    ];

    foreach ($stages as $stage) {
        Capsule::table('mod_lifecycle_stages')->insert($stage);
    }

    return ['status' => 'success'];
}

function customer_lifecycle_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_lifecycle_activities');
    Capsule::schema()->dropIfExists('mod_lifecycle_user_progress');
    Capsule::schema()->dropIfExists('mod_lifecycle_transitions');
    Capsule::schema()->dropIfExists('mod_lifecycle_stages');
    return ['status' => 'success'];
}
```

## Lifecycle Manager Class

```php
<?php
class CustomerLifecycleManager {
    private array $stageCache = [];

    public function getCurrentStage(int $userId): ?object {
        return Capsule::table('mod_lifecycle_user_progress')
            ->where('user_id', $userId)
            ->whereNull('exited_at')
            ->orderBy('entered_at', 'desc')
            ->first();
    }

    public function transitionUser(int $userId, string $toStageName, string $reason = '', array $metadata = []): array {
        $stage = $this->getStageByName($toStageName);
        if (!$stage) {
            return ['success' => false, 'error' => 'Invalid stage'];
        }

        $currentStage = $this->getCurrentStage($userId);

        if ($currentStage && $currentStage->stage_id === $stage->id) {
            return ['success' => false, 'error' => 'User already in this stage'];
        }

        // Exit current stage
        if ($currentStage) {
            Capsule::table('mod_lifecycle_user_progress')
                ->where('id', $currentStage->id)
                ->update(['exited_at' => date('Y-m-d H:i:s')]);
        }

        // Enter new stage
        $progressId = Capsule::table('mod_lifecycle_user_progress')->insertGetId([
            'user_id' => $userId,
            'stage_id' => $stage->id,
            'transition_reason' => $reason,
            'metadata' => json_encode($metadata),
        ]);

        // Log activity
        $this->logActivity($userId, 'stage_transition', [
            'from' => $currentStage->stage_id ?? null,
            'to' => $stage->id,
            'reason' => $reason,
        ]);

        // Execute transitions
        $this->executeTransitionActions($userId, $currentStage->stage_id ?? null, $stage->id);

        return ['success' => true, 'progress_id' => $progressId, 'stage' => $stage];
    }

    public function getStageHistory(int $userId): array {
        return Capsule::table('mod_lifecycle_user_progress')
            ->join('mod_lifecycle_stages', 'mod_lifecycle_user_progress.stage_id', '=', 'mod_lifecycle_stages.id')
            ->where('user_id', $userId)
            ->orderBy('entered_at', 'desc')
            ->get(['mod_lifecycle_user_progress.*', 'mod_lifecycle_stages.display_name']);
    }

    public function getStageMetrics(): array {
        $stages = Capsule::table('mod_lifecycle_stages')
            ->where('is_active', 1)
            ->orderBy('order')
            ->get();

        $metrics = [];
        foreach ($stages as $stage) {
            $currentCount = Capsule::table('mod_lifecycle_user_progress')
                ->where('stage_id', $stage->id)
                ->whereNull('exited_at')
                ->count();

            $totalCount = Capsule::table('mod_lifecycle_user_progress')
                ->where('stage_id', $stage->id)
                ->count();

            $metrics[] = [
                'stage' => $stage,
                'current_count' => $currentCount,
                'total_count' => $totalCount,
            ];
        }

        return $metrics;
    }

    private function getStageByName(string $name): ?object {
        if (isset($this->stageCache[$name])) {
            return $this->stageCache[$name];
        }

        $stage = Capsule::table('mod_lifecycle_stages')
            ->where('name', $name)
            ->where('is_active', 1)
            ->first();

        $this->stageCache[$name] = $stage;
        return $stage;
    }

    private function executeTransitionActions(int $userId, ?int $fromStageId, int $toStageId): void {
        $transitions = Capsule::table('mod_lifecycle_transitions')
            ->where('from_stage_id', $fromStageId)
            ->where('to_stage_id', $toStageId)
            ->where('is_active', 1)
            ->get();

        foreach ($transitions as $transition) {
            $this->executeAction($transition->action, $userId);
        }
    }

    private function executeAction(string $action, int $userId): void {
        switch ($action) {
            case 'send_email':
                $user = Capsule::table('tblclients')->find($userId);
                sendTplEmail($user->email, 'lifecycle_stage_change', [
                    'client_name' => $user->firstname . ' ' . $user->lastname,
                ]);
                break;
            case 'apply_discount':
                // Apply customer discount
                break;
            case 'add_tag':
                // Add CRM tag
                break;
        }
    }

    private function logActivity(int $userId, string $type, array $data): void {
        Capsule::table('mod_lifecycle_activities')->insert([
            'user_id' => $userId,
            'activity_type' => $type,
            'points' => $data['points'] ?? 0,
            'description' => json_encode($data),
        ]);
    }

    public function calculateEngagementScore(int $userId): int {
        $activities = Capsule::table('mod_lifecycle_activities')
            ->where('user_id', $userId)
            ->where('created_at', '>=', date('Y-m-d', strtotime('-90 days')))
            ->get();

        $score = 0;
        foreach ($activities as $activity) {
            $score += $activity->points;
        }

        return $score;
    }
}
```

## Automated Transitions

```php
// Cron job for automated lifecycle transitions
add_hook('DailyCronJob', 1, function($vars) {
    $lifecycleManager = new CustomerLifecycleManager();

    // Check for VIP promotion
    $vipCriteria = Capsule::table('tblorders')
        ->selectRaw('userid, SUM(amount) as total_spent')
        ->where('status', 'Completed')
        ->where('date', '>=', date('Y-m-d', strtotime('-365 days')))
        ->groupBy('userid')
        ->having('total_spent', '>=', 10000)
        ->get();

    foreach ($vipCriteria as $order) {
        $currentStage = $lifecycleManager->getCurrentStage($order->userid);
        if ($currentStage && $currentStage->stage_name !== 'vip') {
            $lifecycleManager->transitionUser($order->userid, 'vip', 'Total spend exceeded $10,000');
        }
    }

    // Check for churn risk
    $churnRiskUsers = $this->identifyChurnRiskUsers();
    foreach ($churnRiskUsers as $userId) {
        $currentStage = $lifecycleManager->getCurrentStage($userId);
        if ($currentStage && $currentStage->stage_name === 'customer') {
            $lifecycleManager->transitionUser($userId, 'churned', 'No activity for 90+ days');
        }
    }
});

private function identifyChurnRiskUsers(): array {
    $threshold = date('Y-m-d', strtotime('-90 days'));

    $users = Capsule::table('tblclients')
        ->leftJoin('tblorders', 'tblclients.id', '=', 'tblorders.userid')
        ->leftJoin('tblticketitems', 'tblclients.id', '=', 'tblticketitems.userid')
        ->whereRaw('(SELECT MAX(date) FROM tblorders WHERE userid = tblclients.id AND status = "Completed") < ?', [$threshold])
        ->whereRaw('(SELECT MAX(created) FROM tblticketitems WHERE userid = tblclients.id) < ?', [$threshold])
        ->groupBy('tblclients.id')
        ->pluck('tblclients.id');

    return $users->toArray();
}
```

## Customer Journey Visualization

```php
function customer_lifecycle_output(array $vars): void {
    $lifecycleManager = new CustomerLifecycleManager();

    $metrics = $lifecycleManager->getStageMetrics();

    echo '<div class="lifecycle-dashboard">';
    echo '<h2>Customer Lifecycle Overview</h2>';

    echo '<div class="stage-funnel">';
    foreach ($metrics as $metric) {
        $stage = $metric['stage'];
        echo '<div class="stage-block" style="background-color: ' . $stage->color . '">';
        echo '<h3>' . $stage->display_name . '</h3>';
        echo '<div class="count">' . $metric['current_count'] . '</div>';
        echo '<div class="total">Total: ' . $metric['total_count'] . '</div>';
        echo '</div>';
    }
    echo '</div>';

    echo '<h3>Recent Transitions</h3>';
    $recentTransitions = Capsule::table('mod_lifecycle_user_progress')
        ->join('mod_lifecycle_stages', 'mod_lifecycle_user_progress.stage_id', '=', 'mod_lifecycle_stages.id')
        ->join('tblclients', 'mod_lifecycle_user_progress.user_id', '=', 'tblclients.id')
        ->orderBy('entered_at', 'desc')
        ->limit(20)
        ->get(['mod_lifecycle_user_progress.*', 'mod_lifecycle_stages.display_name', 'tblclients.email']);

    echo '<table class="datatable"><thead><tr>';
    echo '<th>Customer</th><th>Stage</th><th>Entered</th><th>Reason</th>';
    echo '</tr></thead><tbody>';

    foreach ($recentTransitions as $t) {
        echo '<tr>';
        echo "<td>{$t->email}</td>";
        echo "<td>{$t->display_name}</td>";
        echo "<td>{$t->entered_at}</td>";
        echo "<td>{$t->transition_reason}</td>";
        echo '</tr>';
    }

    echo '</tbody></table>';
    echo '</div>';
}
```

---

**Related Skills:**
- whmcs-client-management
- whmcs-loyalty-program
- whmcs-reporting