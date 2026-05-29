# WHMCS Authorize.Net CIM Setup Workflow

## Description
Configure Authorize.Net Customer Information Manager (CIM) for WHMCS.

## Prerequisites
- Authorize.Net merchant account
- CIM API access
- WHMCS installation

## Steps

### Step 1: Get Authorize.Net Credentials
```bash
# In Authorize.Net Dashboard:
# Account > Settings > API Credentials & Keys
# Get: API Login ID, Transaction Key
# Enable CIM for customer profiles
```

### Step 2: Configure Authorize.Net Gateway
```php
<?php
// modules/gateways/authorizenet_cim/authorizenet_cim.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function authorizenet_cim_MetaData()
{
    return [
        'DisplayName' => 'Authorize.Net CIM',
        'APIVersion' => '1.0',
        'supportsRecurring' => true,
        'supportsCardStorage' => true,
    ];
}

function authorizenet_cim_config()
{
    return [
        'FriendlyName' => ['value' => 'Authorize.Net CIM'],
        'apiLoginId' => ['Type' => 'text', 'Label' => 'API Login ID'],
        'transactionKey' => ['Type' => 'password', 'Label' => 'Transaction Key'],
        'environment' => ['Type' => 'dropdown', 'Options' => 'sandbox,live'],
        'clientId' => ['Type' => 'text', 'Label' => 'Client ID (for OAuth)'],
        'clientSecret' => ['Type' => 'password', 'Label' => 'Client Secret'],
    ];
}

function authorizenet_cim_link($params)
{
    $environment = $params['environment'];
    $apiUrl = $environment === 'sandbox'
        ? 'https://apitest.authorize.net/xml/v1/request.api'
        : 'https://api.authorize.net/xml/v1/request.api';
    
    // Create customer profile
    $customerData = [
        'email' => $params['clientdetails']['email'],
        'description' => 'Customer ' . $params['clientdetails']['userid'],
    ];
    
    // Get/display stored payment methods
    $storedCards = getStoredPaymentMethods($params['clientdetails']['userid']);
    
    $html = '<div class="authorizenet-payment">';
    $html .= '<h4>Select Payment Method</h4>';
    
    if (!empty($storedCards)) {
        foreach ($storedCards as $card) {
            $html .= '<label>';
            $html .= '<input type="radio" name="payment_method" value="' . $card['id'] . '">';
            $html .= ' ****' . $card['last4'] . ' (Exp: ' . $card['exp'] . ')';
            $html .= '</label>';
        }
        $html .= '<label><input type="radio" name="payment_method" value="new"> New Card</label>';
    }
    
    $html .= '<button type="submit" class="btn btn-primary">Pay Now</button>';
    $html .= '</div>';
    
    return $html;
}
```

### Step 3: Create Customer Profile
```php
<?php
function createCustomerProfile($clientId, $email)
{
    $xml = '<?xml version="1.0" encoding="utf-8"?>
    <createCustomerProfileRequest xmlns="AnetApi/xml/v1/schema/AnetApiSchema.xsd">
        <merchantAuthentication>
            <name>' . $apiLoginId . '</name>
            <transactionKey>' . $transactionKey . '</transactionKey>
        </merchantAuthentication>
        <profile>
            <email>' . $email . '</email>
            <description>Customer Profile for WHMCS User ' . $clientId . '</description>
        </profile>
    </createCustomerProfileRequest>';
    
    $response = sendToAuthorize($xml);
    return parseResponse($response);
}
```

### Step 4: Store Payment Method
```php
<?php
function createPaymentProfile($customerProfileId, $cardNumber, $expMonth, $expYear)
{
    $xml = '<?xml version="1.0" encoding="utf-8"?>
    <createCustomerPaymentProfileRequest xmlns="AnetApi/xml/v1/schema/AnetApiSchema.xsd">
        <merchantAuthentication>
            <name>' . $apiLoginId . '</name>
            <transactionKey>' . $transactionKey . '</transactionKey>
        </merchantAuthentication>
        <customerProfileId>' . $customerProfileId . '</customerProfileId>
        <paymentProfile>
            <payment>
                <creditCard>
                    <cardNumber>' . $cardNumber . '</cardNumber>
                    <expirationDate>' . $expYear . '-' . $expMonth . '</expirationDate>
                </creditCard>
            </payment>
        </paymentProfile>
    </createCustomerPaymentProfileRequest>';
    
    return sendToAuthorize($xml);
}
```

## Authorize.Net CIM Features
- Store customer profiles
- Store multiple payment methods
- Process one-time payments
- Process recurring payments
- Update payment methods

## Tags
- authorize.net
- cim
- payment
- tokenization