# WHMCS API Timeout Debug Workflow

## Overview
This workflow guides you through debugging API timeout issues in WHMCS modules.

## Prerequisites
- API access logs
- Timeout configuration
- Network monitoring

## Step-by-Step Guide

### Step 1: Check API Timeout Settings
```php
// In your HTTP client configuration
$client = new \GuzzleHttp\Client([
    'connect_timeout' => 10,
    'timeout' => 30,
]);
```

### Step 2: Log API Calls
```php
public function callApi(string $endpoint, array $params): array
{
    $start = microtime(true);
    $url = $this->baseUrl . $endpoint;

    try {
        $response = $this->client->post($url, [
            'form_params' => $params,
            'timeout' => $this->timeout,
        ]);

        $duration = microtime(true) - $start;

        $this->log->debug("API call completed", [
            'endpoint' => $endpoint,
            'duration' => $duration,
            'status' => $response->getStatusCode(),
        ]);

        return json_decode($response->getBody(), true);

    } catch (\GuzzleHttp\Exception\ConnectException $e) {
        $this->log->error("API connection timeout", [
            'endpoint' => $endpoint,
            'error' => $e->getMessage(),
        ]);
        throw $e;

    } catch (\GuzzleHttp\Exception\RequestException $e) {
        $this->log->error("API request failed", [
            'endpoint' => $endpoint,
            'error' => $e->getMessage(),
        ]);
        throw $e;
    }
}
```

### Step 3: Test API Endpoint
```bash
# Test with curl
time curl -v -X POST "https://api.example.com/endpoint" \
    -d "param1=value1" \
    --max-time 30

# Test with verbose timing
curl -w "@timing-format.txt" -o /dev/null -s -X POST \
    "https://api.example.com/endpoint"
```

### Step 4: Implement Retry Logic
```php
public function callWithRetry(string $endpoint, array $params, int $maxRetries = 3): array
{
    $attempt = 0;

    while ($attempt < $maxRetries) {
        try {
            return $this->callApi($endpoint, $params);
        } catch (TimeoutException $e) {
            $attempt++;
            if ($attempt >= $maxRetries) {
                throw $e;
            }
            sleep(pow(2, $attempt)); // Exponential backoff
        }
    }

    throw new Exception("Max retries exceeded");
}
```

## API Timeout Debug Checklist

### Investigation
- [ ] Timeout confirmed
- [ ] Endpoint tested directly
- [ ] Response time measured
- [ ] Server logs checked

### Resolution
- [ ] Timeout increased if needed
- [ ] Retry logic added
- [ ] Circuit breaker implemented
- [ ] Fallback configured

### Monitoring
- [ ] Timeout rate tracked
- [ ] Alert thresholds set
- [ ] Performance monitored
