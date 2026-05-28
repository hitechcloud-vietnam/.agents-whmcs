# WHMCS Vietnamese Payment Integration Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for integrating Vietnamese payment gateways (VNPay, MoMo, payOS, ZaloPay, etc.) into WHMCS.

## Supported Providers

| Provider | Type | Documentation |
|----------|------|---------------|
| VNPay | Bank Transfer, QR | vnpay.vn |
| MoMo | E-wallet | momo.vn |
| payOS | Bank Transfer | payos.vn |
| ZaloPay | E-wallet | zaloPay.vn |
| AlePay | Bank Transfer | alepay.vn |
| SePay | Bank Transfer | seepay.vn |

## VNPay Integration

```php
<?php
// modules/gateways/vnpay/vnpay.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function vnpay_config(): array {
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'VNPay'],
        'vnp_TmnCode' => ['FriendlyName' => 'Terminal ID', 'Type' => 'text'],
        'vnp_HashSecret' => ['FriendlyName' => 'Hash Secret', 'Type' => 'password'],
        'vnp_Url' => ['FriendlyName' => 'API URL', 'Type' => 'text',
            'Default' => 'https://sandbox.vnpay.vn/payv2/vpcpay.vn'],
        'vnp_ReturnUrl' => ['FriendlyName' => 'Return URL', 'Type' => 'text'],
        'testMode' => ['FriendlyName' => 'Test Mode', 'Type' => 'yesno'],
    ];
}

function vnpay_link(array $params): string {
    $tmnCode = $params['vnp_TmnCode'];
    $hashSecret = $params['vnp_HashSecret'];
    $apiUrl = $params['vnp_Url'];
    $returnUrl = $params['vnp_ReturnUrl'];

    // Build payment URL
    $vnp_TxnRef = 'INV' . $params['invoiceid'] . '_' . time();
    $vnp_Amount = (int) ($params['amount'] * 100);
    $vnp_Locale = 'vn';
    $vnp_Curr = 'VND';
    $vnp_OrderInfo = 'Thanh toan hoa don #' . $params['invoiceid'];

    $inputData = [
        'vnp_Version' => '2.1.0',
        'vnp_Command' => 'pay',
        'vnp_TmnCode' => $tmnCode,
        'vnp_Locale' => $vnp_Locale,
        'vnp_Curr' => $vnp_Curr,
        'vnp_TxnRef' => $vnp_TxnRef,
        'vnp_OrderInfo' => $vnp_OrderInfo,
        'vnp_Amount' => $vnp_Amount,
        'vnp_ReturnUrl' => $returnUrl,
        'vnp_IpAddr' => $_SERVER['REMOTE_ADDR'],
        'vnp_CreateDate' => date('YmdHis'),
    ];

    ksort($inputData);
    $hashData = implode('|', array_values($inputData));
    $vnp_SecureHash = strtoupper(hash_hmac('sha512', $hashData, $hashSecret));

    $inputData['vnp_SecureHash'] = $vnp_SecureHash;

    $paymentUrl = $apiUrl . '?' . http_build_query($inputData);

    return '<form action="' . $paymentUrl . '" method="GET">' .
           '<button type="submit" class="btn btn-success">Thanh toan VNPay</button></form>';
}
```

## VNPay Callback

```php
<?php
// modules/gateways/callback/vnpay.php
if (!defined("WHMCS")) { die("Direct access denied"); }

$response = $_GET;

$vnp_SecureHash = $response['vnp_SecureHash'] ?? '';
unset($response['vnp_SecureHash']);

ksort($response);
$hashData = implode('|', array_values($response));

$config = getGatewayConfig();
$secureHash = strtoupper(hash_hmac('sha512', $hashData, $config['vnp_HashSecret']));

if ($secureHash !== $vnp_SecureHash) {
    logTransaction('VNPay', $response, 'Invalid Signature');
    die('Invalid signature');
}

$vnp_ResponseCode = $response['vnp_ResponseCode'] ?? '';
$vnp_TxnRef = $response['vnp_TxnRef'] ?? '';
$vnp_Amount = ($response['vnp_Amount'] ?? 0) / 100;
$vnp_TransactionNo = $response['vnp_TransactionNo'] ?? '';

preg_match('/^INV(\d+)_/', $vnp_TxnRef, $matches);
$invoiceId = (int) ($matches[1] ?? 0);

if ($vnp_ResponseCode === '00') {
    addInvoicePayment($invoiceId, $vnp_TransactionNo, $vnp_Amount, 0, 'VNPay');
    logTransaction('VNPay', $response, 'Success');
} else {
    logTransaction('VNPay', $response, 'Failed: ' . $vnp_ResponseCode);
}

header('Location: ' . \WHMCS\Config\Setting::getValue('SystemURL') . '/viewinvoice.php?id=' . $invoiceId);
```

## MoMo Integration

```php
<?php
// modules/gateways/momo/momo.php
function momo_config(): array {
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'MoMo'],
        'partnerCode' => ['FriendlyName' => 'Partner Code', 'Type' => 'text'],
        'accessKey' => ['FriendlyName' => 'Access Key', 'Type' => 'text'],
        'secretKey' => ['FriendlyName' => 'Secret Key', 'Type' => 'password'],
        'testMode' => ['FriendlyName' => 'Test Mode', 'Type' => 'yesno'],
    ];
}

function momo_link(array $params): string {
    $partnerCode = $params['partnerCode'];
    $accessKey = $params['accessKey'];
    $secretKey = $params['secretKey'];

    $orderId = 'INV' . $params['invoiceid'] . '_' . time();
    $amount = (int) $params['amount'];
    $orderInfo = 'Thanh toan hoa don #' . $params['invoiceid'];
    $returnUrl = $params['returnurl'];
    $notifyUrl = $params['systemurl'] . '/modules/gateways/callback/momo.php';

    $requestId = time() . '';
    $requestType = 'captureWallet';

    $rawData = 'accessKey=' . $accessKey .
               '&orderId=' . $orderId .
               '&merchantCode=' . $partnerCode .
               '&requestId=' . $requestId .
               '&amount=' . $amount .
               '&orderInfo=' . $orderInfo .
               '&returnUrl=' . $returnUrl .
               '&notifyUrl=' . $notifyUrl .
               '&requestType=' . $requestType;

    $signature = hash_hmac('sha256', $rawData, $secretKey);

    $payload = [
        'partnerCode' => $partnerCode,
        'accessKey' => $accessKey,
        'requestId' => $requestId,
        'amount' => $amount,
        'orderId' => $orderId,
        'orderInfo' => $orderInfo,
        'returnUrl' => $returnUrl,
        'notifyUrl' => $notifyUrl,
        'requestType' => $requestType,
        'signature' => $signature,
    ];

    $ch = curl_init('https://test-payment.momo.vn/v2/gateway/api/create');
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);

    $result = json_decode(curl_exec($ch), true);
    curl_close($ch);

    if (isset($result['payUrl'])) {
        return '<form action="' . $result['payUrl'] . '" method="GET">' .
               '<button type="submit" class="btn btn-primary">Thanh toan MoMo</button></form>';
    }

    return 'Loi tao thanh toan MoMo';
}
```

## payOS Integration

```php
<?php
// modules/gateways/payos/payos.php
function payos_config(): array {
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'payOS'],
        'clientId' => ['FriendlyName' => 'Client ID', 'Type' => 'text'],
        'apiKey' => ['FriendlyName' => 'API Key', 'Type' => 'password'],
        'checksumKey' => ['FriendlyName' => 'Checksum Key', 'Type' => 'password'],
        'testMode' => ['FriendlyName' => 'Test Mode', 'Type' => 'yesno'],
    ];
}

function payos_link(array $params): string {
    $clientId = $params['clientId'];
    $apiKey = $params['apiKey'];
    $checksumKey = $params['checksumKey'];

    $orderId = 'INV' . $params['invoiceid'] . '_' . time();
    $amount = (int) $params['amount'];
    $description = 'Thanh toan hoa don #' . $params['invoiceid'];
    $returnUrl = $params['returnurl'];
    $cancelUrl = $params['cancelurl'];

    $data = [
        'clientId' => $clientId,
        'orderCode' => $orderId,
        'amount' => $amount,
        'description' => $description,
        'returnUrl' => $returnUrl,
        'cancelUrl' => $cancelUrl,
    ];

    $signature = generatePayOSSignature($data, $checksumKey);
    $data['signature'] = $signature;

    $ch = curl_init('https://api.payos.vn/v2/payment/create');
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode($data),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER => [
            'Content-Type: application/json',
            'x-client-id: ' . $clientId,
            'x-api-key: ' . $apiKey,
        ],
    ]);

    $result = json_decode(curl_exec($ch), true);
    curl_close($ch);

    if (isset($result['data']['checkoutUrl'])) {
        return '<form action="' . $result['data']['checkoutUrl'] . '" method="GET">' .
               '<button type="submit" class="btn btn-primary">Thanh toan payOS</button></form>';
    }

    return 'Loi tao thanh toan payOS';
}

function generatePayOSSignature(array $data, string $key): string {
    ksort($data);
    $dataStr = implode('|', array_values($data));
    return hash_hmac('sha256', $dataStr, $key);
}
```

## Checklist

- [ ] Signature generation for each provider
- [ ] Callback verification
- [ ] VND currency handling (no decimals)
- [ ] Test mode support
- [ ] Error handling in Vietnamese
- [ ] Transaction logging

---

**Related Skills:**
- whmcs-gateway-builder
- whmcs-callback-handler
- whmcs-vietnamese-billing