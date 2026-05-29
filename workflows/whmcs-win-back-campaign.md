# WHMCS Win-Back Campaign Workflow

## Overview
Comprehensive workflow for re-engaging inactive customers and winning back churned clients.

## Prerequisites
- WHMCS v8.0+
- Activity tracking

## Step-by-Step Guide

### Step 1: Identify Win-Back Targets
```php
<?php
class WinBackIdentifier
{
    private int $inactiveDaysThreshold;
    private int $churnedDaysThreshold;

    public function __construct(
        int $inactiveDaysThreshold = 30,
        int $churnedDaysThreshold = 90
    ) {
        $this->inactiveDaysThreshold = $inactiveDaysThreshold;
        $this->churnedDaysThreshold = $churnedDaysThreshold;
    }

    public function getInactiveClients(): array
    {
        return \WHMCS\Database\Capsule::table('tblclients')
            ->where('status', 'Active')
            ->where('lastlogin', '<', date('Y-m-d H:i:s', strtotime("-{$this->inactiveDaysThreshold} days")))
            ->get();
    }

    public function getChurnedClients(): array
    {
        return \WHMCS\Database\Capsule::table('tblclients')
            ->where('status', 'Closed')
            ->where('notes', 'LIKE', '%churned%')
            ->where('lastupdate', '<', date('Y-m-d H:i:s', strtotime("-{$this->churnedDaysThreshold} days")))
            ->get();
    }

    public function classifyClient($client): string
    {
        $lastLogin = strtotime($client->lastlogin);
        $daysSinceLogin = (time() - $lastLogin) / 86400;

        if ($daysSinceLogin > $this->churnedDaysThreshold) {
            return 'lapsed';
        } elseif ($daysSinceLogin > $this->inactiveDaysThreshold) {
            return 'inactive';
        }
        return 'active';
    }
}
```

### Step 2: Win-Back Offer Generator
```php
<?php
class WinBackOfferGenerator
{
    private array $offers = [];

    public function __construct()
    {
        $this->offers = [
            'inactive' => [
                'discount' => 20,
                'free_month' => true,
                'priority_support' => true,
            ],
            'lapsed' => [
                'discount' => 30,
                'free_setup' => true,
                'loyalty_points' => 500,
            ],
        ];
    }

    public function getOffer(string $segment): array
    {
        return $this->offers[$segment] ?? [];
    }

    public function createOfferCode(string $segment, int $clientId): string
    {
        $offer = $this->offers[$segment];
        $code = strtoupper(substr(md5($clientId . time()), 0, 8));

        \WHMCS\Database\Capsule::table('mod_yourmodule_winback_offers')->insert([
            'client_id' => $clientId,
            'segment' => $segment,
            'offer_code' => $code,
            'discount_percent' => $offer['discount'],
            'expires_at' => date('Y-m-d H:i:s', strtotime('+7 days')),
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        return $code;
    }
}
```

### Step 3: Automated Win-Back Campaign
```php
<?php
add_hook('AfterCronJob', 1, function() {
    $identifier = new WinBackIdentifier();
    $offerGenerator = new WinBackOfferGenerator();

    // Find inactive clients without recent win-back attempts
    $inactiveClients = $identifier->getInactiveClients();

    foreach ($inactiveClients as $client) {
        $segment = $identifier->classifyClient($client);
        
        // Check if already sent win-back recently
        $recentAttempt = \WHMCS\Database\Capsule::table('mod_yourmodule_winback_offers')
            ->where('client_id', $client->id)
            ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-30 days')))
            ->first();

        if (!$recentAttempt) {
            $offerCode = $offerGenerator->createOfferCode($segment, $client->id);
            $offer = $offerGenerator->getOffer($segment);

            send_email($client->email, 'We Miss You!', [
                'client_name' => $client->fullName,
                'segment' => $segment,
                'offer_code' => $offerCode,
                'discount' => $offer['discount'],
                'expires' => date('Y-m-d', strtotime('+7 days')),
            ]);
        }
    }
});
```

## Win-Back Strategy
1. Identify inactive/churned clients
2. Classify by engagement level
3. Generate personalized offers
4. Send targeted emails
5. Track offer redemption
6. Measure campaign ROI
