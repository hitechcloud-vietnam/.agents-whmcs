# WHMCS Fraud Detection Workflow

## Purpose

Comprehensive guide to implementing fraud detection systems in WHMCS to identify and prevent fraudulent orders, payments, and account activities.

## Prerequisites

- WHMCS installation
- Understanding of fraud patterns
- Database access for custom rules
- Optional: Third-party fraud detection API

## Workflow Steps

### Step 1: Fraud Detection Rules Engine

Implement configurable fraud rules:

```php
// modules/addons/fraud_detection/engine.php

class FraudDetectionEngine
{
    private $rules = [];
    private $riskScore = 0;
    private $flags = [];
    
    public function __construct()
    {
        $this->loadRules();
    }
    
    /**
     * Load fraud detection rules
     */
    private function loadRules(): void
    {
        $this->rules = [
            // IP-based rules
            [
                'type' => 'ip_address',
                'condition' => 'is_in_blacklist',
                'weight' => 100,
                'description' => 'IP address in blacklist',
            ],
            [
                'type' => 'ip_address',
                'condition' => 'is_proxy',
                'weight' => 40,
                'description' => 'IP address is a known proxy/VPN',
            ],
            [
                'type' => 'ip_address',
                'condition' => 'is_tor',
                'weight' => 50,
                'description' => 'IP address from Tor exit node',
            ],
            [
                'type' => 'ip_address',
                'condition' => 'different_country',
                'weight' => 30,
                'description' => 'IP country differs from billing country',
            ],
            
            // Email-based rules
            [
                'type' => 'email',
                'condition' => 'is_disposable',
                'weight' => 60,
                'description' => 'Disposable email domain',
            ],
            [
                'type' => 'email',
                'condition' => 'is_free_provider',
                'weight' => 10,
                'description' => 'Free email provider (lower risk)',
            ],
            [
                'type' => 'email',
                'condition' => 'domain_age_low',
                'weight' => 25,
                'description' => 'Email domain registered recently',
            ],
            
            // Order-based rules
            [
                'type' => 'order',
                'condition' => 'high_value',
                'threshold' => 500,
                'weight' => 25,
                'description' => 'Order value exceeds threshold',
            ],
            [
                'type' => 'order',
                'condition' => 'multiple_cards',
                'threshold' => 3,
                'weight' => 45,
                'description' => 'Multiple different cards used',
            ],
            [
                'type' => 'order',
                'condition' => 'velocity_high',
                'threshold' => 3,
                'timeframe' => '24h',
                'weight' => 40,
                'description' => 'Too many orders in timeframe',
            ],
            
            // Account-based rules
            [
                'type' => 'account',
                'condition' => 'new_account',
                'threshold' => 48, // hours
                'weight' => 20,
                'description' => 'Account created within timeframe',
            ],
            [
                'type' => 'account',
                'condition' => 'failed_payments',
                'threshold' => 3,
                'weight' => 35,
                'description' => 'Multiple failed payment attempts',
            ],
            [
                'type' => 'account',
                'condition' => 'no_history',
                'weight' => 15,
                'description' => 'Client has no payment history',
            ],
            
            // Shipping rules
            [
                'type' => 'shipping',
                'condition' => 'high_risk_country',
                'countries' => ['NG', 'GH', 'VN', 'ID', 'PK'], // High-risk countries
                'weight' => 30,
                'description' => 'Shipping to high-risk country',
            ],
            [
                'type' => 'shipping',
                'condition' => 'billing_mismatch',
                'weight' => 45,
                'description' => 'Shipping address differs significantly',
            ],
            [
                'type' => 'shipping',
                'condition' => 'freight_forwarder',
                'weight' => 35,
                'description' => 'Shipping to freight forwarder',
            ],
        ];
    }
    
    /**
     * Analyze order for fraud indicators
     */
    public function analyzeOrder(array $orderData): array
    {
        $this->riskScore = 0;
        $this->flags = [];
        
        // Run all applicable rules
        foreach ($this->rules as $rule) {
            $result = $this->evaluateRule($rule, $orderData);
            
            if ($result['matched']) {
                $this->riskScore += $rule['weight'];
                $this->flags[] = [
                    'rule' => $rule['description'],
                    'weight' => $rule['weight'],
                    'details' => $result['details'] ?? null,
                ];
            }
        }
        
        // Determine risk level
        $riskLevel = $this->determineRiskLevel($this->riskScore);
        
        // Store fraud check result
        $checkId = $this->storeCheckResult($orderData, $riskLevel);
        
        return [
            'order_id' => $orderData['order_id'],
            'risk_score' => $this->riskScore,
            'risk_level' => $riskLevel,
            'flags' => $this->flags,
            'recommendation' => $this->getRecommendation($riskLevel),
            'check_id' => $checkId,
        ];
    }
    
    /**
     * Evaluate single rule against order data
     */
    private function evaluateRule(array $rule, array $data): array
    {
        $matched = false;
        $details = null;
        
        switch ($rule['condition']) {
            case 'is_in_blacklist':
                $matched = $this->checkIpBlacklist($data['ip_address']);
                break;
                
            case 'is_proxy':
                $matched = $this->checkProxy($data['ip_address']);
                $details = $matched ? 'Proxy/VPN detected' : null;
                break;
                
            case 'is_disposable':
                $matched = $this->checkDisposableEmail($data['email']);
                break;
                
            case 'high_value':
                $matched = $data['total'] >= ($rule['threshold'] ?? 500);
                break;
                
            case 'velocity_high':
                $matched = $this->checkOrderVelocity($data['email'], $rule['threshold'], $rule['timeframe']);
                break;
                
            case 'new_account':
                $age = $this->getAccountAge($data['user_id']);
                $matched = $age <= ($rule['threshold'] ?? 48);
                break;
                
            case 'high_risk_country':
                $matched = in_array($data['country'], $rule['countries'] ?? []);
                break;
                
            case 'billing_mismatch':
                $matched = $this->checkAddressMismatch($data);
                break;
        }
        
        return ['matched' => $matched, 'details' => $details];
    }
    
    /**
     * Determine risk level based on score
     */
    private function determineRiskLevel(int $score): string
    {
        if ($score >= 80) return 'critical';
        if ($score >= 60) return 'high';
        if ($score >= 40) return 'medium';
        if ($score >= 20) return 'low';
        return 'minimal';
    }
    
    /**
     * Get recommendation based on risk level
     */
    private function getRecommendation(string $riskLevel): string
    {
        $recommendations = [
            'critical' => 'auto_decline',
            'high' => 'manual_review_required',
            'medium' => 'verify_customer',
            'low' => 'approve_with_monitoring',
            'minimal' => 'auto_approve',
        ];
        
        return $recommendations[$riskLevel];
    }
    
    /**
     * Store fraud check result
     */
    private function storeCheckResult(array $data, string $riskLevel): int
    {
        return Capsule::table('mod_fraud_checks')->insertGetId([
            'order_id' => $data['order_id'] ?? null,
            'user_id' => $data['user_id'] ?? null,
            'ip_address' => $data['ip_address'] ?? null,
            'email' => $data['email'] ?? null,
            'risk_score' => $this->riskScore,
            'risk_level' => $riskLevel,
            'flags' => json_encode($this->flags),
            'recommendation' => $this->getRecommendation($riskLevel),
            'reviewed' => 0,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

### Step 2: Fraud Detection Hooks

Integrate fraud checks into order flow:

```php
// modules/addons/fraud_detection/hooks.php

/**
 * Hook: Before order completion
 */
add_hook('ShoppingCartCheckoutCompletePage', 1, function($vars) {
    $fraudEngine = new FraudDetectionEngine();
    
    $orderData = [
        'order_id' => $vars['order_id'],
        'user_id' => $_SESSION['uid'],
        'email' => $_SESSION['uid'] ? getClientEmail($_SESSION['uid']) : $_POST['email'],
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'country' => $_POST['country'] ?? '',
        'total' => $vars['total'],
    ];
    
    $result = $fraudEngine->analyzeOrder($orderData);
    
    // Store in session for later reference
    $_SESSION['fraud_check_result'] = $result;
    
    // Handle based on risk level
    switch ($result['risk_level']) {
        case 'critical':
            // Cancel order
            redirect('fraud-blocked.php');
            break;
            
        case 'high':
            // Flag for review, don't complete yet
            $_SESSION['fraud_pending'] = true;
            break;
            
        case 'medium':
            // Require additional verification
            redirect('verify-identity.php');
            break;
    }
});

/**
 * Hook: After order creation
 */
add_hook('OrderCreated', 1, function($vars) {
    $orderId = $vars['order_id'];
    
    // Check if fraud flagged
    $fraudCheck = Capsule::table('mod_fraud_checks')
        ->where('order_id', $orderId)
        ->orderBy('created_at', 'desc')
        ->first();
    
    if ($fraudCheck && $fraudCheck->risk_level === 'high') {
        // Mark order for review
        Capsule::table('tblorders')
            ->where('id', $orderId)
            ->update([
                'status' => 'Pending',
                'notes' => "Fraud review required. Risk score: {$fraudCheck->risk_score}",
            ]);
        
        // Alert admin
        sendAdminNotification('fraud_alert', [
            'order_id' => $orderId,
            'risk_score' => $fraudCheck->risk_score,
            'flags' => $fraudCheck->flags,
        ]);
    }
});

/**
 * Hook: Before payment processing
 */
add_hook('PrePaymentProcessing', 1, function($vars) {
    if (!isset($_SESSION['fraud_check_result'])) {
        return;
    }
    
    $result = $_SESSION['fraud_check_result'];
    
    if ($result['risk_level'] === 'critical') {
        return [
            'abort' => true,
            'reason' => 'Order blocked due to fraud risk',
        ];
    }
});
```

### Step 3: Third-Party Integration

Integrate with fraud detection APIs:

```php
// modules/addons/fraud_detection/third_party.php

class ThirdPartyFraudService
{
    private $apiKey;
    private $baseUrl;
    
    public function __construct(string $apiKey)
    {
        $this->apiKey = $apiKey;
        $this->baseUrl = 'https://api.fraudservice.example.com/v1';
    }
    
    /**
     * Check order with external service
     */
    public function checkOrder(array $orderData): array
    {
        $payload = $this->buildPayload($orderData);
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . '/check',
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
            CURLOPT_TIMEOUT => 10,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode !== 200) {
            logActivity("Fraud API error: HTTP {$httpCode}");
            return ['error' => true];
        }
        
        $result = json_decode($response, true);
        
        // Store response
        $this->storeApiResponse($orderData, $result);
        
        return $result;
    }
    
    /**
     * Build API request payload
     */
    private function buildPayload(array $orderData): array
    {
        return [
            'order' => [
                'id' => $orderData['order_id'],
                'amount' => $orderData['total'],
                'currency' => 'USD',
                'created_at' => date('c'),
            ],
            'customer' => [
                'id' => $orderData['user_id'],
                'email' => $orderData['email'],
                'phone' => $orderData['phone'] ?? null,
                'billing_address' => [
                    'country' => $orderData['country'],
                    'state' => $orderData['state'] ?? null,
                    'city' => $orderData['city'],
                    'postal_code' => $orderData['postcode'],
                    'ip_address' => $orderData['ip_address'],
                ],
            ],
            'payment' => [
                'method' => $orderData['payment_method'],
                'card_bin' => $orderData['card_bin'] ?? null,
                'card_last4' => $orderData['card_last4'] ?? null,
            ],
        ];
    }
    
    /**
     * Report fraud confirmation back to service
     */
    public function reportFraud(string $orderId): void
    {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . '/report',
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode(['order_id' => $orderId, 'fraud' => true]),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
            ],
        ]);
        
        curl_exec($ch);
        curl_close($ch);
    }
}
```

### Step 4: Admin Fraud Review Interface

Create fraud review dashboard:

```php
// modules/addons/fraud_detection/admin.php

function fraud_detection_output(array $vars): void
{
    $action = $_REQUEST['action'] ?? 'pending';
    
    switch ($action) {
        case 'pending':
            echo renderPendingReviews();
            break;
        case 'review':
            echo renderReviewDetail($_GET['id']);
            break;
        case 'approve':
            approveOrder($_GET['id']);
            redir('action=pending');
            break;
        case 'decline':
            declineOrder($_GET['id']);
            redir('action=pending');
            break;
        case 'blacklist':
            addToBlacklist($_GET['id']);
            redir('action=pending');
            break;
    }
}

function renderPendingReviews(): string
{
    $pending = Capsule::table('mod_fraud_checks')
        ->where('reviewed', 0)
        ->whereIn('risk_level', ['critical', 'high'])
        ->orderBy('risk_score', 'desc')
        ->limit(50)
        ->get();
    
    $html = '<div class="fraud-review">';
    $html .= '<h2>Pending Fraud Reviews</h2>';
    $html .= '<table class="datatable"><thead><tr>';
    $html .= '<th>Order ID</th><th>Email</th><th>IP Address</th>';
    $html .= '<th>Risk Score</th><th>Risk Level</th><th>Date</th><th>Actions</th>';
    $html .= '</tr></thead><tbody>';
    
    foreach ($pending as $check) {
        $levelClass = [
            'critical' => 'danger',
            'high' => 'warning',
            'medium' => 'info',
        ][$check->risk_level] ?? 'default';
        
        $html .= '<tr>';
        $html .= '<td><a href="orders.php?action=view&id=' . $check->order_id . '">#' . $check->order_id . '</a></td>';
        $html .= '<td>' . htmlspecialchars($check->email) . '</td>';
        $html .= '<td>' . htmlspecialchars($check->ip_address) . '</td>';
        $html .= '<td>' . $check->risk_score . '</td>';
        $html .= '<td><span class="label label-' . $levelClass . '">' . ucfirst($check->risk_level) . '</span></td>';
        $html .= '<td>' . $check->created_at . '</td>';
        $html .= '<td>';
        $html .= '<a href="?action=review&id=' . $check->id . '" class="btn btn-xs btn-default">Review</a> ';
        $html .= '<a href="?action=blacklist&id=' . $check->id . '" class="btn btn-xs btn-danger">Blacklist</a>';
        $html .= '</td>';
        $html .= '</tr>';
    }
    
    $html .= '</tbody></table></div>';
    
    return $html;
}
```

## Best Practices

1. **Layer detection** - Multiple fraud checks at different stages
2. **Real-time checks** - Don't wait for payment to detect fraud
3. **Keep rules updated** - Fraud patterns evolve
4. **Balance friction and security** - Don't block too many legitimate orders
5. **Monitor false positives** - Track approved orders later flagged
6. **Share intelligence** - Report confirmed fraud
7. **Machine learning** - Consider ML models for patterns
8. **Manual review queue** - Human judgment for borderline cases

## Common Pitfalls to Avoid

1. **Too aggressive** - Blocking legitimate customers
2. **Too lenient** - Missing fraud
3. **Static rules** - Not updating for new patterns
4. **Ignoring feedback** - Not learning from confirmed fraud
5. **Privacy violations** - Collecting too much data
6. **Slow checks** - Delaying order processing
7. **No documentation** - Can't explain decisions
8. **Single point of failure** - Relying on one detection method
