# WHMCS Encryption Functions

Complete reference for encryption and security functions in WHMCS.

## Overview

WHMCS provides comprehensive encryption utilities for securing sensitive data including passwords, personal information, and API keys.

## Password Functions

### hashPassword()

Hashes a password using bcrypt.

```php
/**
 * Hash a password
 * 
 * @param string $password Plain text password
 * @param int $cost Cost factor (default 12)
 * @return string Hashed password
 */
function hashPassword(string $password, int $cost = 12): string
{
    return password_hash($password, PASSWORD_BCRYPT, [
        'cost' => $cost
    ]);
}
```

**Example:**
```php
$hashed = hashPassword('MySecurePassword123');
// Returns something like: $2y$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/X4.VTt1W6HN3.ZKGe
```

### verifyPassword()

Verifies a password against a hash.

```php
/**
 * Verify password
 * 
 * @param string $password Plain text password
 * @param string $hash Hash to verify against
 * @return bool Valid status
 */
function verifyPassword(string $password, string $hash): bool
{
    return password_verify($password, $hash);
}
```

**Example:**
```php
$storedHash = '$2y$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/X4.VTt1W6HN3.ZKGe';

if (verifyPassword('MySecurePassword123', $storedHash)) {
    echo 'Password is correct';
}
```

### needsRehash()

Checks if password needs rehashing.

```php
/**
 * Check if password needs rehashing
 * 
 * @param string $hash Hash to check
 * @param int $cost Cost factor
 * @return bool Needs rehash
 */
function needsRehash(string $hash, int $cost = 12): bool
{
    return password_needs_rehash($hash, PASSWORD_BCRYPT, ['cost' => $cost]);
}
```

**Example:**
```php
if (needsRehash($storedHash)) {
    // Rehash and update in database
    $newHash = hashPassword($password);
    // Update in database
}
```

## Encryption Functions

### encrypt()

Encrypts data using AES-256-CBC.

```php
/**
 * Encrypt data
 * 
 * @param string $data Data to encrypt
 * @param string|null $key Encryption key (null = use system key)
 * @return string Encrypted data (base64 encoded)
 */
function encrypt(string $data, ?string $key = null): string
{
    $key = $key ?: Config\Setting::getValue('EncryptionKey');
    
    if (empty($key)) {
        throw new Exception('Encryption key not configured');
    }
    
    // Generate IV
    $iv = random_bytes(16);
    
    // Encrypt
    $encrypted = openssl_encrypt(
        $data,
        'AES-256-CBC',
        hash('sha256', $key, true),
        OPENSSL_RAW_DATA,
        $iv
    );
    
    // Combine IV and encrypted data
    $combined = $iv . $encrypted;
    
    return base64_encode($combined);
}
```

**Example:**
```php
$encrypted = encrypt('Sensitive data here');
// Returns: base64 encoded string containing IV + encrypted data
```

### decrypt()

Decrypts data.

```php
/**
 * Decrypt data
 * 
 * @param string $encryptedData Encrypted data (base64 encoded)
 * @param string|null $key Encryption key (null = use system key)
 * @return string Decrypted data
 */
function decrypt(string $encryptedData, ?string $key = null): string
{
    $key = $key ?: Config\Setting::getValue('EncryptionKey');
    
    if (empty($key)) {
        throw new Exception('Encryption key not configured');
    }
    
    // Decode from base64
    $combined = base64_decode($encryptedData);
    
    // Extract IV and encrypted data
    $iv = substr($combined, 0, 16);
    $encrypted = substr($combined, 16);
    
    // Decrypt
    return openssl_decrypt(
        $encrypted,
        'AES-256-CBC',
        hash('sha256', $key, true),
        OPENSSL_RAW_DATA,
        $iv
    );
}
```

**Example:**
```php
$decrypted = decrypt($encryptedData);
echo $decrypted;
```

## Two-Way Encryption for Fields

### encryptField()

Encrypts a database field value.

```php
/**
 * Encrypt field value
 * 
 * @param string $value Value to encrypt
 * @param string $fieldName Field name for key derivation
 * @return string Encrypted value
 */
function encryptField(string $value, string $fieldName): string
{
    $key = deriveFieldKey($fieldName);
    
    return encrypt($value, $key);
}

/**
 * Decrypt field value
 * 
 * @param string $encryptedValue Encrypted value
 * @param string $fieldName Field name
 * @return string Decrypted value
 */
function decryptField(string $encryptedValue, string $fieldName): string
{
    $key = deriveFieldKey($fieldName);
    
    return decrypt($encryptedValue, $key);
}

/**
 * Derive field-specific key
 * 
 * @param string $fieldName Field name
 * @return string Derived key
 */
function deriveFieldKey(string $fieldName): string
{
    $masterKey = Config\Setting::getValue('EncryptionKey');
    
    return hash('sha256', $masterKey . ':' . $fieldName);
}
```

**Example:**
```php
// Store encrypted credit card
$encryptedCard = encryptField('4111111111111111', 'credit_card');

// Retrieve and decrypt
$card = decryptField($encryptedCard, 'credit_card');
```

## Hash Functions

### hash()

Creates a hash using SHA-256.

```php
/**
 * Create hash
 * 
 * @param string $data Data to hash
 * @param string $salt Salt value
 * @return string Hash
 */
function hash(string $data, string $salt = ''): string
{
    return hash('sha256', $data . $salt);
}
```

### hashWithSalt()

Creates a salted hash.

```php
/**
 * Create salted hash
 * 
 * @param string $password Password
 * @param string|null $salt Salt (null = generate)
 * @return array ['hash' => string, 'salt' => string]
 */
function hashWithSalt(string $password, ?string $salt = null): array
{
    $salt = $salt ?: bin2hex(random_bytes(16));
    
    return [
        'hash' => hash('sha256', $password . $salt),
        'salt' => $salt
    ];
}
```

### verifyWithSalt()

Verifies password with salt.

```php
/**
 * Verify with salt
 * 
 * @param string $password Password
 * @param string $hash Hash
 * @param string $salt Salt
 * @return bool Valid
 */
function verifyWithSalt(string $password, string $hash, string $salt): bool
{
    return hash('sha256', $password . $salt) === $hash;
}
```

## API Key Functions

### generateApiKey()

Generates a new API key.

```php
/**
 * Generate API key
 * 
 * @param int $length Key length
 * @return string API key
 */
function generateApiKey(int $length = 32): string
{
    return bin2hex(random_bytes($length));
}
```

**Example:**
```php
$apiKey = generateApiKey(32);
// Returns: 64 character hex string

$apiKey = generateApiKey(16);
// Returns: 32 character hex string
```

### hashApiKey()

Hashes an API key for storage.

```php
/**
 * Hash API key
 * 
 * @param string $apiKey API key
 * @return string Hashed key
 */
function hashApiKey(string $apiKey): string
{
    return hash('sha256', $apiKey);
}
```

### verifyApiKey()

Verifies an API key against stored hash.

```php
/**
 * Verify API key
 * 
 * @param string $apiKey API key to verify
 * @param string $storedHash Stored hash
 * @return bool Valid
 */
function verifyApiKey(string $apiKey, string $storedHash): bool
{
    return hash('sha256', $apiKey) === $storedHash;
}
```

## Token Functions

### generateToken()

Generates a secure token.

```php
/**
 * Generate secure token
 * 
 * @param int $length Token length
 * @param string $charset Character set
 * @return string Token
 */
function generateToken(int $length = 32, string $charset = 'alphanumeric'): string
{
    $charsets = [
        'alphanumeric' => 'ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnpqrstuvwxyz23456789',
        'alpha' => 'ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnpqrstuvwxyz',
        'numeric' => '0123456789',
        'hex' => '0123456789abcdef'
    ];
    
    $chars = $charsets[$charset] ?? $charsets['alphanumeric'];
    $max = strlen($chars) - 1;
    
    $token = '';
    for ($i = 0; $i < $length; $i++) {
        $token .= $chars[random_int(0, $max)];
    }
    
    return $token;
}
```

**Example:**
```php
// Generate alphanumeric token
$token = generateToken(32);

// Generate numeric token (for 2FA)
$pin = generateToken(6, 'numeric');

// Generate hex token (for API)
$hexToken = generateToken(32, 'hex');
```

### generateSecureToken()

Generates cryptographically secure token.

```php
/**
 * Generate cryptographically secure token
 * 
 * @param int $bytes Number of bytes (output will be 2x hex)
 * @return string Token
 */
function generateSecureToken(int $bytes = 16): string
{
    return bin2hex(random_bytes($bytes));
}
```

## Random String Functions

### randomString()

Generates random string.

```php
/**
 * Generate random string
 * 
 * @param int $length Length
 * @param string $characters Characters to use
 * @return string Random string
 */
function randomString(int $length = 16, string $characters = 'ABCDEFGHJKLMNPQRSTUVWXYZabcdefghjkmnpqrstuvwxyz23456789'): string
{
    $max = strlen($characters) - 1;
    $result = '';
    
    for ($i = 0; $i < $length; $i++) {
        $result .= $characters[random_int(0, $max)];
    }
    
    return $result;
}
```

## Secure Comparison

### secureCompare()

Constant-time string comparison.

```php
/**
 * Secure string comparison
 * 
 * @param string $known Known string
 * @param string $user User-provided string
 * @return bool Equal
 */
function secureCompare(string $known, string $user): bool
{
    return hash_equals($known, $user);
}
```

**Example:**
```php
if (secureCompare($expectedToken, $providedToken)) {
    // Token is valid
}
```

## Data Masking

### maskString()

Masks a string for display.

```php
/**
 * Mask string
 * 
 * @param string $string String to mask
 * @param int $showStart Characters to show at start
 * @param int $showEnd Characters to show at end
 * @param string $maskChar Mask character
 * @return string Masked string
 */
function maskString(string $string, int $showStart = 4, int $showEnd = 4, string $maskChar = 'X'): string
{
    $length = strlen($string);
    
    if ($length <= $showStart + $showEnd) {
        return str_repeat($maskChar, $length);
    }
    
    $start = substr($string, 0, $showStart);
    $end = substr($string, -$showEnd);
    $middle = str_repeat($maskChar, $length - $showStart - $showEnd);
    
    return $start . $middle . $end;
}
```

**Example:**
```php
echo maskString('john@example.com', 2, 4);
// Output: joXXXXXXX.com

echo maskString('4111111111111111', 0, 4);
// Output: XXXXXXXXXXXXXXXX1111
```

### maskEmail()

Masks email address.

```php
/**
 * Mask email address
 * 
 * @param string $email Email address
 * @param string $maskChar Mask character
 * @return string Masked email
 */
function maskEmail(string $email, string $maskChar = 'X'): string
{
    $parts = explode('@', $email);
    
    if (count($parts) !== 2) {
        return maskString($email);
    }
    
    $local = $parts[0];
    $domain = $parts[1];
    
    $localLength = strlen($local);
    
    if ($localLength <= 2) {
        $maskedLocal = str_repeat($maskChar, $localLength);
    } else {
        $maskedLocal = $local[0] . str_repeat($maskChar, $localLength - 2) . $local[$localLength - 1];
    }
    
    return $maskedLocal . '@' . $domain;
}
```

**Example:**
```php
echo maskEmail('john.doe@example.com');
// Output: jXXXXXXoe@example.com
```

## Key Management

### rotateEncryptionKey()

Rotates encryption key.

```php
/**
 * Rotate encryption key
 * 
 * @param string $newKey New encryption key
 * @return array Results
 */
function rotateEncryptionKey(string $newKey): array
{
    $oldKey = Config\Setting::getValue('EncryptionKey');
    
    $reEncrypted = 0;
    $failed = 0;
    
    // Re-encrypt fields that use encryption
    $fields = ['credit_card', 'tax_id', 'bank_account'];
    
    foreach ($fields as $field) {
        $records = Capsule::table('tblclientdata')
            ->where('field', $field)
            ->whereNotNull('value_encrypted')
            ->get();
        
        foreach ($records as $record) {
            try {
                $decrypted = decrypt($record->value_encrypted, $oldKey);
                $reEncryptedData = encrypt($decrypted, $newKey);
                
                Capsule::table('tblclientdata')
                    ->where('id', $record->id)
                    ->update(['value_encrypted' => $reEncryptedData]);
                
                $reEncrypted++;
            } catch (Exception $e) {
                $failed++;
            }
        }
    }
    
    // Update encryption key
    Config\Setting::setValue('EncryptionKey', $newKey);
    
    return [
        're_encrypted' => $reEncrypted,
        'failed' => $failed
    ];
}
```

## Best Practices

1. **Use bcrypt for passwords** - Never use MD5 or SHA1 for passwords
2. **Store hashes, not plaintext** - Never store passwords unencrypted
3. **Use unique salts** - Each password should have unique salt
4. **Rotate keys periodically** - Regularly rotate encryption keys
5. **Use secure random** - Always use cryptographically secure random
6. **Time-safe comparisons** - Use hash_equals for comparisons

## Related Functions

- [whmcs-functions-sanitization.md](whmcs-functions-sanitization.md) - Data sanitization
- [whmcs-functions-utility.md](whmcs-functions-utility.md) - Utility functions