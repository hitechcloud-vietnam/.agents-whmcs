# WHMCS Payment Routing Workflow

## Description
Configure smart payment routing to optimize transaction success and reduce costs.

## Prerequisites
- Multiple payment gateways
- Transaction analytics
- WHMCS 7.0+

## Steps

### Step 1: Create Payment Router
```php
<?php
/**
 * Smart Payment Router
 * Routes transactions to optimal gateway based on rules
 */

class PaymentRouter
{
    private $rules = [];
    private $gateways = [];
    
    public function __construct()
    {
        $this->loadRules();
        $this->loadGateways();
    }
    
    public function route($amount, $currency, $clientData)
    {
        foreach ($this->rules as $rule) {
            if ($this->evaluateRule($rule, $amount, $currency, $clientData)) {
                return $this->gateways[$rule['gateway']];
            }
        }
        
        // Default gateway
        return $this->gateways['default'];
    }
    
    private function evaluateRule($rule, $amount, $currency, $clientData)
    {
        // Amount range
        if (isset($rule['min_amount']) && $amount < $rule['min_amount']) {
            return false;
        }
        if (isset($rule['max_amount']) && $amount > $rule['max_amount']) {
            return false;
        }
        
        // Currency
        if (isset($rule['currencies']) && !in_array($currency, $rule['currencies'])) {
            return false;
        }
        
        // Client country
        if (isset($rule['countries']) && !in_array($clientData['country'], $rule['countries'])) {
            return false;
        }
        
        // Client tier
        if (isset($rule['client_tiers']) && !in_array($clientData['tier'], $rule['client_tiers'])) {
            return false;
        }
        
        // Card type (if known)
        if (isset($rule['card_types']) && isset($clientData['card_type'])) {
            if (!in_array($clientData['card_type'], $rule['card_types'])) {
                return false;
            }
        }
        
        return true;
    }
    
    private function loadRules()
    {
        $this->rules = [
            [
                'name' => 'High value - Stripe',
                'min_amount' => 500,
                'gateway' => 'stripe',
                'priority' => 1,
            ],
            [
                'name' => 'US cards - Stripe',
                'countries' => ['US'],
                'card_types' => ['visa', 'mastercard', 'amex'],
                'gateway' => 'stripe',
                'priority' => 2,
            ],
            [
                'name' => 'EU - SEPA',
                'countries' => ['DE', 'FR', 'ES', 'IT', 'NL'],
                'gateway' => 'sepa',
                'priority' => 2,
            ],
            [
                'name' => 'Vietnam - Local',
                'countries' => ['VN'],
                'gateway' => 'vnpay',
                'priority' => 2,
            ],
            [
                'name' => 'Default',
                'gateway' => 'paypal',
                'priority' => 99,
            ],
        ];
        
        // Sort by priority
        usort($this->rules, fn($a, $b) => $a['priority'] <=> $b['priority']);
    }
    
    private function loadGateways()
    {
        $this->gateways = [
            'stripe' => getGatewayVariables('stripe'),
            'paypal' => getGatewayVariables('paypal'),
            'vnpay' => getGatewayVariables('vnpay'),
            'sepa' => getGatewayVariables('sepa_direct_debit'),
            'default' => getGatewayVariables('paypal'),
        ];
    }
}
```

### Step 2: Integrate Router
```php
<?php
// In payment gateway module

function smart_payment_link($params)
{
    $router = new PaymentRouter();
    
    $clientData = [
        'country' => $params['clientdetails']['country'],
        'tier' => getClientTier($params['clientdetails']['id']),
        'card_type' => $_POST['card_type'] ?? null,
    ];
    
    $gateway = $router->route(
        $params['amount'],
        $params['currency'],
        $clientData
    );
    
    // Use the selected gateway
    return call_user_func($gateway['function'] . '_link', $params);
}
```

### Step 3: Cost-Based Routing
```php
<?php
class CostBasedRouter extends PaymentRouter
{
    private $gatewayFees = [
        'stripe' => ['fixed' => 0.30, 'percent' => 2.9],
        'paypal' => ['fixed' => 0.30, 'percent' => 3.5],
        'vnpay' => ['fixed' => 0, 'percent' => 2.0],
    ];
    
    public function getCheapestGateway($amount)
    {
        $costs = [];
        
        foreach ($this->gatewayFees as $gateway => $fee) {
            $costs[$gateway] = $fee['fixed'] + ($amount * $fee['percent'] / 100);
        }
        
        asort($costs);
        
        return array_key_first($costs);
    }
}
```

### Step 4: Success Rate Routing
```php
<?php
function getGatewaySuccessRate($gatewayName)
{
    $result = Capsule::table('mod_gateway_stats')
        ->where('gateway', $gatewayName)
        ->where('date', '>=', date('Y-m-d', strtotime('-30 days')))
        ->selectRaw('
            SUM(CASE WHEN status = "success" THEN 1 ELSE 0 END) as successes,
            COUNT(*) as total
        ')
        ->first();
    
    return $result->total > 0 
        ? ($result->successes / $result->total) * 100 
        : 100;
}

function routeBySuccessRate($amount)
{
    $gateways = ['stripe', 'paypal', 'vnpay'];
    $rates = [];
    
    foreach ($gateways as $gateway) {
        $rates[$gateway] = getGatewaySuccessRate($gateway);
    }
    
    // Use gateway with highest success rate
    arsort($rates);
    
    return array_key_first($rates);
}
```

## Routing Strategies
| Strategy | Description |
|----------|-------------|
| Cost-based | Route to cheapest gateway |
| Success-rate | Route to highest success gateway |
| Geographic | Route by client location |
| Volume-based | Route based on amount tiers |
| Failover | Route to backup on primary failure |

## Tags
- payment-routing
- smart-routing
- optimization
- gateway