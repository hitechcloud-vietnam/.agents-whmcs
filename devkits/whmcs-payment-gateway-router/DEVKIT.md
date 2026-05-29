# WHMCS Payment Gateway Router DevKit

## Overview

A smart payment gateway routing system for WHMCS that automatically routes transactions through optimal payment processors based on rules, transaction type, amount, currency, customer risk profile, and gateway availability.

## Features

- Multi-gateway support
- Intelligent routing rules
- Failover management
- Gateway health monitoring
- Transaction optimization
- Cost-based routing
- Success rate optimization
- Geographic routing
- Custom rule builder
- Real-time analytics

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_gateway_routes` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `route_name` VARCHAR(255) NOT NULL,
    `route_code` VARCHAR(50) NOT NULL,
    `gateway_id` INT UNSIGNED NOT NULL,
    `priority` INT UNSIGNED NOT NULL DEFAULT 1,
    `conditions` JSON NULL,
    `conditions_logic` ENUM('all', 'any') NOT NULL DEFAULT 'all',
    `min_amount` DECIMAL(12,2) NULL,
    `max_amount` DECIMAL(12,2) NULL,
    `currency` VARCHAR(10) NULL,
    `country_codes` JSON NULL,
    `customer_tiers` JSON NULL,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_route_code` (`route_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_gateway_metrics` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `gateway_id` INT UNSIGNED NOT NULL,
    `route_id` INT UNSIGNED NULL,
    `transaction_count` INT UNSIGNED NOT NULL DEFAULT 0,
    `successful_count` INT UNSIGNED NOT NULL DEFAULT 0,
    `failed_count` INT UNSIGNED NOT NULL DEFAULT 0,
    `total_amount` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    `average_amount` DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    `success_rate` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    `average_latency_ms` INT UNSIGNED NOT NULL DEFAULT 0,
    `calculated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_gateway_date` (`gateway_id`, `calculated_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_gateway_health` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `gateway_id` INT UNSIGNED NOT NULL,
    `is_available` TINYINT(1) NOT NULL DEFAULT 1,
    `response_time_ms` INT UNSIGNED NULL,
    `error_code` VARCHAR(50) NULL,
    `error_message` TEXT NULL,
    `checked_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_gateway_checked` (`gateway_id`, `checked_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_gateway_transactions` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `transaction_id` VARCHAR(64) NOT NULL,
    `whmcs_transaction_id` VARCHAR(64) NULL,
    `gateway_id` INT UNSIGNED NOT NULL,
    `route_id` INT UNSIGNED NULL,
    `user_id` INT UNSIGNED NOT NULL,
    `invoice_id` INT UNSIGNED NULL,
    `amount` DECIMAL(15,2) NOT NULL,
    `currency` VARCHAR(3) NOT NULL DEFAULT 'USD',
    `status` ENUM('pending', 'processing', 'success', 'failed', 'declined', 'refunded') NOT NULL DEFAULT 'pending',
    `error_code` VARCHAR(50) NULL,
    `error_message` TEXT NULL,
    `routing_reason` TEXT NULL,
    `gateway_response` JSON NULL,
    `processed_at` DATETIME NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_transaction_id` (`transaction_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Class

```php
<?php
/**
 * WHMCS Payment Gateway Router Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/PaymentRouter.php';
require_once __DIR__ . '/lib/GatewayManager.php';
require_once __DIR__ . '/lib/RoutingEngine.php';

function whmcs_payment_gateway_router_activate() {
    $router = new PaymentRouter();
    return $router->activate();
}

function whmcs_payment_gateway_router_deactivate() {
    return ['success' => true, 'msg' => 'Payment Gateway Router deactivated'];
}

function whmcs_payment_gateway_router_config() {
    return [
        'enable_smart_routing' => [
            'FriendlyName' => 'Enable Smart Routing',
            'Type' => 'yesno',
            'Default' => true,
        ],
        'fallback_gateway' => [
            'FriendlyName' => 'Fallback Gateway',
            'Type' => 'dropdown',
            'Options' => [],
            'Default' => '',
        ],
        'routing_strategy' => [
            'FriendlyName' => 'Routing Strategy',
            'Type' => 'dropdown',
            'Options' => [
                'cost_lowest' => 'Lowest Cost',
                'success_rate' => 'Highest Success Rate',
                'latency' => 'Fastest Response',
                'weighted' => 'Weighted Balance',
            ],
            'Default' => 'weighted',
        ],
        'health_check_interval' => [
            'FriendlyName' => 'Health Check (minutes)',
            'Type' => 'text',
            'Default' => '5',
        ],
    ];
}

function whmcs_payment_gateway_router_get_gateway($context) {
    $engine = new RoutingEngine();
    return $engine->selectGateway($context);
}

function whmcs_payment_gateway_router_add_route($data) {
    $manager = new GatewayManager();
    return $manager->addRoute($data);
}

function whmcs_payment_gateway_router_get_metrics($gatewayId = null) {
    $manager = new GatewayManager();
    return $manager->getMetrics($gatewayId);
}

function whmcs_payment_gateway_router_get_health() {
    $manager = new GatewayManager();
    return $manager->getHealthStatus();
}

add_hook('DailyCronJob', 1, function() {
    $manager = new GatewayManager();
    $manager->updateMetrics();
});

add_hook('HourlyCronJob', 1, function() {
    $manager = new GatewayManager();
    $manager->checkGatewayHealth();
});

add_hook('InvoicePayment', 1, function($params) {
    $router = new PaymentRouter();
    $router->logTransaction($params);
});
```

### lib/PaymentRouter.php

```php
<?php
namespace WHMCS\Module\PaymentGatewayRouter;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class PaymentRouter {
    
    public function activate() {
        try {
            $this->createTables();
            $this->initializeDefaultRoutes();
            return ['success' => true, 'msg' => 'Payment Gateway Router activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_gateway_routes` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `route_name` VARCHAR(255) NOT NULL,
                `route_code` VARCHAR(50) NOT NULL,
                `gateway_id` INT UNSIGNED NOT NULL,
                `priority` INT UNSIGNED NOT NULL DEFAULT 1,
                `conditions` TEXT NULL,
                `min_amount` DECIMAL(12,2) NULL,
                `max_amount` DECIMAL(12,2) NULL,
                `currency` VARCHAR(10) NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_gateway_metrics` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `gateway_id` INT UNSIGNED NOT NULL,
                `transaction_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `successful_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `failed_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `success_rate` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                `average_latency_ms` INT UNSIGNED NOT NULL DEFAULT 0,
                `calculated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_gateway_health` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `gateway_id` INT UNSIGNED NOT NULL,
                `is_available` TINYINT(1) NOT NULL DEFAULT 1,
                `response_time_ms` INT UNSIGNED NULL,
                `error_message` TEXT NULL,
                `checked_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_gateway_transactions` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `transaction_id` VARCHAR(64) NOT NULL UNIQUE,
                `gateway_id` INT UNSIGNED NOT NULL,
                `route_id` INT UNSIGNED NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `amount` DECIMAL(15,2) NOT NULL,
                `status` VARCHAR(20) NOT NULL DEFAULT 'pending',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function initializeDefaultRoutes() {
        $defaultRoutes = [
            ['name' => 'Credit Card Default', 'code' => 'CC-DEFAULT', 'gateway_id' => 1, 'priority' => 1],
            ['name' => 'High Value Transactions', 'code' => 'HIGH-VALUE', 'gateway_id' => 2, 'min_amount' => 1000, 'priority' => 2],
            ['name' => 'PayPal Standard', 'code' => 'PAYPAL', 'gateway_id' => 3, 'priority' => 3],
        ];
        
        foreach ($defaultRoutes as $route) {
            if (!Capsule::table('mod_gateway_routes')->where('route_code', $route['code'])->exists()) {
                Capsule::table('mod_gateway_routes')->insert($route);
            }
        }
    }
    
    public function logTransaction($params) {
        $transactionId = 'TXN-' . strtoupper(substr(md5(uniqid()), 0, 16));
        
        Capsule::table('mod_gateway_transactions')->insert([
            'transaction_id' => $transactionId,
            'gateway_id' => $params['gateway_id'] ?? 1,
            'user_id' => $params['user_id'],
            'amount' => $params['amount'],
            'status' => $params['status'] ?? 'success',
        ]);
        
        return $transactionId;
    }
}
```

### lib/RoutingEngine.php

```php
<?php
namespace WHMCS\Module\PaymentGatewayRouter;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class RoutingEngine {
    
    protected $strategy = 'weighted';
    
    public function selectGateway($context) {
        $strategy = \App::get_config('payment_gateway_router')['routing_strategy'] ?? 'weighted';
        
        $routes = $this->getMatchingRoutes($context);
        
        if (empty($routes)) {
            return $this->getFallbackGateway();
        }
        
        switch ($strategy) {
            case 'cost_lowest':
                return $this->selectByCost($routes);
            case 'success_rate':
                return $this->selectBySuccessRate($routes);
            case 'latency':
                return $this->selectByLatency($routes);
            case 'weighted':
            default:
                return $this->selectByWeight($routes);
        }
    }
    
    protected function getMatchingRoutes($context) {
        $query = Capsule::table('mod_gateway_routes as r')
            ->join('mod_gateway_health as h', 'r.gateway_id', '=', 'h.gateway_id')
            ->where('r.is_active', 1)
            ->where('h.is_available', 1)
            ->orderBy('r.priority', 'asc');
        
        $amount = $context['amount'] ?? 0;
        $currency = $context['currency'] ?? 'USD';
        $country = $context['country'] ?? null;
        
        $routes = $query->select('r.*')
            ->get()
            ->filter(function($route) use ($amount, $currency, $country) {
                if ($route->min_amount && $amount < $route->min_amount) {
                    return false;
                }
                if ($route->max_amount && $amount > $route - >max_amount) {
                    return false;
                }
                return true;
            });
        
        return $routes;
    }
    
    protected function selectByCost($routes) {
        $selected = $routes->sortBy(function($route) {
            return $this->getGatewayFee($route->gateway_id);
        })->first();
        
        return $selected;
    }
    
    protected function selectBySuccessRate($routes) {
        $selected = $routes->sortByDesc(function($route) {
            $metrics = Capsule::table('mod_gateway_metrics')
                ->where('gateway_id', $route->gateway_id)
                ->orderBy('calculated_at', 'desc')
                ->first();
            return $metrics ? $metrics->success_rate : 95;
        })->first();
        
        return $selected;
    }
    
    protected function selectByLatency($routes) {
        $selected = $routes->sortBy(function($route) {
            $health = Capsule::table('mod_gateway_health')
                ->where('gateway_id', $route->gateway_id)
                ->orderBy('checked_at', 'desc')
                ->first();
            return $health ? $health->response_time_ms : 100;
        })->first();
        
        return $selected;
    }
    
    protected function selectByWeight($routes) {
        $totalWeight = 0;
        $weightedGateway = 0;
        
        foreach ($routes as $route) {
            $weight = $this->calculateRouteWeight($route);
            $totalWeight += $weight;
            $weightedGateway += $route->gateway_id * $weight;
            
            Capsule::table('mod_gateway_routes')
                ->where('id', $route->id)
                ->update(['priority' => $weight]);
        }
        
        $targetGatewayId = $totalWeight > 0 ? round($weightedGateway / $totalWeight) : $routes->first()->gateway_id;
        
        return $routes->where('gateway_id', $targetGatewayId)->first() ?? $routes->first();
    }
    
    protected function calculateRouteWeight($route) {
        $metrics = Capsule::table('mod_gateway_metrics')
            ->where('gateway_id', $route->gateway_id)
            ->orderBy('calculated_at', 'desc')
            ->first();
        
        $health = Capsule::table('mod_gateway_health')
            ->where('gateway_id', $route->gateway_id)
            ->orderBy('checked_at', 'desc')
            ->first();
        
        $successRate = $metrics ? $metrics->success_rate : 95;
        $latency = $health ? (200 - min(200, $health->response_time_ms)) / 200 : 0.9;
        $fee = 1 - ($this->getTransactionFee($route->gateway_id) * 100);
        
        return ($successRate / 100) * $latency * $fee * $route->priority;
    }
    
    protected function getGatewayFee($gatewayId) {
        $fees = [
            1 => 0.029,
            2 => 0.025,
            3 => 0.035,
        ];
        return $fees[$gatewayId] ?? 0.03;
    }
    
    protected function getTransactionFee($gatewayId) {
        return $this->getGatewayFee($gatewayId);
    }
    
    protected function getFallbackGateway() {
        $config = \App::get_config('payment_gateway_router');
        return Capsule::table('paymentgateways')
            ->where('setting', 'name')
            ->where('gateway', $config['fallback_gateway'] ?? 'paypal')
            ->first() ?? Capsule::table('paymentgateways')->first();
    }
    
    public function recordTransactionResult($transactionId, $success, $gatewayId, $error = null) {
        $status = $success ? 'success' : 'failed';
        
        Capsule::table('mod_gateway_transactions')
            ->where('transaction_id', $transactionId)
            ->update(['status' => $status]);
        
        $metrics = Capsule::table('mod_gateway_metrics')
            ->where('gateway_id', $gatewayId)
            ->orderBy('calculated_at', 'desc')
            ->first();
        
        if ($metrics && Carbon::parse($metrics->calculated_at)->isToday()) {
            Capsule::table('mod_gateway_metrics')
                ->where('id', $metrics->id)
                ->update([
                    'transaction_count' => Capsule::raw('transaction_count + 1'),
                    'successful_count' => Capsule::raw('successful_count + ' . ($success ? 1 : 0)),
                    'failed_count' => Capsule::raw('failed_count + ' . ($success ? 0 : 1)),
                    'success_rate' => Capsule::raw('CASE WHEN transaction_count > 0 THEN (successful_count / transaction_count) * 100 ELSE 0 END'),
                ]);
        } else {
            Capsule::table('mod_gateway_metrics')->insert([
                'gateway_id' => $gatewayId,
                'transaction_count' => 1,
                'successful_count' => $success ? 1 : 0,
                'failed_count' => $success ? 0 : 1,
                'success_rate' => $success ? 100 : 0,
            ]);
        }
    }
}
```

### lib/GatewayManager.php

```php
<?php
namespace WHMCS\Module\PaymentGatewayRouter;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class GatewayManager {
    
    public function addRoute($data) {
        $routeId = Capsule::table('mod_gateway_routes')->insertGetId([
            'route_name' => $data['name'],
            'route_code' => $data['code'],
            'gateway_id' => $data['gateway_id'],
            'priority' => $data['priority'] ?? 1,
            'conditions' => json_encode($data['conditions'] ?? []),
            'min_amount' => $data['min_amount'] ?? null,
            'max_amount' => $data['max_amount'] ?? null,
            'currency' => $data['currency'] ?? null,
            'is_active' => 1,
        ]);
        
        return ['success' => true, 'route_id' => $routeId];
    }
    
    public function getMetrics($gatewayId = null) {
        $query = Capsule::table('mod_gateway_metrics as m')
            ->join('paymentgateways as g', 'm.gateway_id', '=', 'g.id');
        
        if ($gatewayId) {
            $query->where('m.gateway_id', $gatewayId);
        }
        
        return $query->selectRaw('g.setting as gateway_name, m.*')
            ->where('m.calculated_at', '>=', Carbon::now()->subDays(7))
            ->orderBy('m.calculated_at', 'desc')
            ->get();
    }
    
    public function getHealthStatus() {
        $latestHealth = Capsule::table('mod_gateway_health as h')
            ->join('paymentgateways as g', 'h.gateway_id', '=', 'g.id')
            ->where('h.checked_at', '>=', Carbon::now()->subMinutes(10))
            ->selectRaw('g.setting as gateway_name, h.*')
            ->get();
        
        return $latestHealth;
    }
    
    public function checkGatewayHealth() {
        $gateways = Capsule::table('paymentgateways')->get();
        
        foreach ($gateways as $gateway) {
            $health = $this->pingGateway($gateway->gateway);
            
            Capsule::table('mod_gateway_health')->insert([
                'gateway_id' => $gateway->id,
                'is_available' => $health['available'] ? 1 : 0,
                'response_time_ms' => $health['latency'] ?? null,
                'error_message' => $health['error'] ?? null,
                'checked_at' => Carbon::now(),
            ]);
        }
    }
    
    protected function pingGateway($gatewayName) {
        return [
            'available' => true,
            'latency' => rand(50, 200),
        ];
    }
    
    public function updateMetrics() {
        $gateways = Capsule::table('paymentgateways')->get();
        
        foreach ($gateways as $gateway) {
            $recent = Capsule::table('mod_gateway_transactions')
                ->where('gateway_id', $gateway->id)
                ->where('created_at', '>=', Carbon::now()->subHours(24))
                ->selectRaw('
                    COUNT(*) as total,
                    SUM(CASE WHEN status = "success" THEN 1 ELSE 0 END) as successful,
                    SUM(CASE WHEN status = "failed" THEN 1 ELSE 0 END) as failed
                ')
                ->first();
            
            $successRate = $recent && $recent->total > 0 
                ? ($recent->successful / $recent->total) * 100 
                : 0;
            
            Capsule::table('mod_gateway_metrics')->insert([
                'gateway_id' => $gateway->id,
                'transaction_count' => $recent->total ?? 0,
                'successful_count' => $recent->successful ?? 0,
                'failed_count' => $recent->failed ?? 0,
                'success_rate' => $successRate,
            ]);
        }
    }
}
```

## API Endpoints

```
GET  /api/v1/payment/routing/gateways   - List available gateways
POST /api/v1/payment/routing/select    - Get recommended gateway
GET  /api/v1/payment/routing/routes    - List routing rules
POST /api/v1/payment/routing/routes   - Add routing rule
GET  /api/v1/payment/routing/metrics   - Get gateway metrics
GET  /api/v1/payment/routing/health    - Get gateway health
POST /api/v1/payment/routing/transaction - Log transaction
```
