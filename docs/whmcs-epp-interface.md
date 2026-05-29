# WHMCS EPP Interface Documentation

## Overview

The Extensible Provisioning Protocol (EPP) is the standard protocol for communication between registrars and domain registries. WHMCS provides comprehensive EPP interface support for domain operations.

## EPP Protocol Basics

### Protocol Characteristics

- XML-based client-server communication
- TCP/IP transport layer (typically port 700)
- TLS/SSL encryption required
- Synchronous request-response model

### EPP Command Types

| Command | Description | WHMCS Action |
|---------|-------------|--------------|
| Hello | Connection initialization | Authenticate |
| Login | Session establishment | Authenticate credentials |
| Logout | Session termination | Close connection |
| Check | Domain availability | Check availability |
| Info | Domain details | Get domain info |
| Create | Domain registration | Register domain |
| Delete | Domain removal | Delete/expire domain |
| Renew | Registration extension | Renew domain |
| Transfer | Registrar change | Process transfer |
| Update | Modify domain data | Update nameservers |
| Poll | Notification queue | Fetch notifications |

## WHMCS EPP Implementation

### Connection Configuration

```php
// EPP Connection Settings
$eppConfig = [
    'host' => 'epp.registry.tld',
    'port' => 700,
    'timeout' => 30,
    'ssl' => true,
    'local_cert' => '/path/to/client.crt',
    'local_pk' => '/path/to/client.key',
    'ca_cert' => '/path/to/ca.crt',
    'debug' => false
];
```

### EPP Service Class

```php
namespace WHMCS\Module\Registrar\Epp;

class EppConnection
{
    protected $socket;
    protected $connected = false;

    /**
     * Establish EPP connection
     */
    public function connect(): bool
    {
        $context = stream_context_create([
            'ssl' => [
                'local_cert' => $this->config['local_cert'],
                'local_pk' => $this->config['local_pk'],
                'cafile' => $this->config['ca_cert'],
                'verify_peer' => true,
                'verify_peer_name' => true
            ]
        ]);

        $this->socket = stream_socket_client(
            sprintf(
                'ssl://%s:%d',
                $this->config['host'],
                $this->config['port']
            ),
            $errno,
            $errstr,
            $this->config['timeout'],
            STREAM_CLIENT_CONNECT,
            $context
        );

        return $this->connected = ($this->socket !== false);
    }

    /**
     * Send EPP command
     */
    public function sendCommand(string $xml): array
    {
        // Send command
        $length = strlen($xml);
        fwrite($this->socket, pack('N', $length) . $xml);

        // Read response
        $response = $this->readResponse();

        return $this->parseResponse($response);
    }
}
```

## EPP Commands Reference

### Domain Check

```xml
<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <check>
      <domain:check xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name>example.com</domain:name>
        <domain:name>example.net</domain:name>
        <domain:name>example.org</domain:name>
      </domain:check>
    </check>
    <clTRID>ABC-12345</clTRID>
  </command>
</epp>
```

**WHMCS Response Handling:**

```php
public function checkAvailability(array $domains): array
{
    $results = [];

    foreach ($domains as $domain) {
        $command = $this->buildCheckCommand($domain);
        $response = $this->sendCommand($command);

        $results[$domain] = [
            'available' => ($response['code'] === 1000),
            'reason' => $response['reason'] ?? null
        ];
    }

    return $results;
}
```

### Domain Info

```xml
<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <info>
      <domain:info xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name hosts="all">example.com</domain:name>
      </domain:info>
    </info>
    <clTRID>ABC-12346</clTRID>
  </command>
</epp>
```

### Domain Create (Registration)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <create>
      <domain:create xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name>example.com</domain:name>
        <domain:period unit="y">2</domain:period>
        <domain:ns>
          <domain:hostObj>ns1.registrar.com</domain:hostObj>
          <domain:hostObj>ns2.registrar.com</domain:hostObj>
        </domain:ns>
        <domain:registrant>sh8013</domain:registrant>
        <domain:authInfo>
          <domain:pw>2fooBAR</domain:pw>
        </domain:authInfo>
      </domain:create>
    </create>
    <clTRID>ABC-12347</clTRID>
  </command>
</epp>
```

### Domain Renew

```xml
<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <renew>
      <domain:renew xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name>example.com</domain:name>
        <domain:curExpDate>2024-01-15</domain:curExpDate>
        <domain:period unit="y">1</domain:period>
      </domain:renew>
    </renew>
    <clTRID>ABC-12348</clTRID>
  </command>
</epp>
```

### Domain Transfer

```xml
<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <transfer op="request">
      <domain:transfer xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name>example.com</domain:name>
        <domain:authInfo>
          <domain:pw>domainpw</domain:pw>
        </domain:authInfo>
      </domain:transfer>
    </transfer>
    <clTRID>ABC-12349</clTRID>
  </command>
</epp>
```

### Domain Update (Nameserver Change)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <update>
      <domain:update xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name>example.com</domain:name>
        <domain:add>
          <domain:ns>
            <domain:hostObj>ns3.newregistrar.com</domain:hostObj>
          </domain:ns>
        </domain:add>
        <domain:rem>
          <domain:ns>
            <domain:hostObj>ns1.oldregistrar.com</domain:hostObj>
          </domain:ns>
        </domain:rem>
      </domain:update>
    </update>
    <clTRID>ABC-12350</clTRID>
  </command>
</epp>
```

## EPP Response Codes

### Success Codes (1xxx)

| Code | Meaning |
|------|---------|
| 1000 | Command completed successfully |
| 1001 | Command completed; action pending |
| 1300 | No messages received |
| 1301 | Messages received |

### Error Codes (2xxx)

| Code | Meaning |
|------|---------|
| 2000 | Unknown command |
| 2001 | Command syntax error |
| 2002 | Command use error |
| 2100 | Authentication error |
| 2200 | Authorization error |
| 2300 | Object does not exist |
| 2301 | Object exists |
| 2302 | Object status prohibits |
| 2303 | Object association |
| 2400 | Command failed |

## Contact Objects

### Contact Create

```xml
<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <create>
      <contact:create xmlns:contact="urn:ietf:params:xml:ns:contact-1.0">
        <contact:id>sh8013</contact:id>
        <contact:asciiContact type="int">
          <contact:name>John Doe</contact:name>
          <contact:org>Example Inc.</contact:org>
          <contact:addr>
            <contact:street>123 Main Street</contact:street>
            <contact:city>Anytown</contact:city>
            <contact:sp>CA</contact:sp>
            <contact:pc>12345</contact:pc>
            <contact:cc>US</contact:cc>
          </contact:addr>
        </contact:asciiContact>
        <contact:voice>+1.5551234567</contact:voice>
        <contact:email>jdoe@example.com</contact:email>
      </contact:create>
    </create>
  </command>
</epp>
```

## Security

### TLS Configuration

```php
// Secure TLS configuration
$context = stream_context_create([
    'ssl' => [
        'local_cert' => CERT_PATH . 'client.crt',
        'local_pk' => CERT_PATH . 'client.key',
        'cafile' => CERT_PATH . 'registry-ca.crt',
        'peer_name' => 'epp.registry.tld',
        'verify_peer' => true,
        'verify_peer_name' => true,
        'ciphers' => 'TLSv1.2:TLSv1.3',
        'disable_compression' => true
    ]
]);
```

### Connection Pooling

Implement connection pooling for efficiency:

```php
class EppConnectionPool
{
    private $maxConnections = 10;
    private $connections = [];
    private $available = [];

    public function getConnection(): EppConnection
    {
        if (!empty($this->available)) {
            return array_pop($this->available);
        }

        if (count($this->connections) < $this->maxConnections) {
            $conn = new EppConnection();
            $conn->connect();
            $this->connections[] = $conn;
            return $conn;
        }

        // Wait for available connection
        return $this->waitForConnection();
    }

    public function releaseConnection(EppConnection $conn)
    {
        if ($conn->isHealthy()) {
            $this->available[] = $conn;
        }
    }
}
```

## Error Handling

### Retry Logic

```php
public function sendWithRetry(string $xml, int $maxRetries = 3): array
{
    $attempt = 0;
    $backoff = 1;

    while ($attempt < $maxRetries) {
        try {
            return $this->sendCommand($xml);
        } catch (EppException $e) {
            $attempt++;
            if ($attempt >= $maxRetries) {
                throw $e;
            }
            sleep($backoff);
            $backoff *= 2;
            $this->reconnect();
        }
    }
}
```

### Error Mapping

```php
// Map EPP codes to WHMCS errors
private function mapError(int $code): string
{
    $errors = [
        2001 => 'Invalid command syntax',
        2100 => 'Authentication failed - check credentials',
        2200 => 'Insufficient permissions',
        2300 => 'Domain not found',
        2301 => 'Domain already exists',
        2302 => 'Domain status prohibits operation',
        2303 => 'Domain has dependent objects',
        2400 => 'Command failed - contact support'
    ];

    return $errors[$code] ?? 'Unknown EPP error';
}
```

## Logging

### EPP Transaction Logging

```php
// Log all EPP transactions
public function logTransaction(string $command, string $xml, array $response): void
{
    $log = [
        'timestamp' => date('Y-m-d H:i:s'),
        'command' => $command,
        'request' => $xml,
        'response_code' => $response['code'],
        'response_message' => $response['message'] ?? null,
        'duration_ms' => $this->getLastDuration()
    ];

    logActivity('EPP Transaction: ' . json_encode($log));
}
```

## See Also

- [Registrar Commands](./whmcs-registrar-commands.md)
- [Domain Transfer Tool](./whmcs-transfer-tool.md)
- [Sync Daemon Configuration](./whmcs-sync-daemon.md)
