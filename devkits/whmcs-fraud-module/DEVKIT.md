# WHMCS Fraud Detection Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## Purpose
Create a fraud detection and prevention module for WHMCS that analyzes orders and customers for suspicious activity.

## Module Type
Addon Module with API Integration

## Use Case
- Detect fraudulent orders before processing
- Check customer risk scores
- Block high-risk transactions
- Log fraud attempts for review

## DevKit Structure

```
devkits/whmcs-fraud-module/
├── fraud.php              # Main addon module
├── lib/
│   ├── FraudChecker.php  # Fraud check API client
│   └── RiskScorer.php    # Risk scoring logic
├── templates/
│   ├── admin.tpl         # Admin configuration
│   └── admin-dashboard.tpl # Fraud dashboard
├── hooks.php             # Hook integrations
└── DEVKIT.md            # This file
```

## Main Module Template

```php
<?php
/**
 * WHMCS Fraud Detection Module: {Fraud}
 * Fraud Detection/Prevention Module Template
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {fraud}_config(): array {
    return [
        'name' => '{Fraud Detection}',
        'description' => 'Advanced fraud detection and prevention for WHMCS orders',
        'version' => '1.0',
        'author' => '{Author Name}',
        'language' => 'english',

        // API Configuration
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'Your fraud detection API key',
        ],
        'api_endpoint' => [
            'FriendlyName' => 'API Endpoint',
            'Type' => 'text',
            'Size' => '80',
            'Default' => 'https://api.fraudprovider.com/v1',
        ],
        'test_mode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
        ],
        'auto_block' => [
            'FriendlyName' => 'Auto Block High Risk',
            'Type' => 'yesno',
            'Description' => 'Automatically block orders with high risk score',
        ],
        'risk_threshold' => [
            'FriendlyName' => 'Risk Threshold',
            'Type' => 'dropdown',
            'Options' => '50,60,70,80,90',
            'Default' => '70',
            'Description' => 'Orders above this score will be flagged',
        ],
        'email_alerts' => [
            'FriendlyName' => 'Email Alerts',
            'Type' => 'yesno',
            'Description' => 'Send email alerts for high-risk orders',
        ],
        'alert_email' => [
            'FriendlyName' => 'Alert Email',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'Email address for fraud alerts',
        ],
    ];
}

function {fraud}_activate(): array {
    try {
        // Fraud checks log table
        if (!Capsule::schema()->hasTable('mod_{fraud}_checks')) {
            Capsule::schema()->create('mod_{fraud}_checks', function($t) {
                $t->increments('id');
                $t->string('order_id', 50);
                $t->integer('user_id')->unsigned()->nullable();
                $t->string('email', 255);
                $t->string('ip_address', 45);
                $t->decimal('risk_score', 5, 2)->default(0);
                $t->string('risk_level', 20)->default('low');
                $t->text('risk_factors')->nullable();
                $t->string('status', 20)->default('pending');
                $t->text('api_response')->nullable();
                $t->string('decision', 20)->default('pending');
                $t->text('notes')->nullable();
                $t->timestamp('checked_at');
                $t->timestamp('reviewed_at')->nullable();
                $t->integer('reviewed_by')->unsigned()->nullable();
                $t->timestamps();

                $t->index('order_id');
                $t->index('risk_level');
                $t->index('status');
            });
        }

        // Fraud rules table
        if (!Capsule::schema()->hasTable('mod_{fraud}_rules')) {
            Capsule::schema()->create('mod_{fraud}_rules', function($t) {
                $t->increments('id');
                $t->string('rule_name', 100);
                $t->string('rule_type', 50);
                $t->text('rule_config');
                $t->integer('risk_points')->default(0);
                $t->string('status', 20)->default('active');
                $t->integer('sort_order')->default(0);
                $t->timestamps();
            });
        }

        // Fraud statistics table
        if (!Capsule::schema()->hasTable('mod_{fraud}_stats')) {
            Capsule::schema()->create('mod_{fraud}_stats', function($t) {
                $t->increments('id');
                $t->date('date');
                $t->integer('total_checks')->default(0);
                $t->integer('blocked_count')->default(0);
                $t->integer('flagged_count')->default(0);
                $t->integer('approved_count')->default(0);
                $t->decimal('blocked_amount', 15, 2)->default(0);
                $t->timestamps();

                $t->unique('date');
            });
        }

        // Insert default rules
        $defaultRules = [
            ['rule_name' => 'Free Email Domain', 'rule_type' => 'email_domain', 'rule_config' => json_encode(['domains' => ['gmail.com', 'yahoo.com', 'hotmail.com']]), 'risk_points' => 10, 'sort_order' => 1],
            ['rule_name' => 'High Risk Country', 'rule_type' => 'country', 'rule_config' => json_encode(['countries' => []]), 'risk_points' => 25, 'sort_order' => 2],
            ['rule_name' => 'Proxy/VPN Detected', 'rule_type' => 'ip_check', 'rule_config' => json_encode([]), 'risk_points' => 30, 'sort_order' => 3],
            ['rule_name' => 'Disposable Email', 'rule_type' => 'email_disposable', 'rule_config' => json_encode([]), 'risk_points' => 35, 'sort_order' => 4],
            ['rule_name' => 'IP Mismatch', 'rule_type' => 'ip_mismatch', 'rule_config' => json_encode([]), 'risk_points' => 20, 'sort_order' => 5],
        ];

        foreach ($defaultRules as $rule) {
            Capsule::table('mod_{fraud}_rules')->insert($rule);
        }

        return [
            'status' => 'success',
            'description' => '{Fraud Detection} module activated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Activation failed: ' . $e->getMessage(),
        ];
    }
}

function {fraud}_deactivate(): array {
    try {
        Capsule::schema()->dropIfExists('mod_{fraud}_checks');
        Capsule::schema()->dropIfExists('mod_{fraud}_rules');
        Capsule::schema()->dropIfExists('mod_{fraud}_stats');

        return [
            'status' => 'success',
            'description' => '{Fraud Detection} module deactivated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Deactivation failed: ' . $e->getMessage(),
        ];
    }
}

function {fraud}_output(array $vars): void {
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
        handleAdminAction($_POST['action'] ?? 'dashboard');
    }

    $action = $_REQUEST['action'] ?? 'dashboard';
    $templateData = prepareFraudData($action);

    echo '<div class="fraud-module">';
    echo '<div class="fraud-header">';
    echo '<h1><i class="fa fa-shield"></i> Fraud Detection</h1>';
    echo '<div class="header-actions">';
    echo '<a href="?module={fraud}&action=settings" class="btn"><i class="fa fa-cog"></i> Settings</a>';
    echo '<a href="?module={fraud}&action=rules" class="btn"><i class="fa fa-list"></i> Rules</a>';
    echo '</div>';
    echo '</div>';

    include __DIR__ . '/templates/admin/' . $action . '.tpl';
    echo '</div>';
}

function {fraud}_clientarea(array $vars): array {
    return [
        'pagetitle' => 'Fraud Verification',
        'templatefile' => 'templates/clientarea/clientarea',
        'vars' => [],
        'requirelogin' => true,
    ];
}

// Helper Functions
function handleAdminAction(string $action): void {
    switch ($action) {
        case 'review':
            reviewFraudCheck();
            break;
        case 'update_rule':
            updateFraudRule();
            break;
        case 'add_rule':
            addFraudRule();
            break;
        case 'delete_rule':
            deleteFraudRule();
            break;
        case 'bulk_action':
            bulkFraudAction();
            break;
    }

    header('Location: ?module={fraud}&action=' . ($_POST['redirect'] ?? 'dashboard'));
    exit;
}

function prepareFraudData(string $action): array {
    return match ($action) {
        'dashboard' => getFraudDashboard(),
        'pending' => getPendingChecks(),
        'rules' => getFraudRules(),
        'settings' => getSettingsData(),
        'stats' => getFraudStats(),
        default => getFraudDashboard(),
    };
}

function getFraudDashboard(): array {
    $pending = Capsule::table('mod_{fraud}_checks')
        ->where('status', 'pending')
        ->count();

    $todayChecks = Capsule::table('mod_{fraud}_checks')
        ->whereDate('checked_at', date('Y-m-d'))
        ->count();

    $blockedToday = Capsule::table('mod_{fraud}_checks')
        ->whereDate('checked_at', date('Y-m-d'))
        ->where('decision', 'blocked')
        ->count();

    $recentFlagged = Capsule::table('mod_{fraud}_checks')
        ->whereIn('risk_level', ['high', 'medium'])
        ->orderBy('checked_at', 'desc')
        ->limit(10)
        ->get();

    return [
        'pending' => $pending,
        'today_checks' => $todayChecks,
        'blocked_today' => $blockedToday,
        'recent_flagged' => $recentFlagged,
    ];
}

function getPendingChecks(): array {
    $page = max(1, (int)($_GET['page'] ?? 1));
    $perPage = 20;
    $offset = ($page - 1) * $perPage;

    $checks = Capsule::table('mod_{fraud}_checks')
        ->where('status', 'pending')
        ->orderBy('risk_score', 'desc')
        ->limit($perPage)
        ->offset($offset)
        ->get();

    $total = Capsule::table('mod_{fraud}_checks')
        ->where('status', 'pending')
        ->count();

    return [
        'checks' => $checks,
        'total' => $total,
        'page' => $page,
        'per_page' => $perPage,
    ];
}

function getFraudRules(): array {
    $rules = Capsule::table('mod_{fraud}_rules')
        ->orderBy('sort_order')
        ->get();

    return ['rules' => $rules];
}

function reviewFraudCheck(): void {
    $checkId = (int)($_POST['check_id'] ?? 0);
    $decision = $_POST['decision'] ?? '';
    $notes = $_POST['notes'] ?? '';

    if (in_array($decision, ['approved', 'blocked', 'flagged'])) {
        Capsule::table('mod_{fraud}_checks')
            ->where('id', $checkId)
            ->update([
                'decision' => $decision,
                'status' => 'reviewed',
                'notes' => $notes,
                'reviewed_at' => date('Y-m-d H:i:s'),
                'reviewed_by' => $_SESSION['adminid'],
            ]);

        // If blocking, cancel the order
        if ($decision === 'blocked') {
            $check = Capsule::table('mod_{fraud}_checks')
                ->where('id', $checkId)
                ->first();
            if ($check && $check->order_id) {
                localAPI('CancelOrder', ['orderid' => $check->order_id, 'sendnotification' => false]);
            }
        }
    }
}

function updateFraudRule(): void {
    $ruleId = (int)($_POST['rule_id'] ?? 0);
    $riskPoints = (int)($_POST['risk_points'] ?? 0);
    $status = $_POST['status'] ?? 'active';

    Capsule::table('mod_{fraud}_rules')
        ->where('id', $ruleId)
        ->update([
            'risk_points' => $riskPoints,
            'status' => $status,
        ]);
}

function addFraudRule(): void {
    Capsule::table('mod_{fraud}_rules')->insert([
        'rule_name' => $_POST['rule_name'] ?? 'New Rule',
        'rule_type' => $_POST['rule_type'] ?? 'custom',
        'rule_config' => $_POST['rule_config'] ?? '{}',
        'risk_points' => (int)($_POST['risk_points'] ?? 0),
        'status' => 'active',
        'sort_order' => (int)($_POST['sort_order'] ?? 99),
    ]);
}

function deleteFraudRule(): void {
    $ruleId = (int)($_GET['id'] ?? 0);
    Capsule::table('mod_{fraud}_rules')
        ->where('id', $ruleId)
        ->delete();
}

function bulkFraudAction(): void {
    $checkIds = $_POST['check_ids'] ?? [];
    $action = $_POST['bulk_decision'] ?? '';

    foreach ($checkIds as $checkId) {
        Capsule::table('mod_{fraud}_checks')
            ->where('id', (int)$checkId)
            ->update([
                'decision' => $action,
                'status' => 'reviewed',
                'reviewed_at' => date('Y-m-d H:i:s'),
                'reviewed_by' => $_SESSION['adminid'],
            ]);
    }
}
```

## Fraud Checker Class

```php
<?php
namespace WHMCS\Module\Addon\{Fraud};

class FraudChecker {
    private string $apiKey;
    private string $apiEndpoint;
    private bool $testMode;

    public function __construct(array $settings) {
        $this->apiKey = $settings['api_key'] ?? '';
        $this->apiEndpoint = $settings['api_endpoint'] ?? 'https://api.fraudprovider.com/v1';
        $this->testMode = ($settings['test_mode'] ?? '') === 'on';
    }

    public function checkOrder(array $orderData): array {
        $payload = [
            'order_id' => $orderData['order_id'] ?? '',
            'customer' => [
                'email' => $orderData['email'] ?? '',
                'first_name' => $orderData['first_name'] ?? '',
                'last_name' => $orderData['last_name'] ?? '',
                'phone' => $orderData['phone'] ?? '',
                'company' => $orderData['company'] ?? '',
            ],
            'billing' => [
                'address1' => $orderData['address1'] ?? '',
                'address2' => $orderData['address2'] ?? '',
                'city' => $orderData['city'] ?? '',
                'state' => $orderData['state'] ?? '',
                'postcode' => $orderData['postcode'] ?? '',
                'country' => $orderData['country'] ?? '',
            ],
            'ip_address' => $orderData['ip_address'] ?? '',
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'amount' => $orderData['amount'] ?? 0,
            'currency' => $orderData['currency'] ?? 'USD',
        ];

        return $this->makeRequest('/check', $payload);
    }

    public function checkIP(string $ipAddress): array {
        return $this->makeRequest('/ip-check', ['ip' => $ipAddress]);
    }

    public function checkEmail(string $email): array {
        return $this->makeRequest('/email-check', ['email' => $email]);
    }

    private function makeRequest(string $endpoint, array $data): array {
        $url = $this->apiEndpoint . $endpoint;

        if ($this->testMode) {
            $url = str_replace('api.fraudprovider.com', 'sandbox.fraudprovider.com', $url);
        }

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'Authorization: Bearer ' . $this->apiKey,
            ],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('API Error: ' . $error);
        }

        $result = json_decode($response, true);

        if ($httpCode >= 400) {
            throw new \Exception($result['message'] ?? 'API Error: HTTP ' . $httpCode);
        }

        return $result;
    }
}
```

## Risk Scorer Class

```php
<?php
namespace WHMCS\Module\Addon\{Fraud};

use WHMCS\Database\Capsule;

class RiskScorer {
    private array $rules = [];

    public function __construct() {
        $this->loadRules();
    }

    private function loadRules(): void {
        $this->rules = Capsule::table('mod_{fraud}_rules')
            ->where('status', 'active')
            ->orderBy('sort_order')
            ->get()
            ->toArray();
    }

    public function calculateScore(array $orderData): array {
        $totalScore = 0;
        $riskFactors = [];
        $riskDetails = [];

        foreach ($this->rules as $rule) {
            $config = json_decode($rule->rule_config, true) ?? [];
            $result = $this->evaluateRule($rule->rule_type, $rule->rule_name, $orderData, $config);

            if ($result['matched']) {
                $totalScore += $rule->risk_points;
                $riskFactors[] = [
                    'rule' => $rule->rule_name,
                    'points' => $rule->risk_points,
                    'reason' => $result['reason'] ?? 'Rule matched',
                ];
                $riskDetails[$rule->rule_type] = $result;
            }
        }

        $riskLevel = $this->determineRiskLevel($totalScore);

        return [
            'score' => $totalScore,
            'level' => $riskLevel,
            'factors' => $riskFactors,
            'details' => $riskDetails,
            'recommendation' => $this->getRecommendation($riskLevel),
        ];
    }

    private function evaluateRule(string $type, string $name, array $data, array $config): array {
        return match ($type) {
            'email_domain' => $this->checkEmailDomain($data['email'] ?? '', $config),
            'country' => $this->checkCountry($data['country'] ?? '', $config),
            'ip_check' => $this->checkIP($data['ip_address'] ?? '', $config),
            'email_disposable' => $this->checkDisposableEmail($data['email'] ?? '', $config),
            'ip_mismatch' => $this->checkIPMismatch($data, $config),
            'velocity' => $this->checkVelocity($data, $config),
            default => ['matched' => false],
        };
    }

    private function checkEmailDomain(string $email, array $config): array {
        $freeDomains = $config['domains'] ?? ['gmail.com', 'yahoo.com', 'hotmail.com'];
        $domain = strtolower(substr(strrchr($email, "@"), 1));

        if (in_array($domain, $freeDomains)) {
            return [
                'matched' => true,
                'reason' => "Free email domain: {$domain}",
            ];
        }

        return ['matched' => false];
    }

    private function checkCountry(string $country, array $config): array {
        $highRiskCountries = $config['countries'] ?? [];

        if (in_array($country, $highRiskCountries)) {
            return [
                'matched' => true,
                'reason' => "High-risk country: {$country}",
            ];
        }

        return ['matched' => false];
    }

    private function checkIP(string $ip, array $config): array {
        // Check for proxy/VPN indicators
        $suspiciousPatterns = ['proxy', 'vpn', 'tor', 'datacenter'];

        foreach ($suspiciousPatterns as $pattern) {
            if (stripos($ip, $pattern) !== false) {
                return [
                    'matched' => true,
                    'reason' => "Suspicious IP type: {$pattern}",
                ];
            }
        }

        return ['matched' => false];
    }

    private function checkDisposableEmail(string $email, array $config): array {
        $disposableDomains = $config['disposable_domains'] ?? [
            'tempmail.com', 'throwaway.email', 'guerrillamail.com',
            'mailinator.com', '10minutemail.com', 'fakeinbox.com',
        ];

        $domain = strtolower(substr(strrchr($email, "@"), 1));

        if (in_array($domain, $disposableDomains)) {
            return [
                'matched' => true,
                'reason' => "Disposable email domain: {$domain}",
            ];
        }

        return ['matched' => false];
    }

    private function checkIPMismatch(array $data, array $config): array {
        // Compare billing country with IP geolocation
        $ipCountry = $data['ip_country'] ?? '';
        $billingCountry = $data['country'] ?? '';

        if ($ipCountry && $billingCountry && $ipCountry !== $billingCountry) {
            return [
                'matched' => true,
                'reason' => "IP country ({$ipCountry}) differs from billing country ({$billingCountry})",
            ];
        }

        return ['matched' => false];
    }

    private function checkVelocity(array $data, array $config): array {
        // Check for multiple orders from same email/IP in short time
        $timeWindow = $config['time_window'] ?? 3600;
        $maxOrders = $config['max_orders'] ?? 3;

        $recentOrders = Capsule::table('mod_{fraud}_checks')
            ->where('email', $data['email'] ?? '')
            ->where('checked_at', '>', date('Y-m-d H:i:s', time() - $timeWindow))
            ->count();

        if ($recentOrders >= $maxOrders) {
            return [
                'matched' => true,
                'reason' => "Too many orders ({$recentOrders}) in time window",
            ];
        }

        return ['matched' => false];
    }

    private function determineRiskLevel(int $score): string {
        return match (true) {
            $score >= 80 => 'critical',
            $score >= 60 => 'high',
            $score >= 40 => 'medium',
            $score >= 20 => 'low',
            default => 'minimal',
        };
    }

    private function getRecommendation(string $riskLevel): string {
        return match ($riskLevel) {
            'critical' => 'block',
            'high' => 'review',
            'medium' => 'flag',
            default => 'approve',
        };
    }
}
```

## Hooks Integration

```php
<?php
/**
 * WHMCS Fraud Detection Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

add_hook('AcceptOrder', 1, function(array $vars) {
    $orderId = $vars['orderid'];
    runFraudCheck($orderId);
});

add_hook('ShoppingCartValidateCheckout', 1, function(array $vars) {
    $result = ['allowed' => true];

    $orderData = buildOrderData($vars);
    $scorer = new \WHMCS\Module\Addon\{Fraud}\RiskScorer();
    $riskResult = $scorer->calculateScore($orderData);

    $settings = getModuleSettings('{fraud}');

    if ($riskResult['score'] >= (int)($settings['risk_threshold'] ?? 70)) {
        if ($settings['auto_block'] === 'on') {
            $result['allowed'] = false;
            $result['errormessage'] = 'Order flagged for review. Please contact support.';
        }

        logFraudCheck($orderData, $riskResult);

        if ($settings['email_alerts'] === 'on' && $settings['alert_email']) {
            sendFraudAlert($orderData, $riskResult, $settings['alert_email']);
        }
    }

    return $result;
});

function runFraudCheck(int $orderId): array {
    $order = localAPI('GetOrders', ['id' => $orderId]);
    $orderData = $order['orders']['order'][0] ?? [];

    $scorer = new \WHMCS\Module\Addon\{Fraud}\RiskScorer();
    $riskResult = $scorer->calculateScore([
        'email' => $orderData['email'] ?? '',
        'ip_address' => $orderData['ip'] ?? '',
    ]);

    logFraudCheck(['order_id' => $orderId, 'email' => $orderData['email'] ?? ''], $riskResult);

    return $riskResult;
}

function logFraudCheck(array $orderData, array $riskResult): void {
    Capsule::table('mod_{fraud}_checks')->insert([
        'order_id' => $orderData['order_id'] ?? '',
        'user_id' => $orderData['user_id'] ?? null,
        'email' => $orderData['email'] ?? '',
        'ip_address' => $orderData['ip_address'] ?? '',
        'risk_score' => $riskResult['score'],
        'risk_level' => $riskResult['level'],
        'risk_factors' => json_encode($riskResult['factors']),
        'status' => 'pending',
        'api_response' => json_encode($riskResult),
        'decision' => $riskResult['recommendation'],
        'checked_at' => date('Y-m-d H:i:s'),
    ]);
}

function sendFraudAlert(array $orderData, array $riskResult, string $alertEmail): void {
    $subject = "[Fraud Alert] High-risk order detected - Score: {$riskResult['score']}";
    $message = "A high-risk order has been detected.\n\n";
    $message .= "Order ID: {$orderData['order_id']}\n";
    $message .= "Email: {$orderData['email']}\n";
    $message .= "IP: {$orderData['ip_address']}\n";
    $message .= "Risk Score: {$riskResult['score']}\n";
    $message .= "Risk Level: {$riskResult['level']}\n\n";
    $message .= "Risk Factors:\n";

    foreach ($riskResult['factors'] as $factor) {
        $message .= "- {$factor['rule']}: +{$factor['points']} points ({$factor['reason']})\n";
    }

    sendemail($alertEmail, $subject, $message);
}

function buildOrderData(array $vars): array {
    return [
        'order_id' => $vars['orderid'] ?? '',
        'email' => $vars['email'] ?? '',
        'first_name' => $vars['firstname'] ?? '',
        'last_name' => $vars['lastname'] ?? '',
        'phone' => $vars['phonenumber'] ?? '',
        'company' => $vars['companyname'] ?? '',
        'address1' => $vars['address1'] ?? '',
        'address2' => $vars['address2'] ?? '',
        'city' => $vars['city'] ?? '',
        'state' => $vars['state'] ?? '',
        'postcode' => $vars['postcode'] ?? '',
        'country' => $vars['country'] ?? '',
        'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
        'amount' => $vars['amount'] ?? 0,
    ];
}

function getModuleSettings(string $module): array {
    $result = Capsule::table('tbladdon_modules')
        ->where('module', $module)
        ->first();

    return $result ? json_decode($result->value, true) : [];
}
```

## Admin Dashboard Template

```smarty
<div class="fraud-dashboard">
    <div class="row">
        <div class="col-md-3">
            <div class="panel panel-warning">
                <div class="panel-body text-center">
                    <h3>{$pending}</h3>
                    <p>Pending Review</p>
                </div>
            </div>
        </div>
        <div class="col-md-3">
            <div class="panel panel-info">
                <div class="panel-body text-center">
                    <h3>{$today_checks}</h3>
                    <p>Today's Checks</p>
                </div>
            </div>
        </div>
        <div class="col-md-3">
            <div class="panel panel-danger">
                <div class="panel-body text-center">
                    <h3>{$blocked_today}</h3>
                    <p>Blocked Today</p>
                </div>
            </div>
        </div>
        <div class="col-md-3">
            <div class="panel panel-default">
                <div class="panel-body text-center">
                    <h3><a href="?module={fraud}&action=pending">View All</a></h3>
                    <p>Pending Checks</p>
                </div>
            </div>
        </div>
    </div>

    <div class="panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title">Recent Flagged Orders</h3>
        </div>
        <div class="panel-body">
            <table class="table table-striped">
                <thead>
                    <tr>
                        <th>Order ID</th>
                        <th>Email</th>
                        <th>IP Address</th>
                        <th>Risk Score</th>
                        <th>Risk Level</th>
                        <th>Status</th>
                        <th>Date</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {foreach $recent_flagged as $check}
                    <tr class="risk-row-{$check.risk_level}">
                        <td>{$check.order_id}</td>
                        <td>{$check.email|escape:'html'}</td>
                        <td>{$check.ip_address}</td>
                        <td>
                            <span class="badge {if $check.risk_score >= 70}danger{elseif $check.risk_score >= 40}warning{else}success{/if}">
                                {$check.risk_score}
                            </span>
                        </td>
                        <td><span class="label label-{$check.risk_level}">{$check.risk_level|upper}</span></td>
                        <td>{$check.status}</td>
                        <td>{$check.checked_at|date_format:"%Y-%m-d H:i"}</td>
                        <td>
                            <a href="?module={fraud}&action=review&id={$check.id}" class="btn btn-xs btn-primary">
                                Review
                            </a>
                        </td>
                    </tr>
                    {/foreach}
                    {if empty($recent_flagged)}
                    <tr>
                        <td colspan="8" class="text-center">No flagged orders</td>
                    </tr>
                    {/if}
                </tbody>
            </table>
        </div>
    </div>
</div>
```

## Database Schema

### mod_{fraud}_checks
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| order_id | VARCHAR(50) | WHMCS order ID |
| user_id | INT | Client ID |
| email | VARCHAR(255) | Customer email |
| ip_address | VARCHAR(45) | Customer IP |
| risk_score | DECIMAL(5,2) | Calculated risk score |
| risk_level | VARCHAR(20) | low/medium/high/critical |
| risk_factors | TEXT | JSON of triggered rules |
| status | VARCHAR(20) | pending/reviewed |
| api_response | TEXT | Raw API response |
| decision | VARCHAR(20) | approve/block/flag |
| notes | TEXT | Admin notes |
| checked_at | TIMESTAMP | Check timestamp |
| reviewed_at | TIMESTAMP | Review timestamp |
| reviewed_by | INT | Admin who reviewed |

### mod_{fraud}_rules
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| rule_name | VARCHAR(100) | Rule display name |
| rule_type | VARCHAR(50) | Rule type identifier |
| rule_config | TEXT | JSON configuration |
| risk_points | INT | Points to add |
| status | VARCHAR(20) | active/inactive |
| sort_order | INT | Processing order |

## Checklist

```
Pre-Dev:
□ Choose fraud detection API provider
□ Get API documentation and credentials
□ Plan risk scoring rules
□ Define risk thresholds
□ Design review workflow

Development:
□ Implement config() with all settings
□ Implement activate() → Create fraud tables
□ Implement deactivate() → Drop tables
□ Create FraudChecker class for API calls
□ Create RiskScorer class for rule evaluation
□ Implement admin output() with dashboard
□ Create fraud rules management
□ Add hooks for order integration
□ Implement review workflow
□ Add bulk actions
□ Create email alerts

Security:
□ Validate API responses
□ Sanitize all inputs
□ Log all fraud checks
□ Secure API key storage
□ Implement CSRF protection

Testing:
□ Test fraud check on new order
□ Test risk scoring rules
□ Test auto-block functionality
□ Test email alerts
□ Test review workflow
□ Test bulk actions
□ Verify hook integration
```
