# WHMCS HTTP Client

Complete guide to HTTP client implementation.

## Overview

Build robust HTTP clients for external integrations.

## HTTP Client

```php
<?php
/**
 * HTTP client
 */
class HTTPClient
{
    private string $baseUrl;
    private array $headers = [];
    private int $timeout = 30;
    
    public function __construct(string $baseUrl = '')
    {
        $this->baseUrl = rtrim($baseUrl, '/');
    }
    
    /**
     * Set header
     */
    public function setHeader(string $name, string $value): self
    {
        $this->headers[$name] = $value;
        return $this;
    }
    
    /**
     * Set timeout
     */
    public function setTimeout(int $seconds): self
    {
        $this->timeout = $seconds;
        return $this;
    }
    
    /**
     * GET request
     */
    public function get(string $path, array $params = []): array
    {
        $url = $this->baseUrl . $path;
        
        if (!empty($params)) {
            $url .= '?' . http_build_query($params);
        }
        
        return $this->request('GET', $url);
    }
    
    /**
     * POST request
     */
    public function post(string $path, array $data = []): array
    {
        $url = $this->baseUrl . $path;
        
        return $this->request('POST', $url, $data);
    }
    
    /**
     * Make request
     */
    private function request(string $method, string $url, array $data = []): array
    {
        $ch = curl_init();
        
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => $this->buildHeaders(),
        ]);
        
        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        
        curl_close($ch);
        
        return [
            'status' => $httpCode,
            'body' => json_decode($response, true) ?? $response,
            'error' => $error,
        ];
    }
    
    /**
     * Build headers
     */
    private function buildHeaders(): array
    {
        $headers = ['Content-Type: application/json'];
        
        foreach ($this->headers as $name => $value) {
            $headers[] = "{$name}: {$value}";
        }
        
        return $headers;
    }
}
```

## Best Practices

1. **Timeout handling** - Set appropriate timeouts
2. **Retry logic** - Handle transient failures
3. **Error handling** - Check all error conditions
4. **Headers** - Set appropriate headers
5. **Logging** - Log request/response
6. **Testing** - Mock HTTP responses

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-advanced-security.md](whmcs-advanced-security.md)
