# WHMCS Fraud Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-fraud-module/
├── fraud.php            # Fraud detection engine
├── lib/
│   ├── FraudChecker.php  # Checker implementations
│   ├── RuleEngine.php   # Rule processing
│   └── RiskScorer.php   # Risk scoring
└── templates/
    └── admin.tpl         # Admin interface
```

## Fraud Detection Engine Template

```php
<?php
/**
 * WHMCS Fraud Module: {Module}
 * DevKit Template
 * 
 * Installation: Upload to modules/addons/{module}/
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {module}_config(): array {
    return [
        'name' => '{Fraud Module}',
        'description' => 'Fraud detection and risk assessment',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Activate
 */
function {module}_activate(): array {
    Capsule::schema()->create('mod_{module}_fraud_rules', function($t) {
        $t->increments('id');
        $t->string('rule_name');
        $t->string('rule_type');
        $t->text('rule_config');
        $t->integer('risk_score');
        $t->boolean('is_active');
        $t->integer('priority');
    });
    
    Capsule::schema()->create('mod_{module}_fraud_logs', function($t) {
        $t->increments('id');
        $t->string('check_type');
        $t->integer('user_id');
        $t->integer('order_id');
        $t->integer('risk_score');
        $t->string('risk_level');
        $t->text('details');
        $t->text('recommendation');
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_fraud_blocklist', function($t) {
        $t->increments('id');
        $t->string('type'); // email, ip, phone
        $t->string('value');
        $t->string('reason');
        $t->timestamp('created_at');
    });
    
    // Insert default rules
    {module}_insertDefaultRules();
    
    return ['status' => 'success', 'description' => 'Module activated'];
}

/**
 * Insert Default Rules
 */
function {module}_insertDefaultRules(): void {
    $defaultRules = [
        [
            'rule_name' => 'Free Email Provider',
            'rule_type' => 'email_domain',
            'rule_config' => json_encode(['domains' => ['gmail.com', 'yahoo.com', 'hotmail.com', 'outlook.com']]),
            'risk_score' => 5,
            'is_active' => true,
            'priority' => 10,
        ],
        [
            'rule_name' => 'High Risk Country',
            'rule_type' => 'country',
            'rule_config' => json_encode(['countries' => ['RU', 'NG', 'UA', 'CN']]),
            'risk_score' => 15,
            'is_active' => true,
            'priority' => 20,
        ],
        [
            'rule_name' => 'Disposable Email',
            'rule_type' => 'email_disposable',
            'rule_config' => json_encode([]),
            'risk_score' => 25,
            'is_active' => true,
            'priority' => 15,
        ],
        [
            'rule_name' => 'VPN/Proxy Detection',
            'rule_type' => 'ip_vpn_proxy',
            'rule_config' => json_encode([]),
            'risk_score' => 20,
            'is_active' => true,
            'priority' => 25,
        ],
        [
            'rule_name' => 'Billing/Shipping Mismatch',
            'rule_type' => 'address_mismatch',
            'rule_config' => json_encode([]),
            'risk_score' => 15,
            'is_active' => true,
            'priority' => 30,
        ],
        [
            'rule_name' => 'High Value Order',
            'rule_type' => 'order_value',
            'rule_config' => json_encode(['threshold' => 500]),
            'risk_score' => 10,
            'is_active' => true,
            'priority' => 40,
        ],
        [
            'rule_name' => 'Multiple Failed Payments',
            'rule_type' => 'failed_payments',
            'rule_config' => json_encode(['threshold' => 3]),
            'risk_score' => 25,
            'is_active' => true,
            'priority' => 35,
        ],
        [
            'rule_name' => 'New Account High Value',
            'rule_type' => 'new_account_high_value',
            'rule_config' => json_encode(['age_days' => 7, 'amount_threshold' => 200]),
            'risk_score' => 30,
            'is_active' => true,
            'priority' => 45,
        ],
    ];
    
    foreach ($defaultRules as $rule) {
        Capsule::table('mod_{module}_fraud_rules')->insert($rule);
    }
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_fraud_rules');
    Capsule::schema()->dropIfExists('mod_{module}_fraud_logs');
    Capsule::schema()->dropIfExists('mod_{module}_fraud_blocklist');
    
    return ['status' => 'success'];
}

/**
 * Output function
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
    
    switch ($action) {
        case 'rules':
            {module}_manageRules();
            break;
        case 'logs':
            {module}_showLogs();
            break;
        case 'blocklist':
            {module}_manageBlocklist();
            break;
        case 'settings':
            {module}_showSettings();
            break;
        case 'analyze':
            {module}_analyzeOrder();
            break;
        default:
            {module}_showDashboard();
    }
}

/**
 * Show Dashboard
 */
function {module}_showDashboard(): void {
    $stats = [
        'total_checks' => Capsule::table('mod_{module}_fraud_logs')->count(),
        'high_risk' => Capsule::table('mod_{module}_fraud_logs')
            ->where('risk_level', 'high')->count(),
        'blocked_today' => Capsule::table('mod_{module}_fraud_logs')
            ->whereDate('created_at', date('Y-m-d'))
            ->where('risk_level', 'high')->count(),
        'total_rules' => Capsule::table('mod_{module}_fraud_rules')
            ->where('is_active', 1)->count(),
    ];
    
    echo <<<HTML
<div class="fraud-module">
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Fraud Detection Dashboard</h3>
                </div>
                <div class="panel-body">
                    <div class="row">
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['total_checks']}</div>
                                <div class="stat-label">Total Checks</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['high_risk']}</div>
                                <div class="stat-label">High Risk Flags</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['blocked_today']}</div>
                                <div class="stat-label">Blocked Today</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['total_rules']}</div>
                                <div class="stat-label">Active Rules</div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <div class="row">
        <div class="col-md-12">
            <div class="btn-group">
                <a href="?module={module}&action=rules" class="btn btn-primary">
                    <i class="fa fa-gavel"></i> Manage Rules
                </a>
                <a href="?module={module}&action=blocklist" class="btn btn-default">
                    <i class="fa fa-ban"></i> Blocklist
                </a>
                <a href="?module={module}&action=logs" class="btn btn-default">
                    <i class="fa fa-file-alt"></i> Check Logs
                </a>
                <a href="?module={module}&action=analyze" class="btn btn-default">
                    <i class="fa fa-search"></i> Manual Analysis
                </a>
            </div>
        </div>
    </div>
</div>
HTML;
}
```

## Risk Scorer Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class RiskScorer {
    
    private array $checks = [];
    private int $totalScore = 0;
    private array $triggeredRules = [];
    
    public const RISK_LEVEL_LOW = 'low';
    public const RISK_LEVEL_MEDIUM = 'medium';
    public const RISK_LEVEL_HIGH = 'high';
    
    public const THRESHOLD_LOW = 20;
    public const THRESHOLD_MEDIUM = 40;
    public const THRESHOLD_HIGH = 60;
    
    public function addCheck(string $ruleName, int $score, array $details = []): void {
        $this->checks[] = [
            'rule' => $ruleName,
            'score' => $score,
            'details' => $details,
        ];
        $this->totalScore += $score;
        
        if ($score > 0) {
            $this->triggeredRules[] = $ruleName;
        }
    }
    
    public function getScore(): int {
        return min($this->totalScore, 100);
    }
    
    public function getRiskLevel(): string {
        $score = $this->getScore();
        
        if ($score >= self::THRESHOLD_HIGH) {
            return self::RISK_LEVEL_HIGH;
        } elseif ($score >= self::THRESHOLD_MEDIUM) {
            return self::RISK_LEVEL_MEDIUM;
        }
        
        return self::RISK_LEVEL_LOW;
    }
    
    public function getRecommendation(): string {
        $level = $this->getRiskLevel();
        
        return match($level) {
            self::RISK_LEVEL_HIGH => 'Block order and require manual review',
            self::RISK_LEVEL_MEDIUM => 'Flag for review and request additional verification',
            default => 'Approve order',
        };
    }
    
    public function getChecks(): array {
        return $this->checks;
    }
    
    public function getTriggeredRules(): array {
        return $this->triggeredRules;
    }
    
    public function shouldBlock(): bool {
        return $this->getRiskLevel() === self::RISK_LEVEL_HIGH;
    }
    
    public function shouldReview(): bool {
        return in_array($this->getRiskLevel(), [
            self::RISK_LEVEL_HIGH,
            self::RISK_LEVEL_MEDIUM,
        ]);
    }
}

/**
 * Order Check Hook
 */
function {module}_checkOrder(array $vars): array {
    $orderId = $vars['orderid'] ?? 0;
    $userId = $vars['userid'] ?? 0;
    
    if (!$orderId) {
        return ['proceed' => true];
    }
    
    $checker = new \{Module}\FraudChecker($userId, $orderId);
    $result = $checker->runAllChecks();
    
    // Log the check
    Capsule::table('mod_{module}_fraud_logs')->insert([
        'check_type' => 'order',
        'user_id' => $userId,
        'order_id' => $orderId,
        'risk_score' => $result['score'],
        'risk_level' => $result['level'],
        'details' => json_encode($result['checks']),
        'recommendation' => $result['recommendation'],
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Return based on risk level
    if ($result['block']) {
        return [
            'proceed' => false,
            'reason' => 'High fraud risk detected',
            'details' => $result,
        ];
    }
    
    return ['proceed' => true, 'details' => $result];
}

// Register hook
add_hook('AcceptOrder', 1, function($vars) {
    $result = {module}_checkOrder($vars);
    
    if (!$result['proceed']) {
        throw new \Exception('Order blocked due to fraud risk');
    }
});
```

## Fraud Checker Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class FraudChecker {
    
    private int $userId;
    private int $orderId;
    private RiskScorer $scorer;
    
    private const FREE_EMAIL_DOMAINS = [
        'gmail.com', 'yahoo.com', 'hotmail.com', 'outlook.com',
        'aol.com', 'icloud.com', 'mail.com', 'protonmail.com',
    ];
    
    private const HIGH_RISK_COUNTRIES = [
        'RU', 'NG', 'UA', 'CN', 'VN', 'PK', 'BD', 'ID',
    ];
    
    private const DISPOSABLE_DOMAINS = [
        'tempmail.com', 'throwaway.email', 'guerrillamail.com',
        'mailinator.com', '10minutemail.com', 'temp-mail.org',
    ];
    
    public function __construct(int $userId, int $orderId) {
        $this->userId = $userId;
        $this->orderId = $orderId;
        $this->scorer = new RiskScorer();
    }
    
    public function runAllChecks(): array {
        $user = $this->getUser();
        $order = $this->getOrder();
        
        $this->checkFreeEmailDomain($user['email'] ?? '');
        $this->checkCountryRisk($user['country'] ?? '');
        $this->checkDisposableEmail($user['email'] ?? '');
        $this->checkIpVpnProxy();
        $this->checkAddressMismatch($user, $order);
        $this->checkOrderValue($order['amount'] ?? 0);
        $this->checkFailedPayments();
        $this->checkNewAccountHighValue();
        $this->checkBlocklist($user, $order);
        
        return [
            'score' => $this->scorer->getScore(),
            'level' => $this->scorer->getRiskLevel(),
            'checks' => $this->scorer->getChecks(),
            'triggered_rules' => $this->scorer->getTriggeredRules(),
            'recommendation' => $this->scorer->getRecommendation(),
            'block' => $this->scorer->shouldBlock(),
        ];
    }
    
    private function getUser(): array {
        return Capsule::table('tblclients')
            ->where('id', $this->userId)
            ->first() ? (array) Capsule::table('tblclients')
            ->where('id', $this->userId)->first() : [];
    }
    
    private function getOrder(): array {
        return Capsule::table('tblorders')
            ->where('id', $this->orderId)
            ->first() ? (array) Capsule::table('tblorders')
            ->where('id', $this->orderId)->first() : [];
    }
    
    private function checkFreeEmailDomain(string $email): void {
        if (empty($email)) return;
        
        $domain = strtolower(substr($email, strpos($email, '@') + 1));
        
        if (in_array($domain, self::FREE_EMAIL_DOMAINS)) {
            $this->scorer->addCheck('Free Email Domain', 5, [
                'domain' => $domain,
            ]);
        }
    }
    
    private function checkCountryRisk(string $country): void {
        if (empty($country)) return;
        
        if (in_array($country, self::HIGH_RISK_COUNTRIES)) {
            $this->scorer->addCheck('High Risk Country', 15, [
                'country' => $country,
            ]);
        }
    }
    
    private function checkDisposableEmail(string $email): void {
        if (empty($email)) return;
        
        $domain = strtolower(substr($email, strpos($email, '@') + 1));
        
        if (in_array($domain, self::DISPOSABLE_DOMAINS)) {
            $this->scorer->addCheck('Disposable Email', 30, [
                'domain' => $domain,
            ]);
        }
    }
    
    private function checkIpVpnProxy(): void {
        $ip = $_SERVER['REMOTE_ADDR'] ?? '';
        
        // Check against known VPN/proxy services
        // This is a simplified check - production should use a VPN detection API
        $vpnRanges = [
            '185.220.', // Example VPN range
        ];
        
        foreach ($vpnRanges as $range) {
            if (strpos($ip, $range) === 0) {
                $this->scorer->addCheck('VPN/Proxy Detected', 25, [
                    'ip' => $ip,
                ]);
                return;
            }
        }
    }
    
    private function checkAddressMismatch(array $user, array $order): void {
        // Check if billing and shipping addresses differ significantly
        // This is a simplified implementation
        if (empty($user['address1']) || empty($order['ip'])) {
            return;
        }
        
        // For demonstration - in production, implement proper address comparison
    }
    
    private function checkOrderValue(float $amount): void {
        if ($amount >= 500) {
            $this->scorer->addCheck('High Value Order', 10, [
                'amount' => $amount,
            ]);
        }
    }
    
    private function checkFailedPayments(): void {
        $failedCount = Capsule::table('tblaccounts')
            ->where('userid', $this->userId)
            ->where('failurereason', '!=', '')
            ->count();
        
        if ($failedCount >= 3) {
            $this->scorer->addCheck('Multiple Failed Payments', 25, [
                'count' => $failedCount,
            ]);
        }
    }
    
    private function checkNewAccountHighValue(): void {
        $user = $this->getUser();
        $order = $this->getOrder();
        
        if (empty($user['created_at']) || empty($order['amount'])) {
            return;
        }
        
        $accountAge = (time() - strtotime($user['created_at'])) / 86400;
        $orderAmount = (float) $order['amount'];
        
        if ($accountAge <= 7 && $orderAmount >= 200) {
            $this->scorer->addCheck('New Account High Value Order', 30, [
                'account_age_days' => round($accountAge),
                'order_amount' => $orderAmount,
            ]);
        }
    }
    
    private function checkBlocklist(array $user, array $order): void {
        // Check email blocklist
        if (!empty($user['email'])) {
            $blocked = Capsule::table('mod_{module}_fraud_blocklist')
                ->where('type', 'email')
                ->where('value', $user['email'])
                ->first();
            
            if ($blocked) {
                $this->scorer->addCheck('Email Blocklisted', 100, [
                    'reason' => $blocked->reason,
                ]);
            }
        }
        
        // Check IP blocklist
        if (!empty($order['ip'])) {
            $blocked = Capsule::table('mod_{module}_fraud_blocklist')
                ->where('type', 'ip')
                ->where('value', $order['ip'])
                ->first();
            
            if ($blocked) {
                $this->scorer->addCheck('IP Blocklisted', 100, [
                    'reason' => $blocked->reason,
                ]);
            }
        }
    }
}
```

## Checklist

```
Pre-Dev:
□ Identify fraud detection rules
□ Plan risk scoring algorithm
□ Design rule configuration
□ Identify external data sources (IP databases, etc.)
□ Plan blocklist management

Development:
□ Create fraud detection tables
□ Implement RiskScorer class
□ Implement FraudChecker class
□ Add email domain checks
□ Add country risk checks
□ Add VPN/proxy detection
□ Add disposable email detection
□ Add address mismatch check
□ Add high value order check
□ Add failed payments check
□ Create admin rule management
□ Add blocklist functionality
□ Create AcceptOrder hook

Testing:
□ Test risk scoring algorithm
□ Test all rule types
□ Test blocklist detection
□ Test hook integration
□ Verify threshold settings
□ Test with various scenarios
□ Test false positive rates
```