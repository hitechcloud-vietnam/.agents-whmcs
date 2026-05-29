# WHMCS API Client Creation Workflow

## Purpose
Guide developers through creating clients via WHMCS API.

## Prerequisites
- WHMCS installation
- Client API access
- PHP skills

## Steps

### Phase 1: Client Creation API

1. Create client
   ```php
   function createClient($data): array {
       return localAPI('AddClient', [
           'firstname' => $data['firstname'],
           'lastname' => $data['lastname'],
           'email' => $data['email'],
           'companyname' => $data['companyname'] ?? '',
           'phonenumber' => $data['phonenumber'] ?? '',
           'address1' => $data['address1'] ?? '',
           'city' => $data['city'] ?? '',
           'state' => $data['state'] ?? '',
           'postcode' => $data['postcode'] ?? '',
           'country' => $data['country'] ?? 'US',
           'password' => $data['password'] ?? generateRandomPassword(),
           'clientip' => $_SERVER['REMOTE_ADDR'],
       ]);
   }
   ```

2. Full client creation example
   ```php
   $result = localAPI('AddClient', [
       'firstname' => 'John',
       'lastname' => 'Doe',
       'email' => 'john.doe@example.com',
       'companyname' => 'Acme Inc',
       'phonenumber' => '+1234567890',
       'address1' => '123 Main Street',
       'address2' => 'Suite 100',
       'city' => 'New York',
       'state' => 'NY',
       'postcode' => '10001',
       'country' => 'US',
       'password' => 'SecurePassword123!',
       'currency' => 1,
   ]);
   
   if ($result['result'] === 'success') {
       $clientId = $result['clientid'];
   }
   ```

### Phase 2: Client Validation

1. Email verification
   ```php
   function validateEmail($email): bool {
       return filter_var($email, FILTER_VALIDATE_EMAIL) !== false;
   }
   ```

2. Check existing client
   ```php
   function clientExists($email): bool {
       $existing = localAPI('GetClients', [
           'search' => $email,
           'limitnum' => 1,
       ]);
       
       return !empty($existing['clients']['client']);
   }
   ```

## Related Workflows
- whmcs-api-user-sync
- whmcs-api-ticket-manage
- whmcs-api-integration
