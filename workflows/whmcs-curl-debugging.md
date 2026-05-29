# WHMCS cURL Debug Workflow

## Overview
This workflow guides you through debugging cURL/HTTP issues in WHMCS modules.

## Prerequisites
- cURL installed
- HTTP debugging tools

## Step-by-Step Guide

### Step 1: Enable Verbose cURL Logging
```php
public function makeRequest(string $url, array $params = []): array
{
    $ch = curl_init();

    curl_setopt_array($ch, [
        CURLOPT_URL => $url,
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => http_build_query($params),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_VERBOSE => true,
        CURLOPT_STDERR => fopen('/tmp/curl_verbose.log', 'w+'),
    ]);

    $response = curl_exec($ch);
    $error = curl_error($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);

    curl_close($ch);

    // Log details
    $this->log->debug("cURL request completed", [
        'url' => $url,
        'http_code' => $httpCode,
        'response_length' => strlen($response),
        'error' => $error,
    ]);

    return json_decode($response, true);
}
```

### Step 2: Test with curl CLI
```bash
# Basic request
curl -v "https://api.example.com/endpoint"

# POST request
curl -v -X POST "https://api.example.com/endpoint" \
    -d "param1=value1" \
    -d "param2=value2"

# With headers
curl -v -X POST "https://api.example.com/endpoint" \
    -H "Authorization: Bearer TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"key": "value"}'

# Check SSL
curl -v --ssl-reqd "https://api.example.com/endpoint"
```

### Step 3: Common cURL Issues
```php
// Issue: SSL certificate error
// Fix:
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
curl_setopt($ch, CURLOPT_CAINFO, '/path/to/ca-bundle.crt');

// Issue: Timeout
// Fix:
curl_setopt($ch, CURLOPT_TIMEOUT, 30);
curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, 10);

// Issue: Following redirects
// Fix:
curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
curl_setopt($ch, CURLOPT_MAXREDIRS, 5);
```

## cURL Debug Checklist

### Investigation
- [ ] Verbose logging enabled
- [ ] Request tested with curl CLI
- [ ] SSL verified
- [ ] Response captured

### Resolution
- [ ] SSL certificates updated
- [ ] Timeout increased
- [ ] Headers corrected
- [ ] Redirects handled
