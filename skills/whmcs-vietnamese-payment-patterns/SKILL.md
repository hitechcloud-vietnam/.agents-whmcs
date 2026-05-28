# WHMCS Vietnamese Payment Patterns Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Specific patterns for Vietnamese payment gateways.

## VNPay Integration

```php
<?php
class VNPay {
    private string $vnpUrl = 'https://sandbox.vnpay.vn/payv2/vpcpay.html';
    private string $vnpReturnUrl;
    private string $vnpTmnCode;
    private string $vnpHashSecret;

    public function createPayment(int $invoiceId, float $amount, string $returnUrl): string {
        $vnp_Params = [
            'vnp_Version' => '2.1.0',
            'vnp_Command' => 'pay',
            'vnp_TmnCode' => $this->vnpTmnCode,
            'vnp_Amount' => $amount * 100, // VND in cents
            'vnp_CreateDate' => date('YmdHis'),
            'vnp_CurrCode' => 'VND',
            'vnp_IpAddr' => $_SERVER['REMOTE_ADDR'],
            'vnp_Locale' => 'vn',
            'vnp_OrderInfo' => 'Thanh toan hoa don #' . $invoiceId,
            'vnp_OrderType' => 'billpayment',
            'vnp_ReturnUrl' => $returnUrl,
            'vnp_TxnRef' => $invoiceId,
        ];

        ksort($vnp_Params);
        $signData = http_build_query($vnp_Params);
        $vnpSecureHash = hash_hmac('sha256', $signData, $this->vnpHashSecret);

        return $this->vnpUrl . '?' . $signData . '&vnp_SecureHash=' . $vnpSecureHash;
    }

    public function verifyReturn(array $data): bool {
        $vnp_HashSecret = $data['vnp_SecureHash'] ?? '';
        unset($data['vnp_SecureHash']);

        ksort($data);
        $signData = http_build_query($data);
        $secureHash = hash_hmac('sha256', $signData, $this->vnpHashSecret);

        return hash_equals($secureHash, $vnp_HashSecret);
    }
}
```

## MoMo Integration

```php
<?php
class MoMo {
    private string $endpoint = 'https://test-payment.momo.vn/v2/gateway`;

    public function createPayment(int $invoiceId, float $amount, string $returnUrl): array {
        $requestId = time() . '';
        $requestType = 'captureWallet';

        $rawData = "accessKey={$this->accessKey}&amount={$amount}&orderId={$invoiceId}&orderInfo=Thanh+toan+hoa+don&partnerCode={$this->partnerCode}&requestId={$requestId}&requestType={$requestType}";

        $signature = hash_hmac('sha256', $rawData, $this->secretKey);

        $data = [
            'partnerCode' => $this->partnerCode,
            'partnerName' => 'WHMCSPayment',
            'storeId' => 'WHMCS',
            'requestId' => $requestId,
            'amount' => $amount,
            'orderId' => $invoiceId,
            'orderInfo' => 'Thanh toan hoa don',
            'requestType' => $requestType,
            'signature' => $signature,
            'returnUrl' => $returnUrl,
            'notifyUrl' => $this->notifyUrl,
        ];

        $result = $this->callApi('/create', $data);

        return [
            'payUrl' => $result['payUrl'],
            'deeplink' => $result['deeplink'] ?? null,
        ];
    }
}
```

## payOS Integration

```php
<?php
class PayOS {
    private string $apiUrl = 'https://payos.vn/v1/payment';

    public function createPayment(int $invoiceId, float $amount, string $returnUrl): array {
        $orderCode = time() . $invoiceId;

        $data = [
            'orderCode' => $orderCode,
            'amount' => (int) $amount,
            'description' => 'Thanh toan hoa don ' . $invoiceId,
            'returnUrl' => $returnUrl,
            'cancelUrl' => $returnUrl,
            'signature' => $this->generateSignature($orderCode, (int) $amount),
        ];

        $result = $this->callApi('/orders', $data);

        return [
            'checkoutUrl' => $result['data']['checkoutUrl'],
            'orderCode' => $orderCode,
        ];
    }

    private function generateSignature(string $orderCode, int $amount): string {
        $data = "{$orderCode}|{$amount}|{$this->checksumKey}";
        return hash_hmac('sha256', $data, $this->checksumKey);
    }
}
```

---

**Related Skills:**
- whmcs-gateway-builder
- whmcs-callback-handler
