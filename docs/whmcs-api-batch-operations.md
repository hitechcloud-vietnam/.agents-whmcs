# WHMCS API Batch Operations

## Overview

Efficiently process multiple operations in a single request or queue them for asynchronous processing.

## Sequential Batch Processing

```php
<?php
class WhmcsBatchProcessor {
    private WhmcsApiClient $api;
    private int $batchSize = 50;
    private int $delayBetweenRequests = 100; // milliseconds
    
    public function __construct(WhmcsApiClient $api)
    {
        $this->api = $api;
    }
    
    public function setBatchSize(int $size): self
    {
        $this->batchSize = max(1, min($size, 100));
        return $this;
    }
    
    public function processClients(array $operations, callable $progress = null): array
    {
        $results = [];
        $total = count($operations);
        
        foreach (array_chunk($operations, $this->batchSize) as $batchIndex => $batch) {
            foreach ($batch as $index => $operation) {
                $result = $this->executeClientOperation($operation);
                $results[] = $result;
                
                if ($progress) {
                    $current = ($batchIndex * $this->batchSize) + $index + 1;
                    $progress($current, $total, $result);
                }
                
                $this->delay();
            }
        }
        
        return $results;
    }
    
    private function executeClientOperation(array $operation): array
    {
        $action = $operation['action'] ?? '';
        $params = $operation['params'] ?? [];
        
        try {
            $response = $this->api->makeRequest(
                array_merge(['action' => $action], $params)
            );
            
            return [
                'success' => true,
                'action' => $action,
                'params' => $params,
                'response' => $response,
            ];
        } catch (Exception $e) {
            return [
                'success' => false,
                'action' => $action,
                'params' => $params,
                'error' => $e->getMessage(),
            ];
        }
    }
    
    private function delay(): void
    {
        if ($this->delayBetweenRequests > 0) {
            usleep($this->delayBetweenRequests * 1000);
        }
    }
    
    public function getSummary(array $results): array
    {
        $success = array_filter($results, fn($r) => $r['success'] ?? false);
        $failed = array_filter($results, fn($r) => !($r['success'] ?? false));
        
        return [
            'total' => count($results),
            'successful' => count($success),
            'failed' => count($failed),
            'success_rate' => count($results) > 0 
                ? round(count($success) / count($results) * 100, 2) 
                : 0,
            'failures' => array_values($failed),
        ];
    }
}
```

## Bulk Client Updates

```php
<?php
class BulkClientOperations {
    private WhmcsApiClient $api;
    
    public function __construct(WhmcsApiClient $api)
    {
        $this->api = $api;
    }
    
    public function updateClientStatuses(array $clientIds, string $status): array
    {
        $results = [];
        
        foreach ($clientIds as $clientId) {
            $results[$clientId] = $this->updateClientStatus($clientId, $status);
        }
        
        return $results;
    }
    
    private function updateClientStatus(int $clientId, string $status): array
    {
        return $this->api->makeRequest([
            'action' => 'UpdateClient',
            'clientid' => $clientId,
            'status' => $status,
        ]);
    }
    
    public function addClientNotes(int $clientId, array $notes): array
    {
        $results = [];
        
        foreach ($notes as $note) {
            $result = $this->api->makeRequest([
                'action' => 'AddClientNote',
                'userid' => $clientId,
                'note' => $note,
            ]);
            $results[] = $result;
        }
        
        return $results;
    }
    
    public function massUpdateCustomFields(int $clientId, array $customFields): array
    {
        $params = ['clientid' => $clientId];
        
        foreach ($customFields as $fieldId => $value) {
            $params["customfields[{$fieldId}]"] = $value;
        }
        
        return $this->api->makeRequest(
            array_merge(['action' => 'UpdateClient'], $params)
        );
    }
}
```

## Invoice Batch Processing

```php
<?php
class BulkInvoiceOperations {
    private WhmcsApiClient $api;
    
    public function __construct(WhmcsApiClient $api)
    {
        $this->api = $api;
    }
    
    public function createInvoicesFromOrders(array $orderIds): array
    {
        $results = [];
        
        foreach ($orderIds as $orderId) {
            $result = $this->api->makeRequest([
                'action' => 'CreateInvoiceFromOrder',
                'orderid' => $orderId,
                'date' => date('Y-m-d'),
            ]);
            
            $results[$orderId] = $result;
        }
        
        return $results;
    }
    
    public function payInvoices(array $invoiceIds, string $paymentMethod): array
    {
        $results = [];
        
        foreach ($invoiceIds as $invoiceId) {
            $result = $this->api->makeRequest([
                'action' => 'LocalCredit',
                'invoiceid' => $invoiceId,
                'amount' => 'full', // Or specific amount
                'type' => 'invoice',
            ]);
            
            if ($result['result'] === 'success') {
                $results[$invoiceId] = $this->addPayment($invoiceId, $paymentMethod);
            } else {
                $results[$invoiceId] = $result;
            }
        }
        
        return $results;
    }
    
    private function addPayment(int $invoiceId, string $paymentMethod): array
    {
        return $this->api->makeRequest([
            'action' => 'AddInvoicePayment',
            'invoiceid' => $invoiceId,
            'transid' => 'BATCH-' . $invoiceId . '-' . time(),
            'gateway' => $paymentMethod,
            'date' => date('Y-m-d H:i:s'),
        ]);
    }
    
    public function sendInvoiceReminders(array $invoiceIds): array
    {
        $results = [];
        
        foreach ($invoiceIds as $invoiceId) {
            $result = $this->api->makeRequest([
                'action' => 'SendInvoice',
                'invoiceid' => $invoiceId,
            ]);
            $results[$invoiceId] = $result;
        }
        
        return $results;
    }
}
```

## Service Bulk Operations

```php
<?php
class BulkServiceOperations {
    private WhmcsApiClient $api;
    
    public function __construct(WhmcsApiClient $api)
    {
        $this->api = $api;
    }
    
    public function suspendExpiredServices(array $serviceIds): array
    {
        $results = [];
        
        foreach ($serviceIds as $serviceId) {
            $result = $this->api->makeRequest([
                'action' => 'ModuleSuspend',
                'serviceid' => $serviceId,
            ]);
            $results[$serviceId] = $result;
        }
        
        return $results;
    }
    
    public function terminateServices(array $serviceIds, string $reason = ''): array
    {
        $results = [];
        
        foreach ($serviceIds as $serviceId) {
            $result = $this->api->makeRequest([
                'action' => 'ModuleTerminate',
                'serviceid' => $serviceId,
                'terminatereason' => $reason,
            ]);
            $results[$serviceId] = $result;
        }
        
        return $results;
    }
    
    public function renewServices(array $serviceIds, int $years = 1): array
    {
        $results = [];
        
        foreach ($serviceIds as $serviceId) {
            $result = $this->api->makeRequest([
                'action' => 'UpdateService',
                'serviceid' => $serviceId,
                'regdate' => date('Y-m-d'),
                'nextduedate' => date('Y-m-d', strtotime("+{$years} years")),
                'billingcycle' => 'Annually',
            ]);
            $results[$serviceId] = $result;
        }
        
        return $results;
    }
    
    public function changeServicePackages(array $serviceIds, int $newPackageId): array
    {
        $results = [];
        
        foreach ($serviceIds as $serviceId) {
            $result = $this->api->makeRequest([
                'action' => 'UpdateClientProduct',
                'serviceid' => $serviceId,
                'pid' => $newPackageId,
            ]);
            $results[$serviceId] = $result;
        }
        
        return $results;
    }
}
```

## Async Batch Queue

```php
<?php
class AsyncBatchQueue {
    private array $queue = [];
    private int $maxConcurrent = 5;
    private bool $running = false;
    
    public function __construct(int $maxConcurrent = 5)
    {
        $this->maxConcurrent = $maxConcurrent;
    }
    
    public function enqueue(array $operation): string
    {
        $jobId = bin2hex(random_bytes(8));
        
        $this->queue[$jobId] = [
            'id' => $jobId,
            'operation' => $operation,
            'status' => 'pending',
            'created_at' => time(),
            'attempts' => 0,
        ];
        
        return $jobId;
    }
    
    public function enqueueBulk(array $operations): array
    {
        $jobIds = [];
        
        foreach ($operations as $operation) {
            $jobIds[] = $this->enqueue($operation);
        }
        
        return $jobIds;
    }
    
    public function process(callable $apiCall): array
    {
        $pending = array_filter(
            $this->queue,
            fn($job) => $job['status'] === 'pending'
        );
        
        $chunks = array_chunk($pending, $this->maxConcurrent, true);
        
        $results = [];
        
        foreach ($chunks as $chunk) {
            $chunkResults = $this->processChunk($chunk, $apiCall);
            $results = array_merge($results, $chunkResults);
        }
        
        return $results;
    }
    
    private function processChunk(array $chunk, callable $apiCall): array
    {
        $multiHandle = curl_multi_init();
        $handles = [];
        
        foreach ($chunk as $jobId => $job) {
            $handles[$jobId] = $this->createHandle($job['operation']);
            curl_multi_add_handle($multiHandle, $handles[$jobId]);
        }
        
        $running = null;
        do {
            curl_multi_exec($multiHandle, $running);
            curl_multi_select($multiHandle);
        } while ($running > 0);
        
        $results = [];
        
        foreach ($handles as $jobId => $handle) {
            $response = curl_multi_getcontent($handle);
            curl_multi_remove_handle($multiHandle, $handle);
            
            $results[$jobId] = json_decode($response, true);
            $this->queue[$jobId]['status'] = 'completed';
            $this->queue[$jobId]['result'] = $results[$jobId];
        }
        
        curl_multi_close($multiHandle);
        
        return $results;
    }
    
    private function createHandle(array $operation)
    {
        $ch = curl_init($this->apiUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($operation),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        return $ch;
    }
    
    public function getStatus(string $jobId): ?array
    {
        return $this->queue[$jobId] ?? null;
    }
    
    public function getQueueStats(): array
    {
        $byStatus = [];
        
        foreach ($this->queue as $job) {
            $status = $job['status'];
            if (!isset($byStatus[$status])) {
                $byStatus[$status] = 0;
            }
            $byStatus[$status]++;
        }
        
        return [
            'total' => count($this->queue),
            'by_status' => $byStatus,
        ];
    }
}
```

## Transaction Processing

```php
<?php
class BulkTransactionProcessor {
    private WhmcsApiClient $api;
    private int $transactionLimit = 1000;
    
    public function __construct(WhmcsApiClient $api)
    {
        $this->api = $api;
    }
    
    public function getAllTransactions(
        ?string $startDate = null,
        ?string $endDate = null
    ): array {
        $allTransactions = [];
        $offset = 0;
        
        do {
            $result = $this->api->makeRequest([
                'action' => 'GetTransactions',
                'limitstart' => $offset,
                'limitnum' => 1000,
                'startdate' => $startDate,
                'enddate' => $endDate,
            ]);
            
            $transactions = $result['transactions'] ?? [];
            $allTransactions = array_merge($allTransactions, $transactions);
            
            $offset += count($transactions);
            
        } while (count($transactions) === 1000);
        
        return $allTransactions;
    }
    
    public function applyCredits(array $credits): array
    {
        $results = [];
        
        foreach ($credits as $credit) {
            $result = $this->api->makeRequest([
                'action' => 'AddCredit',
                'clientid' => $credit['client_id'],
                'amount' => $credit['amount'],
                'description' => $credit['description'] ?? 'Bulk credit adjustment',
            ]);
            
            $results[$credit['client_id']] = $result;
        }
        
        return $results;
    }
    
    public function processRefunds(array $refunds): array
    {
        $results = [];
        
        foreach ($refunds as $refund) {
            $result = $this->api->makeRequest([
                'action' => 'RefundInvoicePayment',
                'invoiceid' => $refund['invoice_id'],
                'transid' => $refund['transaction_id'],
                'amount' => $refund['amount'],
                'sendnotification' => $refund['notify'] ?? false,
            ]);
            
            $results[$refund['invoice_id']] = $result;
        }
        
        return $results;
    }
}
```

## Best Practices

1. **Use batching wisely** - Group similar operations
2. **Implement delays** - Respect rate limits between batches
3. **Track progress** - Provide feedback during long operations
4. **Handle failures gracefully** - Continue processing on individual failures
5. **Log everything** - Keep detailed logs for debugging
6. **Use transactions** - Wrap related operations atomically where possible

## Related Documentation

- [WHMCS API Pagination](/docs/whmcs-api-pagination.md)
- [WHMCS API Rate Limiting](/docs/whmcs-api-rate-limiting.md)