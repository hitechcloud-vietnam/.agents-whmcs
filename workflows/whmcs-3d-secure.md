# WHMCS 3D Secure Setup Workflow

## Description
Implement 3D Secure (Visa/Mastercard Secure) for enhanced payment security.

## Prerequisites
- Payment gateway with 3D Secure support
- SSL certificate (required)
- WHMCS 7.0+

## Steps

### Step 1: Understand 3D Secure
```markdown
3D Secure adds an authentication step:
1. Client enters card details
2. Card network verifies with issuer
3. Client enters OTP/sms code
4. Transaction authenticated
5. Liability shifted to issuer

Benefits:
- Reduced fraud
- Liability protection
- Higher authorization rates
```

### Step 2: Configure 3D Secure in Gateway
```php
<?php
function secure_gateway_config()
{
    return [
        'FriendlyName' => ['value' => 'Secure Payment Gateway'],
        'enable3DSecure' => [
            'Type' => 'yesno',
            'Label' => 'Enable 3D Secure',
            'Description' => 'Require additional authentication',
        ],
        'challengeWindowSize' => [
            'Type' => 'dropdown',
            'Label' => 'Challenge Window',
            'Options' => '250x400,390x400,600x400,fullscreen',
        ],
    ];
}
```

### Step 3: Implement 3DS Flow
```php
<?php
function secure_gateway_link($params)
{
    // Check if 3DS enabled
    $enable3DS = $params['enable3DSecure'] ?? false;
    
    if ($enable3DS) {
        // Initiate 3DS authentication
        $authResult = initiate3D Secure($params);
        
        if ($authResult['redirect']) {
            // Return iframe or redirect for verification
            return '<iframe src="' . $authResult['acsUrl'] . '" 
                width="' . $params['challengeWindowSize'] . '">
            </iframe>';
        }
    }
    
    // Standard payment form
    return '<form>...</form>';
}

function initiate3D Secure($params)
{
    // Call payment provider's 3DS API
    $result = $api->authenticate([
        'amount' => $params['amount'],
        'currency' => $params['currency'],
        'card' => [
            'number' => $_POST['card_number'],
            'exp_month' => $_POST['exp_month'],
            'exp_year' => $_POST['exp_year'],
        ],
        'return_url' => $params['systemurl'] . '/callback',
        'fail_url' => $params['systemurl'] . '/failed',
    ]);
    
    return [
        'redirect' => true,
        'acsUrl' => $result['acs_url'],
        'creq' => $result['creq'],
        'threeDSessionData' => $result['session_data'],
    ];
}
```

### Step 4: Handle 3DS Callback
```php
<?php
// callback.php - Handle 3DS response

function handle3DSecureResponse()
{
    $cres = $_POST['cres'];
    $sessionData = $_SESSION['threeDSessionData'];
    
    // Verify authentication
    $result = verify3D Secure($cres, $sessionData);
    
    if ($result['authenticated']) {
        // Process payment
        $paymentResult = processPayment([
            'amount' => $_SESSION['payment_amount'],
            'authenticated' => true,
            'eci' => $result['eci'],
            'cavv' => $result['cavv'],
        ]);
        
        return $paymentResult;
    } else {
        // Authentication failed
        return ['error' => '3D Secure verification failed'];
    }
}
```

### Step 5: Process with Authentication Data
```php
<?php
function processPayment($params)
{
    // Include 3DS authentication results
    $paymentData = [
        'amount' => $params['amount'],
        'currency' => $params['currency'],
        'payment_token' => $params['token'],
        'three_d_secure' => [
            'authenticated' => $params['authenticated'] ?? false,
            'eci' => $params['eci'] ?? '05',
            'cavv' => $params['cavv'] ?? '',
            'xid' => $params['xid'] ?? '',
        ],
    ];
    
    return $api->charge($paymentData);
}
```

## 3DS ECI Values
| ECI | Description |
|-----|-------------|
| 05 | Successful authentication (Visa/MC) |
| 06 | Authentication attempted |
| 07 | Failed authentication |
| 02 | Successful liability shift (Visa) |
| 01 | Attempted authentication (Visa) |

## Testing 3DS
```bash
# Use test cards that trigger 3DS
# Visa: 4000000000000002
# Mastercard: 5200000000000004

# Test scenarios:
# - Successful authentication
# - Failed authentication
# - Error scenarios
```

## Tags
- 3d-secure
- authentication
- security
- payment