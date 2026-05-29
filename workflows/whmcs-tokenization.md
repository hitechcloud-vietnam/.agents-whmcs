# WHMCS Tokenization Setup Workflow

## Description
Implement card tokenization for secure payment storage in WHMCS.

## Prerequisites
- Payment gateway with tokenization support
- PCI compliance considerations
- WHMCS 7.0+

## Steps

### Step 1: Understand Tokenization
```markdown
Tokenization replaces sensitive card data with tokens:
- Card Number: 4111111111111111
- Token: tok_abc123xyz789

Benefits:
- No card data stored on your server
- Reduced PCI scope
- Faster checkout for returning customers
```

### Step 2: Create Tokenized Gateway
```php
<?php
// modules/gateways/tokenized/tokenized.php

function tokenized_MetaData()
{
    return [
        'DisplayName' => 'Tokenized Payments',
        'APIVersion' => '1.0',
        'TokenisedStorageAllowed' => true, // Enable storage
    ];
}

function tokenized_config()
{
    return [
        'FriendlyName' => ['value' => 'Tokenized Payments'],
        'publicKey' => ['Type' => 'text', 'Label' => 'Public Key'],
        'secretKey' => ['Type' => 'password', 'Label' => 'Secret Key'],
    ];
}
```

### Step 3: Create Token
```php
<?php
function createCardToken($cardNumber, $expMonth, $expYear, $cvv)
{
    // Use payment provider's tokenization API
    $response = $api->createToken([
        'card' => [
            'number' => $cardNumber,
            'exp_month' => $expMonth,
            'exp_year' => $expYear,
            'cvc' => $cvv,
        ],
    ]);
    
    return [
        'token' => $response['id'],
        'last4' => substr($cardNumber, -4),
        'exp_month' => $expMonth,
        'exp_year' => $expYear,
        'brand' => detectCardBrand($cardNumber),
    ];
}
```

### Step 4: Store Token
```php
<?php
function storeCardToken($clientId, $tokenData)
{
    // Store in WHMCS database
    Capsule::table('mod_stored_tokens')->insert([
        'client_id' => $clientId,
        'gateway' => 'tokenized',
        'token' => $tokenData['token'],
        'last4' => $tokenData['last4'],
        'brand' => $tokenData['brand'],
        'exp_month' => $tokenData['exp_month'],
        'exp_year' => $tokenData['exp_year'],
        'is_default' => 0,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}
```

### Step 5: Process with Token
```php
<?php
function chargeWithToken($clientId, $amount, $currency)
{
    // Get stored token
    $token = Capsule::table('mod_stored_tokens')
        ->where('client_id', $clientId)
        ->where('is_default', 1)
        ->first();
    
    if (!$token) {
        return ['error' => 'No stored payment method'];
    }
    
    // Charge using token
    $result = $api->charge([
        'amount' => $amount,
        'currency' => $currency,
        'payment_token' => $token->token,
    ]);
    
    return $result;
}
```

### Step 6: Delete Token
```php
<?php
function deleteCardToken($tokenId)
{
    $token = Capsule::table('mod_stored_tokens')->find($tokenId);
    
    // Notify gateway to delete token
    $api->deleteToken($token->token);
    
    // Remove from database
    Capsule::table('mod_stored_tokens')
        ->where('id', $tokenId)
        ->delete();
}
```

## Tokenization Flow
```
1. Client enters card details
2. Card data sent directly to payment provider (not your server)
3. Provider returns token
4. Token stored in WHMCS
5. Future charges use token
6. Original card data never touches your server
```

## Security Benefits
- PCI DSS scope reduction
- No card data on your servers
- Compliance made easier
- Customer trust

## Tags
- tokenization
- security
- pci-compliance
- payment