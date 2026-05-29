# WHMCS Multi-Currency Module

## Overview
Currency conversion module with real-time exchange rates.

## Module File: multi_currency.php

```php
<?php
/**
 * WHMCS Multi-Currency Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Database\Capsule;

class WHMCS_Multi_Currency
{
    protected $config;
    protected $cacheTime = 3600;

    public function __construct()
    {
        $this->config = require __DIR__ . '/config.php';
    }

    /**
     * Get exchange rate
     */
    public function getExchangeRate(string $from, string $to): float
    {
        if ($from === $to) {
            return 1.0;
        }

        $cacheKey = "exchange_rate_{$from}_{$to}";
        $cached = $this->getCachedRate($cacheKey);

        if ($cached !== null) {
            return $cached;
        }

        $rate = $this->fetchExchangeRate($from, $to);
        $this->cacheRate($cacheKey, $rate);

        return $rate;
    }

    /**
     * Fetch exchange rate from API
     */
    protected function fetchExchangeRate(string $from, string $to): float
    {
        $apiUrl = $this->config['exchange_api_url'] ?? '';
        $apiKey = $this->config['exchange_api_key'] ?? '';

        if (empty($apiUrl)) {
            return 1.0;
        }

        $url = str_replace(['{FROM}', '{TO}'], [$from, $to], $apiUrl);

        $response = wp_remote_get($url, [
            'headers' => ['Authorization' => 'Bearer ' . $apiKey],
            'timeout' => 10,
        ]);

        if (is_wp_error($response)) {
            return 1.0;
        }

        $data = json_decode(wp_remote_retrieve_body($response), true);

        return floatval($data['rate'] ?? $data['result'] ?? 1.0);
    }

    /**
     * Get cached rate
     */
    protected function getCachedRate(string $cacheKey): ?float
    {
        $cached = Capsule::table('mod_currency_cache')
            ->where('cache_key', $cacheKey)
            ->where('created_at', '>', date('Y-m-d H:i:s', time() - $this->cacheTime))
            ->first();

        return $cached ? floatval($cached->cache_value) : null;
    }

    /**
     * Cache rate
     */
    protected function cacheRate(string $cacheKey, float $rate): void
    {
        Capsule::table('mod_currency_cache')->updateOrInsert(
            ['cache_key' => $cacheKey],
            ['cache_value' => $rate, 'created_at' => date('Y-m-d H:i:s')]
        );
    }

    /**
     * Convert amount
     */
    public function convert(float $amount, string $from, string $to): float
    {
        $rate = $this->getExchangeRate($from, $to);
        return round($amount * $rate, 2);
    }

    /**
     * Update all exchange rates
     */
    public function updateAllRates(): void
    {
        $currencies = Capsule::table('tblcurrencies')
            ->where('default', 0)
            ->get(['code']);

        $defaultCurrency = Capsule::table('tblcurrencies')
            ->where('default', 1)
            ->value('code') ?? 'USD';

        foreach ($currencies as $currency) {
            $this->getExchangeRate($defaultCurrency, $currency->code);
        }
    }
}

function whmcs_multi_currency_activate(): array
{
    try {
        if (!Capsule::schema()->hasTable('mod_currency_cache')) {
            Capsule::schema()->create('mod_currency_cache', function ($table) {
                $table->string('cache_key', 100)->primary();
                $table->decimal('cache_value', 18, 8);
                $table->timestamp('created_at')->useCurrent();
            });
        }
        return ['status' => 'success', 'description' => 'Multi-Currency activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => $e->getMessage()];
    }
}

function whmcs_multi_currency_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Multi-Currency deactivated'];
}

function whmcs_multi_currency_config(): array
{
    return [
        'exchange_api_url' => ['FriendlyName' => 'Exchange API URL', 'Type' => 'text', 'Size' => '50', 'Description' => 'Use {FROM} and {TO} as placeholders'],
        'exchange_api_key' => ['FriendlyName' => 'API Key', 'Type' => 'password', 'Size' => '50'],
        'cache_duration' => ['FriendlyName' => 'Cache Duration (seconds)', 'Type' => 'text', 'Default' => '3600'],
    ];
}
```

## Configuration File: config.php

```php
<?php
return [
    'exchange_api_url' => '',
    'exchange_api_key' => '',
    'cache_duration' => 3600,
];
```

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
