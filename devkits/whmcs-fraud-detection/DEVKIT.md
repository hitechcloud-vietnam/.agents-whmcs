# WHMCS Fraud Detection Module

## Overview
Fraud scoring and risk assessment module for orders.

## Module File: fraud_detection.php

```php
<?php
/**
 * WHMCS Fraud Detection Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Database\Capsule;

class WHMCS_Fraud_Detection
{
    protected $config;

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    /**
     * Score order for fraud
     */
    public function scoreOrder(int $orderId): array
    {
        $order = Capsule::table('tblorders')
            ->where('id', $orderId)
            ->first();

        $client = Capsule::table('tblclients')
            ->where('id', $order->userid)
            ->first();

        $score = 0;
        $riskFactors = [];

        // Check for proxy/VPN
        if ($this->isProxyIP($_SERVER['REMOTE_ADDR'] ?? '')) {
            $score += 30;
            $riskFactors[] = 'proxy_detected';
        }

        // Check email domain
        if ($this->isFreeEmailDomain($client->email ?? '')) {
            $score += 10;
            $riskFactors[] = 'free_email';
        }

        // Check for previous fraud
        if ($this->hasPreviousFraud($client->id)) {
            $score += 50;
            $riskFactors[] = 'previous_fraud';
        }

        // Check order amount
        if ($order->total > ($this->config['high_value_threshold'] ?? 500)) {
            $score += 15;
            $riskFactors[] = 'high_value';
        }

        // Check country mismatch
        if (!$this->isCountryMatch($client, $_SERVER['REMOTE_ADDR'] ?? '')) {
            $score += 25;
            $riskFactors[] = 'country_mismatch';
        }

        // Determine risk level
        $riskLevel = 'low';
        if ($score >= 50) {
            $riskLevel = 'high';
        } elseif ($score >= 25) {
            $riskLevel = 'medium';
        }

        // Store score
        Capsule::table('mod_fraud_scores')->insert([
            'order_id' => $orderId,
            'score' => $score,
            'risk_level' => $riskLevel,
            'risk_factors' => json_encode($riskFactors),
            'analyzed_at' => date('Y-m-d H:i:s'),
        ]);

        return [
            'score' => $score,
            'risk_level' => $riskLevel,
            'risk_factors' => $riskFactors,
        ];
    }

    /**
     * Check if IP is proxy/VPN
     */
    protected function isProxyIP(string $ip): bool
    {
        // Implement actual proxy detection logic
        return false;
    }

    /**
     * Check free email domain
     */
    protected function isFreeEmailDomain(string $email): bool
    {
        $freeDomains = ['gmail.com', 'yahoo.com', 'hotmail.com', 'outlook.com'];
        $domain = strtolower(substr($email, strpos($email, '@') + 1));
        return in_array($domain, $freeDomains);
    }

    /**
     * Check previous fraud
     */
    protected function hasPreviousFraud(int $clientId): bool
    {
        return Capsule::table('mod_fraud_scores')
            ->join('tblorders', 'mod_fraud_scores.order_id', '=', 'tblorders.id')
            ->where('tblorders.userid', $clientId)
            ->where('mod_fraud_scores.risk_level', 'high')
            ->exists();
    }

    /**
     * Check country match
     */
    protected function isCountryMatch(object $client, string $ip): bool
    {
        // Implement actual geolocation check
        return true;
    }
}

// Hook into order creation
add_hook('OrderPaid', 1, function($params) {
    $fraud = new WHMCS_Fraud_Detection();
    $result = $fraud->scoreOrder($params['orderid']);

    if ($result['risk_level'] === 'high') {
        Capsule::table('tblorders')
            ->where('id', $params['orderid'])
            ->update(['status' => 'Fraud']);
    }
});

function whmcs_fraud_detection_activate(): array
{
    try {
        if (!Capsule::schema()->hasTable('mod_fraud_scores')) {
            Capsule::schema()->create('mod_fraud_scores', function ($table) {
                $table->increments('id');
                $table->integer('order_id')->unsigned();
                $table->integer('score')->default(0);
                $table->enum('risk_level', ['low', 'medium', 'high']);
                $table->longText('risk_factors')->nullable();
                $table->timestamp('analyzed_at');
                
                $table->index('order_id');
            });
        }
        return ['status' => 'success', 'description' => 'Fraud Detection activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_fraud_detection_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Fraud Detection deactivated'];
}

function whmcs_fraud_detection_config(): array
{
    return [
        'high_value_threshold' => ['FriendlyName' => 'High Value Threshold', 'Type' => 'text', 'Default' => '500'],
        'auto_flag_fraud' => ['FriendlyName' => 'Auto-Flag High Risk', 'Type' => 'yesno'],
        'proxy_detection' => ['FriendlyName' => 'Enable Proxy Detection', 'Type' => 'yesno'],
    ];
}
```

## Configuration File: config.php

```php
<?php
return [
    'high_value_threshold' => 500,
    'auto_flag_fraud' => true,
    'proxy_detection' => false,
];
```

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
