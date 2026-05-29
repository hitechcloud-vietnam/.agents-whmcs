# WHMCS Client NPS Tracking Workflow

## Description
Track Net Promoter Score for clients.

## Steps

### Step 1: Send NPS Survey
```php
<?php
function sendNPSSurvey($clientId, $daysAfterOrder = 30)
{
    $lastOrder = Capsule::table('tblorders')
        ->where('userid', $clientId)
        ->where('status', 'Active')
        ->orderBy('date', 'desc')
        ->first();
    
    if (!$lastOrder) {
        return false;
    }
    
    $daysSinceOrder = daysSince($lastOrder->date);
    if ($daysSinceOrder < $daysAfterOrder) {
        return false;
    }
    
    // Check if already sent
    $existing = Capsule::table('mod_nps_surveys')
        ->where('client_id', $clientId)
        ->where('status', 'completed')
        ->first();
    
    if ($existing) {
        return false;
    }
    
    $token = bin2hex(random_bytes(16));
    
    Capsule::table('mod_nps_surveys')->insert([
        'client_id' => $clientId,
        'order_id' => $lastOrder->id,
        'token' => $token,
        'sent_at' => date('Y-m-d H:i:s'),
    ]);
    
    $surveyUrl = $systemUrl . '/nps.php?token=' . $token;
    
    sendEmail('NPSSurvey', $clientId, [
        'order_id' => $lastOrder->id,
        'survey_url' => $surveyUrl,
    ]);
}
```

### Step 2: Calculate NPS
```php
<?php
function calculateNPS()
{
    $responses = Capsule::table('mod_nps_surveys')
        ->where('status', 'completed')
        ->get();
    
    $promoters = $responses->filter(fn($r) => $r->score >= 9)->count();
    $passives = $responses->filter(fn($r) => $r->score >= 7 && $r->score < 9)->count();
    $detractors = $responses->filter(fn($r) => $r->score < 7)->count();
    $total = $responses->count();
    
    if ($total === 0) {
        return ['nps' => 0, 'promoters' => 0, 'passives' => 0, 'detractors' => 0];
    }
    
    $nps = round((($promoters - $detractors) / $total * 100);
    
    return [
        'nps' => $nps,
        'promoters' => round($promoters / $total * 100),
        'passives' => round($passives / $total * 100),
        'detractors' => round($detractors / $total * 100),
    ];
}
```

## Tags
- nps
- net-promoter-score
- loyalty-metrics
- survey