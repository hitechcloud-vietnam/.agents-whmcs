# WHMCS Client Churn Analysis Workflow

## Description
Analyze and predict customer churn in WHMCS.

## Steps

### Step 1: Churn Prediction Model
```php
<?php
function calculateChurnRisk($clientId)
{
    $client = Capsule::table('tblclients')->find($clientId);
    
    $riskFactors = [];
    $score = 0;
    
    // Factor 1: Days since last purchase
    $lastOrder = Capsule::table('tblorders')
        ->where('userid', $clientId)
        ->where('status', 'Active')
        ->orderBy('date', 'desc')
        ->first();
    
    $daysSinceOrder = $lastOrder 
        ? daysSince($lastOrder->date) 
        : 999;
    
    if ($daysSinceOrder > 90) {
        $score += 40;
        $riskFactors[] = 'No recent orders';
    } elseif ($daysSinceOrder > 60) {
        $score += 20;
        $riskFactors[] = 'Low order frequency';
    }
    
    // Factor 2: Support tickets
    $openTickets = Capsule::table('tbltickets')
        ->where('userid', $clientId)
        ->whereIn('status', ['Open', 'Awaiting Reply'])
        ->count();
    
    if ($openTickets > 5) {
        $score += 20;
        $riskFactors[] = 'Multiple open tickets';
    }
    
    // Factor 3: Payment issues
    $failedPayments = Capsule::table('tblaccounts')
        ->where('userid', $clientId)
        ->where('description', 'LIKE', '%Failed%')
        ->where('date', '>=', date('Y-m-d', strtotime('-30 days')))
        ->count();
    
    if ($failedPayments > 0) {
        $score += 15;
        $riskFactors[] = 'Recent payment failures';
    }
    
    // Determine risk level
    if ($score >= 60) {
        $riskLevel = 'critical';
    } elseif ($score >= 40) {
        $riskLevel = 'high';
    } elseif ($score >= 20) {
        $riskLevel = 'medium';
    } else {
        $riskLevel = 'low';
    }
    
    return [
        'score' => $score,
        'risk_level' => $riskLevel,
        'risk_factors' => $riskFactors,
    ];
}
```

### Step 2: Churn Dashboard
```php
<?php
function getChurnDashboard()
{
    $totalClients = Capsule::table('tblclients')
        ->where('status', 'Active')
        ->count();
    
    $atRisk = 0;
    $churned = 0;
    
    $clients = Capsule::table('tblclients')->get();
    foreach ($clients as $client) {
        $risk = calculateChurnRisk($client->id);
        if ($risk['risk_level'] === 'critical') {
            $atRisk++;
        }
    }
    
    $churnRate = $totalClients > 0 
        ? ($churned / $totalClients) * 100 
        : 0;
    
    return [
        'total_active' => $totalClients,
        'at_risk' => $atRisk,
        'churn_rate' => round($churnRate, 2),
        'risk_distribution' => getRiskDistribution(),
    ];
}
```

## Tags
- churn
- analytics
- prediction
- risk-assessment