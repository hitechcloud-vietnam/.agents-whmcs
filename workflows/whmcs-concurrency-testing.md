# WHMCS Concurrency Testing Workflow

## Overview
This workflow provides comprehensive guidance for testing concurrent operations in WHMCS modules and customizations.

## Prerequisites
- WHMCS installation (v8.0+)
- Concurrent testing tools
- Multi-threading support

## Step-by-Step Guide

### Step 1: Create Concurrency Test Suite
```php
// tests/ConcurrencyTest.php
<?php
namespace WHMCS\Tests;

use PHPUnit\Framework\TestCase;

class ConcurrencyTest extends TestCase
{
    public function testConcurrentOrderCreation()
    {
        $orders = [];
        $results = [];
        
        // Create 10 concurrent orders
        for ($i = 0; $i < 10; $i++) {
            $orders[] = function() use ($i) {
                return [
                    'userid' => 1,
                    'pid' => 1,
                    'billingcycle' => 'monthly',
                    'paymentmethod' => 'paypal',
                    'clientid' => $i + 100,
                ];
            };
        }
        
        $handler = new \WHMCS\Module\YourModule\ConcurrentOrderHandler();
        $results = $handler->processOrders($orders);
        
        // All orders should succeed
        $successCount = count(array_filter($results, fn($r) => $r['success']));
        $this->assertEquals(10, $successCount);
    }

    public function testConcurrentClientCreation()
    {
        $clients = [];
        
        for ($i = 0; $i < 5; $i++) {
            $clients[] = [
                'firstname' => 'Test',
                'lastname' => 'User' . $i,
                'email' => 'concurrent_test_' . $i . '_' . time() . '@example.com',
                'password2' => 'SecurePass123!',
            ];
        }
        
        $handler = new \WHMCS\Module\YourModule\ClientCreator();
        $results = $handler->createClients($clients);
        
        // All clients should be created
        $this->assertCount(5, $results);
        foreach ($results as $result) {
            $this->assertTrue($result['success']);
            $this->assertGreaterThan(0, $result['client_id']);
        }
    }

    public function testRaceConditionInCounter()
    {
        // Initialize counter
        $key = 'test_counter_' . time();
        \WHMCS\Module\YourModule\Counter::set($key, 0);
        
        $incrementOperations = [];
        
        // Create 100 concurrent increments
        for ($i = 0; $i < 100; $i++) {
            $incrementOperations[] = function() use ($key) {
                \WHMCS\Module\YourModule\Counter::increment($key);
            };
        }
        
        $handler = new \WHMCS\Module\YourModule\ConcurrentExecutor();
        $handler->executeAll($incrementOperations);
        
        // Final value should be 100
        $finalValue = \WHMCS\Module\YourModule\Counter::get($key);
        $this->assertEquals(100, $finalValue);
        
        // Cleanup
        \WHMCS\Module\YourModule\Counter::delete($key);
    }

    public function testConcurrentInvoiceGeneration()
    {
        $invoices = [];
        
        for ($i = 0; $i < 10; $i++) {
            $invoices[] = [
                'userid' => 1,
                'items' => [
                    ['description' => 'Item ' . $i, 'amount' => 10.00 * ($i + 1)],
                ],
            ];
        }
        
        $handler = new \WHMCS\Module\YourModule\InvoiceGenerator();
        $results = $handler->generateInvoices($invoices);
        
        // All invoices should be generated
        $this->assertCount(10, $results);
        
        // Each invoice should have unique ID
        $ids = array_column($results, 'invoice_id');
        $uniqueIds = array_unique($ids);
        $this->assertCount(10, $uniqueIds);
    }

    public function testConcurrentPaymentProcessing()
    {
        // Create test invoice
        $invoice = \WHMCS\Billing\Invoice::create([
            'userid' => 1,
            'status' => 'Unpaid',
            'total' => 100.00,
        ]);
        
        $invoice->addItem([
            'description' => 'Test Service',
            'amount' => 100.00,
        ]);
        
        // Simulate multiple payment callbacks arriving concurrently
        $payments = [];
        for ($i = 0; $i < 5; $i++) {
            $payments[] = [
                'invoice_id' => $invoice->id,
                'transaction_id' => 'txn_' . time() . '_' . $i,
                'amount' => 100.00,
            ];
        }
        
        $handler = new \WHMCS\Module\YourModule\PaymentProcessor();
        $results = $handler->processPayments($payments);
        
        // Only one payment should succeed
        $successCount = count(array_filter($results, fn($r) => $r['applied']));
        $this->assertEquals(1, $successCount);
        
        // Cleanup
        $invoice->delete();
    }

    public function testConcurrentCacheUpdates()
    {
        $key = 'cache_test_' . time();
        
        // Initialize cache
        \WHMCS\Module\YourModule\Cache::set($key, ['version' => 0]);
        
        $updates = [];
        
        // Create 20 concurrent updates
        for ($i = 0; $i < 20; $i++) {
            $updates[] = function() use ($key, $i) {
                $current = \WHMCS\Module\YourModule\Cache::get($key);
                $current['version']++;
                \WHMCS\Module\YourModule\Cache::set($key, $current);
            };
        }
        
        $handler = new \WHMCS\Module\YourModule\ConcurrentExecutor();
        $handler->executeAll($updates);
        
        // Cache should have consistent value
        $final = \WHMCS\Module\YourModule\Cache::get($key);
        $this->assertEquals(20, $final['version']);
        
        // Cleanup
        \WHMCS\Module\YourModule\Cache::delete($key);
    }

    public function testLockMechanism()
    {
        $resourceId = 'lock_test_' . time();
        
        // First lock should succeed
        $lock1 = \WHMCS\Module\YourModule\Lock::acquire($resourceId, 60);
        $this->assertTrue($lock1->acquired());
        
        // Second lock should fail or wait
        $lock2 = \WHMCS\Module\YourModule\Lock::acquire($resourceId, 60);
        
        // Should either fail immediately or wait
        if ($lock2->acquired()) {
            $this->assertNotEquals($lock1->getHolderId(), $lock2->getHolderId());
            $lock2->release();
        } else {
            // Expected behavior - lock already held
            $this->assertFalse($lock2->acquired());
        }
        
        // Release first lock
        $lock1->release();
        
        // Now second lock should succeed
        $lock3 = \WHMCS\Module\YourModule\Lock::acquire($resourceId, 60);
        $this->assertTrue($lock3->acquired());
        $lock3->release();
    }

    public function testSemaphoreForLimitedResource()
    {
        $semaphore = new \WHMCS\Module\YourModule\Semaphore(3); // Max 3 concurrent
        
        $tasks = [];
        $results = [];
        
        // Create 10 tasks competing for semaphore
        for ($i = 0; $i < 10; $i++) {
            $tasks[] = function() use ($semaphore, $i) {
                $acquired = $semaphore->acquire(5); // Wait up to 5 seconds
                if ($acquired) {
                    usleep(100000); // 100ms
                    $semaphore->release();
                    return ['success' => true, 'task' => $i];
                }
                return ['success' => false, 'task' => $i, 'reason' => 'timeout'];
            };
        }
        
        $handler = new \WHMCS\Module\YourModule\ConcurrentExecutor();
        $results = $handler->executeAll($tasks);
        
        // Most tasks should succeed
        $successCount = count(array_filter($results, fn($r) => $r['success']));
        $this->assertGreaterThanOrEqual(8, $successCount);
    }

    public function testReadWriteLock()
    {
        $rwLock = new \WHMCS\Module\YourModule\ReadWriteLock();
        $key = 'rw_test_' . time();
        
        // Initialize
        \WHMCS\Database\Capsule::table('mod_test_rw')->insert(['key' => $key, 'value' => 0]);
        
        $readers = [];
        $writers = [];
        
        // Create readers
        for ($i = 0; $i < 5; $i++) {
            $readers[] = function() use ($rwLock, $key) {
                $lock = $rwLock->readLock();
                $lock->acquire();
                
                $value = \WHMCS\Database\Capsule::table('mod_test_rw')
                    ->where('key', $key)
                    ->value('value');
                
                $lock->release();
                return $value;
            };
        }
        
        // Create writers
        for ($i = 0; $i < 2; $i++) {
            $writers[] = function() use ($rwLock, $key, $i) {
                $lock = $rwLock->writeLock();
                $lock->acquire();
                
                $current = \WHMCS\Database\Capsule::table('mod_test_rw')
                    ->where('key', $key)
                    ->value('value');
                
                \WHMCS\Database\Capsule::table('mod_test_rw')
                    ->where('key', $key)
                    ->update(['value' => $current + 1]);
                
                $lock->release();
                return $current + 1;
            };
        }
        
        $handler = new \WHMCS\Module\YourModule\ConcurrentExecutor();
        
        // Run all
        $handler->executeAll(array_merge($readers, $writers));
        
        // Final value should be 2 (2 writers)
        $final = \WHMCS\Database\Capsule::table('mod_test_rw')
            ->where('key', $key)
            ->value('value');
        $this->assertEquals(2, $final);
        
        // Cleanup
        \WHMCS\Database\Capsule::table('mod_test_rw')->where('key', $key)->delete();
    }
}
```

### Step 2: Run Concurrency Tests
```bash
# Run all concurrency tests
./vendor/bin/phpunit tests/ConcurrencyTest.php

# Run with verbose output
./vendor/bin/phpunit tests/ConcurrencyTest.php --testdox

# Run specific test
./vendor/bin/phpunit tests/ConcurrencyTest.php --filter testRaceConditionInCounter
```

## Concurrency Testing Checklist

### Race Conditions
- [ ] Counter increments are atomic
- [ ] Read-modify-write operations are protected
- [ ] Database transactions are isolated

### Locks
- [ ] Deadlocks are prevented
- [ ] Lock timeouts are set
- [ ] Locks are always released

### Resource Limits
- [ ] Semaphores limit concurrent access
- [ ] Connection pools are sized correctly
- [ ] Memory limits are enforced

### Data Consistency
- [ ] Final state is consistent
- [ ] No lost updates
- [ ] No duplicate processing
