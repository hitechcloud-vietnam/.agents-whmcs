# WHMCS API SSO Workflow

## Purpose
Guide developers through implementing Single Sign-On (SSO) with WHMCS API.

## Prerequisites
- WHMCS installation
- SSO protocol understanding
- SSL certificate
- External application

## Steps

### Phase 1: SSO Overview

1. SSO methods
   ```
   WHMCS SSO Options:
   ├── Direct login (API-based)
   ├── OAuth 2.0 (Modern)
   ├── SAML (Enterprise)
   ├── JWT tokens
   └── Session sharing
   ```

2. SSO use cases
   ```
   Common SSO Scenarios:
   - Portal integration
   - WordPress integration
   - Custom applications
   - Third-party platforms
   ```

### Phase 2: API-Based SSO

1. SSO login function
   ```php
   function whmcsSSOLogin($email, $password): ?array {
       $apiUrl = WHMCS_URL . '/includes/api.php';
       
       $ch = curl_init();
       curl_setopt_array($ch, [
           CURLOPT_URL => $apiUrl,
           CURLOPT_POST => true,
           CURLOPT_POSTFIELDS => http_build_query([
               'action' => 'ValidateLogin',
               'email' => $email,
               'password2' => $password,
               'responsetype' => 'json',
           ]),
           CURLOPT_RETURNTRANSFER => true,
       ]);
       
       $response = json_decode(curl_exec($ch), true);
       curl_close($ch);
       
       if ($response['result'] === 'success') {
           // Create session
           return [
               'client_id' => $response['user_id'],
               'email' => $email,
           ];
       }
       
       return null;
   }
   ```

2. SSO session creation
   ```php
   function createWHMCSSession($clientId) {
       // Get client details
       $client = Capsule::table('tblclients')
           ->where('id', $clientId)
           ->first();
       
       // Set session variables
       $_SESSION['uid'] = $clientId;
       $_SESSION['username'] = $client->email;
       $_SESSION['upassword'] = $client->password;
       $_SESSION['clientname'] = $client->firstname . ' ' . $client->lastname;
       
       // Generate session hash
       $hash = md5($client->email . $client->password . session_id());
       $_SESSION['session_hash'] = $hash;
       
       return $hash;
   }
   ```

### Phase 3: JWT-Based SSO

1. JWT token generation
   ```php
   function generateSSOToken($clientId, $expirySeconds = 3600): string {
       $secret = getWHMCSConfig('SSO_SECRET');
       
       $header = [
           'typ' => 'JWT',
           'alg' => 'HS256',
       ];
       
       $payload = [
           'iss' => 'whmcs',
           'sub' => $clientId,
           'iat' => time(),
           'exp' => time() + $expirySeconds,
       ];
       
       $headerEncoded = rtrim(strtr(base64_encode(json_encode($header)), '+/', '-_'), '=');
       $payloadEncoded = rtrim(strtr(base64_encode(json_encode($payload)), '+/', '-_'), '=');
       
       $signature = hash_hmac('sha256', "$headerEncoded.$payloadEncoded", $secret, true);
       $signatureEncoded = rtrim(strtr(base64_encode($signature), '+/', '-_'), '=');
       
       return "$headerEncoded.$payloadEncoded.$signatureEncoded";
   }
   ```

2. JWT token verification
   ```php
   function verifySSOToken($token): ?int {
       $parts = explode('.', $token);
       if (count($parts) !== 3) return null;
       
       [$headerEnc, $payloadEnc, $signature] = $parts;
       
       $secret = getWHMCSConfig('SSO_SECRET');
       $expectedSig = rtrim(strtr(base64_encode(
           hash_hmac('sha256', "$headerEnc.$payloadEnc", $secret, true)
       ), '+/', '-_'), '=');
       
       if (!hash_equals($expectedSig, $signature)) return null;
       
       $payload = json_decode(base64_decode(strtr($payloadEnc, '-_', '+/')), true);
       
       if ($payload['exp'] < time()) return null;
       
       return $payload['sub'];
   }
   ```

### Phase 4: SSO Integration

1. SSO redirect handler
   ```php
   // Generate SSO link for client
   function generateSSOLink($clientId, $returnUrl): string {
       $token = generateSSOToken($clientId, 60); // 60 second expiry
       
       return WHMCS_URL . '/sso.php?token=' . $token . '&return=' . urlencode($returnUrl);
   }
   ```

2. SSO endpoint
   ```php
   // /sso.php
   $token = $_GET['token'] ?? '';
   $returnUrl = $_GET['return'] ?? WHMCS_URL . '/clientarea.php';
   
   $clientId = verifySSOToken($token);
   
   if (!$clientId) {
       header('Location: ' . WHMCS_URL . '/login.php');
       exit;
   }
   
   createWHMCSSession($clientId);
   header('Location: ' . $returnUrl);
   ```

### Phase 5: SSO Security

1. Security measures
   ```
   SSO Security:
   - Short token expiration
   - HTTPS only
   - IP validation
   - Token rotation
   - Audit logging
   ```

2. SSO logging
   ```php
   function logSSOAccess($clientId, $success, $ip, $userAgent) {
       Capsule::table('mod_sso_logs')->insert([
           'client_id' => $clientId,
           'success' => $success,
           'ip_address' => $ip,
           'user_agent' => $userAgent,
           'created_at' => date('Y-m-d H:i:s'),
       ]);
   }
   ```

## Related Workflows
- whmcs-api-authentication
- whmcs-api-oauth
- whmcs-api-integration
