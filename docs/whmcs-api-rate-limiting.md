# WHMCS API Rate Limiting

## Overview

WHMCS implements rate limiting to prevent API abuse and ensure system stability.

## Rate Limits

### Default Limits

| Endpoint Type | Limit | Window |
|--------------|-------|--------|
| Standard API | 60 requests | Per minute |
| Authentication | 10 requests | Per minute |
| Bulk operations | 5 requests | Per minute |

### Rate Limit Headers

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1623456789
```

## Implementation

### Handling Rate Limits

```php
<?php
class WhmcsApiClient {
    private int $maxRetries = 3;
    private int $retryDelay = 1000; // milliseconds
    
    public function makeRequest(array $params): array
    {
        $retries = 0;
        
        while ($retries < $this->maxRetries) {
            $response = $this->executeRequest($params);
            
            if ($response['status_code'] === 429) {
                $retries++;
                $retryAfter = $response['retry_after'] ?? $this->retryDelay;
                $this->wait($retryAfter);
                continue;
            }
            
            return $response;
        }
        
        throw new RateLimitException('Max retries exceeded');
    }
    
    private function wait(int $milliseconds): void
    {
        usleep($milliseconds * 1000);
    }
    
    private function executeRequest(array $params): array
    {
        $ch = curl_init($this->apiUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($params),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HEADER => true,
        ]);
        
        $response = curl_exec($ch);
        $headerSize = curl_getinfo($ch, CURLINFO_HEADER_SIZE);
        $body = substr($response, $headerSize);
        $headers = substr($response, 0, $headerSize);
        
        curl_close($ch);
        
        return $this->parseResponse($body, $headers);
    }
    
    private function parseResponse(string $body, string $headers): array
    {
        $headerLines = explode("\r\n", $headers);
        $rateLimitHeaders = [];
        
        foreach ($headerLines as $line) {
            if (stripos($line, 'X-RateLimit-') === 0 || 
                stripos($line, 'Retry-After:') === 0) {
                $parts = explode(':', $line, 2);
                $key = strtolower(str_replace('-', '_', trim($parts[0])));
                $rateLimitHeaders[$key] = trim($parts[1] ?? '');
            }
        }
        
        return array_merge(
            json_decode($body, true) ?? [],
            [
                'status_code' => 200,
                'rate_limits' => $rateLimitHeaders,
            ]
        );
    }
}
```

### Exponential Backoff

```php
<?php
class ExponentialBackoffClient {
    private int $baseDelay = 1000;
    private int $maxDelay = 32000;
    private int $maxRetries = 5;
    
    public function withRetry(callable $operation): mixed
    {
        $attempt = 0;
        
        while (true) {
            try {
                return $operation();
            } catch (RateLimitException $e) {
                $attempt++;
                
                if ($attempt >= $this->maxRetries) {
                    throw $e;
                }
                
                $delay = min($this->baseDelay * pow(2, $attempt), $this->maxDelay);
                $jitter = random_int(0, $delay * 0.1);
                
                $this->wait($delay + $jitter);
            }
        }
    }
    
    private function wait(int $milliseconds): void
    {
        usleep($milliseconds * 1000);
    }
}
```

### Request Queue with Rate Limiting

```php
<?php
class RateLimitedRequestQueue {
    private array $queue = [];
    private int $requestsPerMinute;
    private int $requestCount = 0;
    private ?int $windowStart = null;
    
    public function __construct(int $requestsPerMinute = 60)
    {
        $this->requestsPerMinute = $requestsPerMinute;
    }
    
    public function enqueue(callable $request, array $params): Promise
    {
        $this->queue[] = [
            'request' => $request,
            'params' => $params,
            'promise' => new Promise(),
        ];
        
        return $this->processNext();
    }
    
    private function processNext(): ?Promise
    {
        if (empty($this->queue)) {
            return null;
        }
        
        $this->checkRateLimit();
        
        $item = array_shift($this->queue);
        $result = ($item['request'])($item['params']);
        $item['promise']->resolve($result);
        
        $this->requestCount++;
        
        if (!empty($this->queue)) {
            $delay = ($this->requestCount >= $this->requestsPerMinute)
                ? $this->getRemainingWindowTime()
                : 1000 / ($this->requestsPerMinute / 60);
            
            $this->scheduleNext($delay);
        }
        
        return $item['promise'];
    }
    
    private function checkRateLimit(): void
    {
        $now = time();
        
        if ($this->windowStart === null) {
            $this->windowStart = $now;
            $this->requestCount = 0;
        }
        
        if ($now - $this->windowStart >= 60) {
            $this->windowStart = $now;
            $this->requestCount = 0;
        }
        
        if ($this->requestCount >= $this->requestsPerMinute) {
            $waitTime = 60 - ($now - $this->windowStart);
            sleep($waitTime);
            $this->windowStart = time();
            $this->requestCount = 0;
        }
    }
    
    private function getRemainingWindowTime(): int
    {
        return 60000 - ((time() - $this->windowStart) * 1000);
    }
    
    private function scheduleNext(int $delayMs): void
    {
        // Would use async scheduler in production
        usleep($delayMs * 1000);
        $this->processNext();
    }
}
```

## Monitoring Rate Limits

```php
<?php
class RateLimitMonitor {
    private array $stats = [];
    
    public function recordRequest(string $endpoint, int $limit, int $remaining, int $reset): void
    {
        $this->stats[$endpoint] = [
            'limit' => $limit,
            'remaining' => $remaining,
            'reset' => $reset,
            'last_updated' => time(),
        ];
    }
    
    public function getUtilization(string $endpoint): float
    {
        if (!isset($this->stats[$endpoint])) {
            return 0.0;
        }
        
        $stats = $this->stats[$endpoint];
        return ($stats['limit'] - $stats['remaining']) / $stats['limit'];
    }
    
    public function shouldThrottle(string $endpoint, float $threshold = 0.8): bool
    {
        return $this->getUtilization($endpoint) >= $threshold;
    }
    
    public function getReport(): array
    {
        $report = [];
        
        foreach ($this->stats as $endpoint => $stats) {
            $report[$endpoint] = [
                'utilization_percent' => round($this->getUtilization($endpoint) * 100, 2),
                'requests_remaining' => $stats['remaining'],
                'resets_at' => date('Y-m-d H:i:s', $stats['reset']),
            ];
        }
        
        return $report;
    }
}
```

## Related Documentation

- [WHMCS API Authentication](/docs/whmcs-api-authentication.md)
- [WHMCS API Error Handling](/docs/whmcs-api-error-handling.md)
- [WHMCS API Batch Operations](/docs/whmcs-api-batch-operations.md)