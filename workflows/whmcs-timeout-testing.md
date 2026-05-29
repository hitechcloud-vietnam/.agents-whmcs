# WHMCS Timeout Testing Workflow

## Overview
This workflow provides comprehensive guidance for testing timeout handling in WHMCS modules and integrations.

## Prerequisites
- WHMCS installation (v8.0+)
- Network simulation tools
- Timeout configuration

## Step-by-Step Guide

### Step 1: Configure Timeout Settings
```php
// modules/addons/yourmodule/config.php
<?php
return [
    'timeout' => [
        'api' => 30,           // API call timeout (seconds)
        'webhook' => 10,       // Webhook response timeout
        'database' => 30,       // Database query timeout
        'external' => 60,       // External service timeout
        'cache' => 5,          // Cache operation timeout
    ],
    'retry' => [
        'max_attempts' => 3,
        'delay_ms' => 1000,
        'backoff_multiplier' => 2,
    ],
];
```

### Step 2: Write Timeout Tests
```php
// tests/TimeoutTest.php
<?php
namespace WHMCS\Tests;

use PHPUnit\Framework\TestCase;

class TimeoutTest extends TestCase
{
    public function testApiTimeoutHandling()
    {
        $client = new \GuzzleHttp\Client([
            'timeout' => 1, // 1 second timeout
        ]);
        
        $this->expectException(\GuzzleHttp\Exception\ConnectException::class);
        
        // This should timeout connecting to non-routable IP
        $client->get('http://10.255.255.1/');
    }

    public function testReadTimeoutHandling()
    {
        $client = new \GuzzleHttp\Client([
            'connect_timeout' => 10,
            'timeout' => 1, // 1 second read timeout
        ]);
        
        $this->expectException(\GuzzleHttp\Exception\ConnectException::class);
        
        // Simulate slow response
        $client->get('http://httpbin.org/delay/10');
    }

    public function testDatabaseQueryTimeout()
    {
        $this->expectException(\PDOException::class);
        
        // Set query timeout
        \WHMCS\Database\Capsule::connection()
            ->getPdo()
            ->query('SET MAX_EXECUTION_TIME=100');
        
        // Run long query
        \WHMCS\Database\Capsule::table('tblactivitylog')->get();
    }

    public function testSessionTimeout()
    {
        // Set short session lifetime for testing
        ini_set('session.gc_maxlifetime', 1);
        
        session_start();
        $_SESSION['test'] = 'value';
        
        // Simulate time passing
        sleep(2);
        
        session_start(); // Regenerate session
        
        // Session should be expired
        $this->assertEmpty($_SESSION['test']);
    }

    public function testCacheTimeout()
    {
        $cache = new \WHMCS\Module\YourModule\Cache();
        
        $cache->set('test_key', 'test_value', 1); // 1 second TTL
        
        // Immediate read should work
        $this->assertEquals('test_value', $cache->get('test_key'));
        
        // Wait for expiration
        sleep(2);
        
        // Should return null after expiration
        $this->assertNull($cache->get('test_key'));
    }

    public function testWebhookTimeoutResponse()
    {
        // Simulate slow webhook processing
        $handler = new \WHMCS\Module\YourModule\SlowWebhookHandler(60);
        
        $start = microtime(true);
        
        $result = $handler->processWithTimeout(30);
        
        $duration = microtime(true) - $start;
        
        $this->assertTrue($result['completed']);
        $this->assertLessThan(35, $duration);
    }

    public function testTimeoutGracefulShutdown()
    {
        $processor = new \WHMCS\Module\YourModule\BatchProcessor();
        
        // Set timeout
        $processor->setTimeout(1);
        
        // Process batch that would exceed timeout
        $items = array_fill(0, 1000, ['id' => 1, 'data' => 'test']);
        
        $result = $processor->processWithTimeout($items);
        
        // Should have processed some items before timeout
        $this->assertGreaterThan(0, $result['processed']);
        $this->assertArrayHasKey('timeout_reached', $result);
    }

    public function testConcurrentTimeoutHandling()
    {
        $timeout = 2;
        $tasks = [];
        
        for ($i = 0; $i < 5; $i++) {
            $tasks[] = new class($i, $timeout) {
                private int $id;
                private int $timeout;
                
                public function __construct(int $id, int $timeout) {
                    $this->id = $id;
                    $this->timeout = $timeout;
                }
                
                public function execute(): array {
                    $start = microtime(true);
                    
                    // Simulate varying execution times
                    $delay = ($this->id % 3) * 0.5 + 0.5;
                    usleep((int)($delay * 1000000));
                    
                    $duration = microtime(true) - $start;
                    
                    return [
                        'id' => $this->id,
                        'duration' => $duration,
                        'timeout' => $this->timeout,
                        'completed' => $duration < $this->timeout,
                    ];
                }
            };
        }
        
        $handler = new \WHMCS\Module\YourModule\ConcurrentHandler($timeout);
        $results = $handler->executeAll($tasks);
        
        // All should complete within timeout
        foreach ($results as $result) {
            $this->assertTrue($result['completed']);
        }
    }

    public function testRetryOnTimeout()
    {
        $attempts = 0;
        $maxRetries = 3;
        
        $client = new class {
            public function call() {
                global $attempts;
                $attempts++;
                
                if ($attempts < 3) {
                    throw new \GuzzleHttp\Exception\ConnectException(
                        'Connection timeout',
                        new \GuzzleHttp\Psr7\Request('GET', 'http://example.com')
                    );
                }
                
                return ['success' => true];
            }
        };
        
        $handler = new \WHMCS\Module\YourModule\RetryHandler($maxRetries, 100);
        $result = $handler->executeWithRetry([$client, 'call']);
        
        $this->assertEquals(3, $attempts);
        $this->assertTrue($result['success']);
    }
}
```

### Step 3: Integration Timeout Tests
```php
// tests/Integration/TimeoutIntegrationTest.php
<?php
namespace WHMCS\Tests\Integration;

class TimeoutIntegrationTest extends IntegrationTestCase
{
    public function testInvoiceGenerationTimeout()
    {
        // Test with large number of items
        $start = microtime(true);
        
        try {
            $invoice = \WHMCS\Billing\Invoice::find(1);
            $invoice->items()->createMany(
                array_fill(0, 1000, [
                    'userid' => 1,
                    'type' => 'Item',
                    'relid' => 1,
                    'description' => 'Test Item',
                    'amount' => 10.00,
                ])
            );
            
            $invoice->refresh();
            $total = $invoice->subtotal;
            
            $duration = microtime(true) - $start;
            
            // Should complete within reasonable time
            $this->assertLessThan(10, $duration);
        } catch (\Exception $e) {
            // Should handle timeout gracefully
            $this->assertStringContains('timeout', strtolower($e->getMessage()));
        }
    }

    public function testApiSyncTimeout()
    {
        $sync = new \WHMCS\Module\YourModule\SyncService();
        $sync->setTimeout(5);
        
        $result = $sync->syncAllClients();
        
        if ($result['timeout_reached']) {
            $this->assertGreaterThan(0, $result['synced']);
            $this->assertGreaterThan($result['synced'], $result['total']);
        }
    }
}
```

### Step 4: Run Timeout Tests
```bash
# Run timeout tests
./vendor/bin/phpunit tests/TimeoutTest.php

# Run with verbose output
./vendor/bin/phpunit tests/TimeoutTest.php --testdox

# Run specific timeout test
./vendor/bin/phpunit tests/TimeoutTest.php --filter testApiTimeoutHandling
```

## Timeout Testing Checklist

### API Timeouts
- [ ] Connection timeout set
- [ ] Read timeout configured
- [ ] Timeout handling tested
- [ ] Retry logic implemented

### Database Timeouts
- [ ] Query timeout set
- [ ] Lock timeout configured
- [ ] Long query handling tested

### Session Timeouts
- [ ] Session lifetime configured
- [ ] Session timeout tested
- [ ] Graceful logout tested

### Background Process Timeouts
- [ ] Background job timeout set
- [ ] Progress tracking implemented
- [ ] Partial completion handled
