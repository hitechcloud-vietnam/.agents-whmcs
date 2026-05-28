# WHMCS Fraud Detection Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing fraud detection and risk management modules.

## When to Use

- Building fraud detection modules
- Creating risk scoring systems
- Implementing order validation

## Fraud Detection Module Pattern

```php
<?php
// modules/fraud/{fraudmodule}/{fraudmodule}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {fraudmodule}_config(): array {
    return [
        'name' => 'Fraud Detection',
        'description' => 'Advanced fraud detection module',
        'version' => '1.0',
        'author' => 'Author',
        'api_key' => ['FriendlyName' => 'API Key', 'Type' => 'password'],
        'risk_threshold' => ['FriendlyName' => 'Risk Threshold', 'Type' => 'text', 'Default' => '70'],
    ];
}

function {fraudmodule}_activate(): array {
    Capsule::schema()->create('mod_fraud_logs', function($t) {
        $t->increments('id');
        $t->integer('userid')->unsigned();
        $t->string('order_id', 100);
        $t->string('risk_level', 20);
        $t->integer('risk_score')->unsigned();
        $t->text('risk_factors');
        $t->string('action_taken', 50);
        $t->timestamp('created_at');
    });

    Capsule::schema()->create('mod_fraud_rules', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->text('condition');
        $t->integer('score_weight')->default(10);
        $t->boolean('active')->default(true);
        $t->timestamp('created_at');
    });

    return ['status' => 'success', 'description' => 'Fraud module activated'];
}

function {fraudmodule}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_fraud_logs');
    Capsule::schema()->dropIfExists('mod_fraud_rules');
    return ['status' => 'success', 'description' => 'Fraud module deactivated'];
}

function {fraudmodule}_output(array $vars): void {
    $action = $_GET['action'] ?? 'dashboard';

    echo '<div class="fraud-module">';
    echo '<h1>Fraud Detection Dashboard</h1>';
    echo '</div>';

    switch ($action) {
        case 'rules':
            include __DIR__ . '/templates/admin/rules.tpl';
            break;
        case 'logs':
            include __DIR__ . '/templates/admin/logs.tpl';
            break;
        case 'settings':
            include __DIR__ . '/templates/admin/settings.tpl';
            break;
        default:
            include __DIR__ . '/templates/admin/dashboard.tpl';
    }
}

// Core fraud check hook
function {fraudmodule}_checkOrderRisk(array $params): array {
    $api = new \Fraud\ApiClient($params);

    $riskScore = 0;
    $riskFactors = [];

    // Check IP reputation (example rules)
    $ipCheck = $api->checkIP($_SERVER['REMOTE_ADDR']);
    if ($ipCheck['is_proxy'] || $ipCheck['is_vpn']) {
        $riskScore += 25;
        $riskFactors[] = 'Suspicious IP (proxy/VPN detected)';
    }

    // Check email domain reputation
    $emailDomain = explode('@', $params['email'])[1] ?? '';
    if ($api->isDisposableEmail($emailDomain)) {
        $riskScore += 30;
        $riskFactors[] = 'Disposable email domain';
    }

    // Check for free email providers (lower risk)
    $freeProviders = ['gmail.com', 'yahoo.com', 'outlook.com', 'hotmail.com'];
    if (in_array(strtolower($emailDomain), $freeProviders)) {
        $riskScore += 5;
        $riskFactors[] = 'Free email provider';
    }

    // Check billing/shipping match
    if ($params['billing_firstname'] !== $params['shipping_firstname'] ?? '') {
        $riskScore += 10;
        $riskFactors[] = 'Billing/shipping name mismatch';
    }

    // Check high-value orders
    if (($params['amount'] ?? 0) > 500) {
        $riskScore += 15;
        $riskFactors[] = 'High-value order (>$500)';
    }

    // Custom rules from database
    $rules = Capsule::table('mod_fraud_rules')
        ->where('active', 1)
        ->get();

    foreach ($rules as $rule) {
        if (evaluateRule($rule->condition, $params)) {
            $riskScore += $rule->score_weight;
            $riskFactors[] = $rule->name;
        }
    }

    // Determine action
    $threshold = (int) Capsule::table('mod_configuration')
        ->where('setting', 'risk_threshold')
        ->value('value') ?? 70;

    $action = 'approve';
    if ($riskScore >= $threshold) {
        $action = 'reject';
    } elseif ($riskScore >= ($threshold * 0.6)) {
        $action = 'review';
    }

    // Log the check
    Capsule::table('mod_fraud_logs')->insert([
        'userid' => $params['userid'] ?? 0,
        'order_id' => $params['order_id'] ?? '',
        'risk_level' => $action,
        'risk_score' => $riskScore,
        'risk_factors' => json_encode($riskFactors),
        'action_taken' => $action,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return [
        'action' => $action,
        'risk_score' => $riskScore,
        'risk_factors' => $riskFactors,
    ];
}

function evaluateRule(string $condition, array $params): bool {
    // Simple rule evaluator
    extract($params);
    return eval('return ' . $condition . ';');
}
```

### Fraud Check Hooks

```php
// hooks.php - Register fraud check hooks

add_hook('OrderProductValidation', 1, function($vars) {
    $fraudResult = {fraudmodule}_checkOrderRisk($vars);

    if ($fraudResult['action'] === 'reject') {
        return [
            'abort' => true,
            'error' => 'Order could not be processed. Please contact support.',
        ];
    }

    if ($fraudResult['action'] === 'review') {
        // Flag for manual review
        Capsule::table('tblorders')
            ->where('id', $vars['order_id'])
            ->update(['status' => 'Fraud']);
    }
});
```

### Admin Dashboard Template

```smarty
<div class="fraud-dashboard">
    <div class="stats-row">
        <div class="stat-box">
            <div class="stat-value text-danger">{$high_risk_count}</div>
            <div class="stat-label">High Risk</div>
        </div>
        <div class="stat-box">
            <div class="stat-value text-warning">{$review_count}</div>
            <div class="stat-label">Under Review</div>
        </div>
        <div class="stat-box">
            <div class="stat-value text-success">{$approved_count}</div>
            <div class="stat-label">Approved</div>
        </div>
        <div class="stat-box">
            <div class="stat-value">{$avg_risk_score}</div>
            <div class="stat-label">Avg Risk Score</div>
        </div>
    </div>

    <h3>Recent Fraud Checks</h3>
    <table class="data-table">
        <thead>
            <tr>
                <th>Order ID</th>
                <th>Risk Score</th>
                <th>Risk Level</th>
                <th>Factors</th>
                <th>Action</th>
                <th>Date</th>
            </tr>
        </thead>
        <tbody>
            {foreach $logs as $log}
            <tr>
                <td>#{$log->order_id}</td>
                <td>{$log->risk_score}</td>
                <td><span class="badge badge-{$log->risk_level}">{$log->risk_level}</span></td>
                <td>{$log->risk_factors|json_decode|join:', '}</td>
                <td>{$log->action_taken}</td>
                <td>{$log->created_at|date_format:'%Y-%m-d H:i'}</td>
            </tr>
            {/foreach}
        </tbody>
    </table>
</div>
```

---

**Related Skills:**
- whmcs-hooks-development
- whmcs-security-hardening
- whmcs-admin-ui-builder
