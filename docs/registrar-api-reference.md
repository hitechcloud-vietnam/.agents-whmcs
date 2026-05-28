# WHMCS Domain Registrar API Reference
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Complete reference for registrar module API patterns.

## API Patterns by Registrar

### EPP (Extensible Provisioning Protocol)
```php
interface RegistrarEPP {
    public function login(string $id, string $password): bool;
    public function logout(): bool;
    public function check_domain(string $domain): bool;
    public function register_domain(array $params): bool;
    public function transfer_domain(array $params): bool;
    public function renew_domain(array $params): bool;
    public function get_info(string $domain): array;
    public function update_domain(string $domain, array $data): bool;
}
```

### REST Registrar API
```php
class RESTRegistrar {
    private string $baseUrl = 'https://api.registrar.com/v1';

    public function registerDomain(array $data): array {
        return $this->request('POST', '/domains', [
            'name' => $data['domain'],
            'period' => $data['years'],
            'contacts' => $data['contacts'],
        ]);
    }

    public function transferDomain(array $data): array {
        return $this->request('POST', '/transfers', [
            'domain' => $data['domain'],
            'auth_code' => $data['eppcode'],
        ]);
    }

    public function renewDomain(array $data): array {
        return $this->request('POST', '/domains/' . $data['domain'] . '/renew', [
            'period' => $data['years'],
        ]);
    }

    private function request(string $method, string $endpoint, array $data = []): array {
        // Implementation
    }
}
```

### SOAP Registrar API
```php
class SOAPRegistrar {
    private SoapClient $client;

    public function __construct(string $wsdl, array $options) {
        $this->client = new SoapClient($wsdl, $options);
    }

    public function call(string $method, array $args = []): mixed {
        return $this->client->__call($method, $args);
    }
}
```

## Common Features

### Nameserver Management
```php
function saveNameservers(string $domain, array $nameservers): bool {
    return $this->call('domain nsupdate', [
        'domain' => $domain,
        'nameservers' => $nameservers,
    ]);
}
```

### DNSSEC
```php
function getDnsSec(string $domain): array {
    return $this->call('domain dnsseckeys get', ['domain' => $domain]);
}

function setDnsSec(string $domain, array $keys): bool {
    return $this->call('domain dnsseckeys add', [
        'domain' => $domain,
        'keys' => $keys,
    ]);
}
```

---

**Related Skills:**
- whmcs-registrar-builder
- whmcs-domain-sync
