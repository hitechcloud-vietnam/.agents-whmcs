# WHMCS Payment Gateway Types Reference
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Reference for all WHMCS payment gateway types.

## Gateway Type 1: Standard Redirect

```php
function {gateway}_link(array $params): string {
    // Returns HTML form that submits to payment provider
    return '<form action="https://provider.com/pay" method="POST">
        <input type="hidden" name="order" value="' . $params['invoiceid'] . '">
        <input type="submit" value="Pay Now">
    </form>';
}
```

**Use case:** Redirect to payment page (QR codes, bank transfer)

---

## Gateway Type 2: Merchant/Capture

```php
function {gateway}_capture_form(array $params): string {
    // Returns HTML form for card input
    return '<form>
        <input type="text" name="card_number">
        <input type="text" name="expiry">
        <input type="text" name="cvv">
        <input type="submit">
    </form>';
}

function {gateway}_capture(array $params): array {
    // Process payment
    return ['status' => 'success', 'transid' => 'xxx'];
}
```

**Use case:** Card is captured and processed immediately. PCI compliant.

---

## Gateway Type 3: Tokenization

```php
function {gateway}_capture_form(array $params): string {
    return '<form>Renders hosted fields</form>';
}

function {gateway}_capture_token(array $params): array {
    // Store token for later use
    return ['status' => 'success', 'token' => 'tok_xxx'];
}

function {gateway}_capture(array $params): array {
    // Charge using stored token
    return ['status' => 'success'];
}
```

**Use case:** Tokenize card, charge later (subscription, auto-billing)

---

## Gateway Type 4: Remote Input (iFrame)

```php
function {gateway}_remote_input(array $params): array {
    return [
        'provider' => 'https:// hosted-fields-provider.com',
        'client_token' => 'token_xxx',
    ];
}
```

**Use case:** Hosted payment form in iframe. Most PCI compliant.

---

## Gateway Type 5: Remote Bank

```php
function {gateway}_remote_bank(array $params): array {
    return [
        'banklist' => ['bank1', 'bank2', 'bank3'],
        'banks' => [...],
    ];
}

// Callback in callback/{gateway}.php handles bank selection
```

**Use case:** Bank selection with external validation (Vietnamese banks)

---

## Callback Patterns

### Standard Callback
```php
function {gateway}_callback() {
    $data = $_POST;
    // Validate signature
    // Process payment
    logTransaction();
    addInvoicePayment();
}
```

### Remote Bank Callback
```php
// Uses IPN-style callback
// Bank notifies WHMCS of payment status
```

---

**Related Skills:**
- whmcs-gateway-builder
- whmcs-vietnamese-payment-builder
