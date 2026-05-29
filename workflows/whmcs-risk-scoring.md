# WHMCS Risk Scoring Workflow

## Description
Implement risk scoring system for transactions and customer accounts.

## Steps

### Step 1: Create Risk Scoring Engine
```php
<?php
/**
 * Risk Scoring Engine
 * Evaluates transactions and customers for risk levels
 */

class RiskScoringEngine
{
    private $weights = [];
    
    public function __construct()
    {
        $this->weights = [
            'email_quality' => 15,
            'address_verification' => 20,
            'ip_location' => 25,
            'device_fingerprint' => 20,
            'velocity' => 30,
            'historical' => 25,
            'geographic' => 20,
        ];
    }
    
    /**
     * Calculate overall risk score
     */
    public function calculateScore($params)
    {
        $scores = [];
        
        $scores['email'] = $this->scoreEmail($params['email']);
        $scores['address'] = $this->scoreAddress($params);
        $scores['ip'] = $this->scoreIPLocation($params['ip'], $params['country']);
        $scores['device'] = $this->scoreDevice($params);
        $scores['velocity'] = $this->scoreVelocity($params['user_id']);
        $scores['history'] = $this->scoreHistory($params['user_id']);
        $scores['geo'] = $this->scoreGeography($params['country']);
        
        // Calculate weighted score
        $totalWeight = array_sum($this->weights);
        $weightedScore = 0;
        
        foreach ($scores as $factor => $score) {
            $weight = $this->weights[$factor] ?? 0;
            $weightedScore += ($score * $weight / $totalWeight);
        }
        
        return [
            'total_score' => round($weightedScore, 2),
            'max_score' => 100,
            'risk_level' => $this->getRiskLevel($weightedScore),
            'factors' => $scores,
            'recommendations' => $this->getRecommendations($scores),
        ];
    }
    
    private function scoreEmail($email)
    {
        $score = 0;
        
        // Free email providers
        $freeDomains = ['gmail.com', 'yahoo.com', 'hotmail.com', 'outlook.com'];
        $domain = substr(strrchr($email, '@'), 1);
        
        if (in_array(strtolower($domain), $freeDomains)) {
            $score += 30;
        }
        
        // Disposable email
        if ($this->isDisposableEmail($email)) {
            $score += 50;
        }
        
        // Email age (would need third-party data)
        return min($score, 100);
    }
    
    private function scoreAddress($params)
    {
        $score = 0;
        
        if (empty($params['address1'])) {
            $score += 20;
        }
        
        if (empty($params['city']) || empty($params['state'])) {
            $score += 15;
        }
        
        if (empty($params['postcode'])) {
            $score += 15;
        }
        
        return min($score, 100);
    }
    
    private function scoreIPLocation($ip, $country)
    {
        $ipCountry = $this->getCountryFromIP($ip);
        
        if ($ipCountry && $ipCountry !== $country) {
            return 100; // High risk - country mismatch
        }
        
        if ($this->isVPN($ip)) {
            return 80;
        }
        
        if ($this->isProxy($ip)) {
            return 70;
        }
        
        return 0;
    }
    
    private function getRiskLevel($score)
    {
        if ($score >= 80) return 'critical';
        if ($score >= 60) return 'high';
        if ($score >= 40) return 'medium';
        if ($score >= 20) return 'low';
        return 'minimal';
    }
}
```

### Step 2: Store Risk Profiles
```php
<?php
// Database table for risk profiles

Capsule::schema()->create('mod_risk_profiles', function($table) {
    $table->increments('id');
    $table->integer('user_id');
    $table->decimal('risk_score', 5, 2)->default(0);
    $table->string('risk_level', 20);
    $table->json('risk_factors');
    $table->timestamp('last_calculated');
    $table->integer('total_transactions');
    $table->integer('flagged_transactions');
});

function updateRiskProfile($userId)
{
    $client = Capsule::table('tblclients')->find($userId);
    $scorer = new RiskScoringEngine();
    
    $result = $scorer->calculateScore([
        'email' => $client->email,
        'country' => $client->country,
        'address1' => $client->address1,
        'city' => $client->city,
        'state' => $client->state,
        'postcode' => $client->postcode,
        'ip' => $client->ip,
        'user_id' => $userId,
    ]);
    
    Capsule::table('mod_risk_profiles')
        ->updateOrInsert(
            ['user_id' => $userId],
            [
                'risk_score' => $result['total_score'],
                'risk_level' => $result['risk_level'],
                'risk_factors' => json_encode($result['factors']),
                'last_calculated' => date('Y-m-d H:i:s'),
            ]
        );
    
    return $result;
}
```

### Step 3: Use Risk Scores
```php
<?php
add_hook('OrderPlaced', 1, function($vars) {
    $riskProfile = Capsule::table('mod_risk_profiles')
        ->where('user_id', $vars['userId'])
        ->first();
    
    if ($riskProfile && $riskProfile->risk_level === 'critical') {
        // Block the order
        return [
            'result' => 'error',
            'error_msg' => 'Order requires additional verification. Please contact support.',
        ];
    }
    
    if ($riskProfile && $riskProfile->risk_level === 'high') {
        // Flag for review and continue
        Capsule::table('mod_order_reviews')->insert([
            'order_id' => $vars['orderId'],
            'user_id' => $vars['userId'],
            'risk_score' => $riskProfile->risk_score,
            'reason' => 'High risk profile',
            'status' => 'pending',
        ]);
    }
});
```

## Risk Levels
| Level | Score Range | Action |
|-------|-------------|--------|
| Minimal | 0-19 | Auto-approve |
| Low | 20-39 | Auto-approve with logging |
| Medium | 40-59 | Flag for review |
| High | 60-79 | Manual approval required |
| Critical | 80-100 | Block or additional verification |

## Tags
- risk-scoring
- fraud
- security
- assessment