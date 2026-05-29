# WHMCS EPP Registrar Module - DEVKIT

## Module Information
- **Name**: EPP Registrar (Generic)
- **Version**: 1.0.0
- **Type**: Registrar Module
- **Description**: Generic EPP protocol registrar for custom integrations

## Installation
1. Copy to `/modules/registrars/epp_registrar/`
2. Activate via WHMCS Admin > Configuration > Domain Registrars

## epp_registrar.php
```php
<?php
/**
 * WHMCS EPP Registrar Module
 * 
 * Generic EPP (Extensible Provisioning Protocol) registrar module
 * for custom registry integrations
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function epp_registrar_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'EPP Registrar'
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'Generic EPP Protocol Registrar Module'
        ],
        'server_host' => [
            'FriendlyName' => 'Server Host',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'EPP server hostname'
        ],
        'server_port' => [
            'FriendlyName' => 'Server Port',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '700',
            'Description' => 'EPP server port'
        ],
        'ssl_enabled' => [
            'FriendlyName' => 'SSL/TLS',
            'Type' => 'yesno',
            'Default' => '1',
            'Description' => 'Enable SSL/TLS connection'
        ],
        'registry_clid' => [
            'FriendlyName' => 'Registry Client ID',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'EPP client identifier'
        ],
        'registry_pw' => [
            'FriendlyName' => 'Registry Password',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'EPP client password'
        ],
        'certificate_path' => [
            'FriendlyName' => 'SSL Certificate Path',
            'Type' => 'text',
            'Size' => '100',
            'Description' => 'Path to client SSL certificate'
        ],
        'timeout' => [
            'FriendlyName' => 'Connection Timeout',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '30',
            'Description' => 'Connection timeout in seconds'
        ],
        'default_years' => [
            'FriendlyName' => 'Default Years',
            'Type' => 'text',
            'Size' => '5',
            'Default' => '1'
        ]
    ];
}

class EPPRegistrarConnection
{
    private $socket;
    private $params;
    
    public function __construct($params)
    {
        $this->params = $params;
    }
    
    public function connect()
    {
        $host = $this->params['server_host'];
        $port = $this->params['server_port'];
        $ssl = $this->params['ssl_enabled'];
        
        $protocol = $ssl ? 'ssl' : 'tcp';
        $address = $protocol . '://' . $host . ':' . $port;
        
        $timeout = $this->params['timeout'] ?? 30;
        
        $this->socket = stream_socket_client(
            $address,
            $errno,
            $errstr,
            $timeout
        );
        
        if (!$this->socket) {
            throw new Exception("Connection failed: $errstr ($errno)");
        }
        
        // Read greeting
        $greeting = $this->read();
        
        // Login
        $this->login();
        
        return true;
    }
    
    public function login()
    {
        $clid = $this->params['registry_clid'];
        $pw = $this->params['registry_pw'];
        
        $loginXml = '<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <login>
      <clID>' . htmlspecialchars($clid) . '</clID>
      <pw>' . htmlspecialchars($pw) . '</pw>
      <options>
        <version>1.0</version>
        <lang>en</lang>
      </options>
      <svcs>
        <objURI>urn:ietf:params:xml:ns:domain-1.0</objURI>
        <objURI>urn:ietf:params:xml:ns:contact-1.0</objURI>
      </svcs>
    </login>
  </command>
</epp>';
        
        $this->write($loginXml);
        $response = $this->read();
        
        // Check if login successful
        if (strpos($response, '<result code="1000"') === false) {
            throw new Exception("Login failed");
        }
        
        return true;
    }
    
    public function logout()
    {
        $logoutXml = '<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <logout/>
  </command>
</epp>';
        
        $this->write($logoutXml);
        $this->read();
        
        fclose($this->socket);
    }
    
    private function write($xml)
    {
        $length = strlen($xml);
        $header = pack('N', $length);
        fwrite($this->socket, $header . $xml);
    }
    
    private function read()
    {
        $header = fread($this->socket, 4);
        if (!$header) return '';
        
        $length = unpack('N', $header)[1];
        return fread($this->socket, $length);
    }
    
    public function sendCommand($commandXml)
    {
        $this->write($commandXml);
        return $this->read();
    }
}

function epp_registrar_registerDomain($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    try {
        $conn = new EPPRegistrarConnection($params);
        $conn->connect();
        
        $contactId = epp_createContact($params);
        
        $registerXml = '<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <create>
      <domain:create xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name>' . htmlspecialchars($domain) . '</domain:name>
        <domain:period unit="y">' . (int)$params['regperiod'] . '</domain:period>
        <domain:registrant>' . $contactId . '</domain:registrant>
        <domain:authInfo>
          <domain:pw>' . htmlspecialchars(epp_generateAuthCode()) . '</domain:pw>
        </domain:authInfo>
      </domain:create>
    </command>
  </command>
</epp>';
        
        $response = $conn->sendCommand($registerXml);
        $conn->logout();
        
        if (strpos($response, '<result code="1000"') !== false) {
            return ['success' => true, 'domain' => $domain];
        }
        
        return ['success' => false, 'error' => 'Registration failed'];
        
    } catch (Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function epp_registrar_transferDomain($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    $authCode = $params['transfersecret'];
    
    try {
        $conn = new EPPRegistrarConnection($params);
        $conn->connect();
        
        $transferXml = '<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <transfer op="request">
      <domain:transfer xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name>' . htmlspecialchars($domain) . '</domain:name>
        <domain:authInfo>
          <domain:pw>' . htmlspecialchars($authCode) . '</domain:pw>
        </domain:authInfo>
      </domain:transfer>
    </transfer>
  </command>
</epp>';
        
        $response = $conn->sendCommand($transferXml);
        $conn->logout();
        
        if (strpos($response, '<result code="1000"') !== false) {
            return ['success' => true];
        }
        
        return ['success' => false, 'error' => 'Transfer failed'];
        
    } catch (Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function epp_registrar_renewDomain($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    try {
        $conn = new EPPRegistrarConnection($params);
        $conn->connect();
        
        $renewXml = '<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <renew>
      <domain:renew xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name>' . htmlspecialchars($domain) . '</domain:name>
        <domain:period unit="y">' . (int)$params['regperiod'] . '</domain:period>
      </domain:renew>
    </renew>
  </command>
</epp>';
        
        $response = $conn->sendCommand($renewXml);
        $conn->logout();
        
        if (strpos($response, '<result code="1000"') !== false) {
            return ['success' => true];
        }
        
        return ['success' => false, 'error' => 'Renewal failed'];
        
    } catch (Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function epp_registrar_getNameservers($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    try {
        $conn = new EPPRegistrarConnection($params);
        $conn->connect();
        
        $infoXml = '<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <info>
      <domain:info xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name>' . htmlspecialchars($domain) . '</domain:name>
      </domain:info>
    </command>
  </command>
</epp>';
        
        $response = $conn->sendCommand($infoXml);
        $conn->logout();
        
        // Parse nameservers from response
        preg_match_all('/<domain:ns>(.*?)<\/domain:ns>/s', $response, $matches);
        
        $nservers = [];
        foreach ($matches[1] as $match) {
            preg_match('/<domain:hostObj>(.*?)<\/domain:hostObj>/', $match, $host);
            if (!empty($host[1])) {
                $nservers[] = $host[1];
            }
        }
        
        return [
            'success' => true,
            'ns1' => $nservers[0] ?? '',
            'ns2' => $nservers[1] ?? '',
            'ns3' => $nservers[2] ?? '',
            'ns4' => $nservers[3] ?? ''
        ];
        
    } catch (Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function epp_registrar_saveNameservers($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    try {
        $conn = new EPPRegistrarConnection($params);
        $conn->connect();
        
        $nsList = array_filter([
            $params['ns1'],
            $params['ns2'],
            $params['ns3'],
            $params['ns4'],
            $params['ns5']
        ]);
        
        $nsXml = '';
        foreach ($nsList as $ns) {
            $nsXml .= '<domain:hostObj>' . htmlspecialchars($ns) . '</domain:hostObj>';
        }
        
        $updateXml = '<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <update>
      <domain:update xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name>' . htmlspecialchars($domain) . '</domain:name>
        <domain:add>
          <domain:ns>' . $nsXml . '</domain:ns>
        </domain:add>
      </domain:update>
    </update>
  </command>
</epp>';
        
        $response = $conn->sendCommand($updateXml);
        $conn->logout();
        
        return ['success' => strpos($response, '<result code="1000"') !== false];
        
    } catch (Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function epp_registrar_getDomainInfo($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    try {
        $conn = new EPPRegistrarConnection($params);
        $conn->connect();
        
        $infoXml = '<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <info>
      <domain:info xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name>' . htmlspecialchars($domain) . '</domain:name>
      </domain:info>
    </command>
  </command>
</epp>';
        
        $response = $conn->sendCommand($infoXml);
        $conn->logout();
        
        // Parse dates and status
        preg_match('/<domain:crDate>(.*?)<\/domain:crDate>/', $response, $created);
        preg_match('/<domain:exDate>(.*?)<\/domain:exDate>/', $response, $expiry);
        
        return [
            'registrationdate' => $created[1] ?? '',
            'expirydate' => $expiry[1] ?? '',
            'locked' => strpos($response, '<domain:status s="clientTransferProhibited"') !== false,
            'dnssec' => false
        ];
        
    } catch (Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function epp_registrar_sync($params)
{
    return epp_registrar_getDomainInfo($params);
}

function epp_registrar_getContactDetails($params)
{
    return [
        'Registrant' => [
            'First Name' => $params['firstname'],
            'Last Name' => $params['lastname'],
            'Email' => $params['email']
        ]
    ];
}

function epp_registrar_saveContactDetails($params)
{
    return ['success' => true];
}

function epp_registrar_requestDelete($params)
{
    $domain = $params['sld'] . '.' . $params['tld'];
    
    try {
        $conn = new EPPRegistrarConnection($params);
        $conn->connect();
        
        $deleteXml = '<?xml version="1.0" encoding="UTF-8"?>
<epp xmlns="urn:ietf:params:xml:ns:epp-1.0">
  <command>
    <delete>
      <domain:delete xmlns:domain="urn:ietf:params:xml:ns:domain-1.0">
        <domain:name>' . htmlspecialchars($domain) . '</domain:name>
      </domain:delete>
    </delete>
  </command>
</epp>';
        
        $response = $conn->sendCommand($deleteXml);
        $conn->logout();
        
        return ['success' => strpos($response, '<result code="1000"') !== false];
        
    } catch (Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

// Helper functions
function epp_createContact($params)
{
    return 'C' . time(); // Return generated contact ID
}

function epp_generateAuthCode()
{
    return bin2hex(random_bytes(16));
}
```

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('DomainTransferCompleted', 1, function($vars) {
    logActivity("EPP Registrar transfer completed for: " . $vars['domain']);
});

add_hook('DomainRenewed', 1, function($vars) {
    logActivity("EPP Registrar renewal completed for: " . $vars['domain']);
});
```