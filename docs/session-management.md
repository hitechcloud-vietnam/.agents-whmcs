# Session Management Patterns

Proper session management is critical for security and user experience. This guide covers patterns for managing sessions in WHMCS modules.

## Session Configuration

### Custom Session Handler

```php
<?php
/**
 * Custom session handler for WHMCS modules
 */
class ModuleSessionHandler implements SessionHandlerInterface
{
    private $connection;
    private $tableName = 'mod_session_data';
    private $maxLifetime = 3600; // 1 hour

    public function __construct()
    {
        $this->connection = Capsule::connection();
    }

    /**
     * Open session
     */
    public function open($savePath, $sessionName): bool
    {
        $this->ensureSessionTable();
        return true;
    }

    /**
     * Close session
     */
    public function close(): bool
    {
        return true;
    }

    /**
     * Read session data
     */
    public function read($sessionId): string
    {
        $stmt = $this->connection->selectOne(
            "SELECT data FROM {$this->tableName}
             WHERE id = ? AND expires > ?",
            [$sessionId, time()]
        );

        return $stmt ? (string) $stmt->data : '';
    }

    /**
     * Write session data
     */
    public function write($sessionId, $data): bool
    {
        $expires = time() + $this->maxLifetime;

        $this->connection->statement(
            "INSERT INTO {$this->tableName} (id, data, expires)
             VALUES (?, ?, ?)
             ON DUPLICATE KEY UPDATE data = VALUES(data), expires = VALUES(expires)",
            [$sessionId, $data, $expires]
        );

        return true;
    }

    /**
     * Destroy session
     */
    public function destroy($sessionId): bool
    {
        $this->connection->statement(
            "DELETE FROM {$this->tableName} WHERE id = ?",
            [$sessionId]
        );

        return true;
    }

    /**
     * Garbage collection
     */
    public function gc($maxLifetime): bool
    {
        $this->connection->statement(
            "DELETE FROM {$this->tableName} WHERE expires < ?",
            [time()]
        );

        return true;
    }

    private function ensureSessionTable(): void
    {
        if (!Capsule::schema()->hasTable($this->tableName)) {
            Capsule::schema()->create($this->tableName, function ($table) {
                $table->string('id', 128)->primary();
                $table->longText('data');
                $table->integer('expires')->index();
            });
        }
    }
}
```

## Session Security

### Secure Session Manager

```php
<?php
/**
 * Secure session management
 */
class SecureSessionManager
{
    private const SESSION_PREFIX = 'whmcs_module_';
    private const CSRF_TOKEN_KEY = 'csrf_token';
    private const ADMIN_ID_KEY = 'admin_id';

    /**
     * Initialize secure session
     */
    public static function init(): void
    {
        if (session_status() === PHP_SESSION_NONE) {
            // Set secure session parameters
            ini_set('session.cookie_httponly', 1);
            ini_set('session.use_only_cookies', 1);
            ini_set('session.cookie_lifetime', 0);
            ini_set('session.use_strict_mode', 1);
            ini_set('session.cookie_samesite', 'Strict');

            // Use secure cookie if HTTPS
            if ($this->isHttps()) {
                ini_set('session.cookie_secure', 1);
            }

            session_start();
        }

        // Regenerate session ID periodically
        self::regenerateIdPeriodically();
    }

    /**
     * Set session value with prefix
     */
    public static function set(string $key, $value): void
    {
        $_SESSION[self::SESSION_PREFIX . $key] = $value;
    }

    /**
     * Get session value
     */
    public static function get(string $key, $default = null)
    {
        return $_SESSION[self::SESSION_PREFIX . $key] ?? $default;
    }

    /**
     * Check if session has key
     */
    public static function has(string $key): bool
    {
        return isset($_SESSION[self::SESSION_PREFIX . $key]);
    }

    /**
     * Remove session value
     */
    public static function remove(string $key): void
    {
        unset($_SESSION[self::SESSION_PREFIX . $key]);
    }

    /**
     * Store admin authentication
     */
    public static function setAdminAuth(int $adminId, array $permissions): void
    {
        self::set(self::ADMIN_ID_KEY, $adminId);
        self::set('admin_permissions', $permissions);
        self::set('admin_auth_time', time());
    }

    /**
     * Get authenticated admin ID
     */
    public static function getAdminId(): ?int
    {
        return self::get(self::ADMIN_ID_KEY);
    }

    /**
     * Check if admin is authenticated
     */
    public static function isAdminAuthenticated(): bool
    {
        return self::has(self::ADMIN_ID_KEY);
    }

    /**
     * Generate and store CSRF token
     */
    public static function generateCsrfToken(): string
    {
        $token = bin2hex(random_bytes(32));
        self::set(self::CSRF_TOKEN_KEY, [
            'token' => $token,
            'created' => time(),
        ]);
        return $token;
    }

    /**
     * Validate CSRF token
     */
    public static function validateCsrfToken(string $token): bool
    {
        $stored = self::get(self::CSRF_TOKEN_KEY);

        if (!$stored || !isset($stored['token'])) {
            return false;
        }

        // Check token age (1 hour)
        if (time() - $stored['created'] > 3600) {
            self::remove(self::CSRF_TOKEN_KEY);
            return false;
        }

        return hash_equals($stored['token'], $token);
    }

    /**
     * Regenerate session ID
     */
    public static function regenerate(): void
    {
        if (session_status() === PHP_SESSION_ACTIVE) {
            session_regenerate_id(true);
        }
    }

    /**
     * Destroy session
     */
    public static function destroy(): void
    {
        $_SESSION = [];

        if (ini_get('session.use_cookies')) {
            $params = session_get_cookie_params();
            setcookie(
                session_name(),
                '',
                time() - 42000,
                $params['path'],
                $params['domain'],
                $params['secure'],
                $params['httponly']
            );
        }

        session_destroy();
    }

    private static function regenerateIdPeriodically(): void
    {
        $lastRegenerate = self::get('last_session_regenerate', 0);
        $regenerateInterval = 900; // 15 minutes

        if (time() - $lastRegenerate > $regenerateInterval) {
            self::regenerate();
            self::set('last_session_regenerate', time());
        }
    }

    private function isHttps(): bool
    {
        return (
            (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off') ||
            (!empty($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') ||
            (!empty($_SERVER['SERVER_PORT']) && $_SERVER['SERVER_PORT'] === 443)
        );
    }
}
```

## Flash Messages

### Flash Message Manager

```php
<?php
/**
 * Flash message manager for one-time notifications
 */
class FlashMessageManager
{
    private const FLASH_KEY = 'flash_messages';

    /**
     * Set flash message
     */
    public static function set(string $type, string $message, array $data = []): void
    {
        $flashes = self::getAll();
        $flashes[$type][] = [
            'message' => $message,
            'data' => $data,
            'created' => time(),
        ];
        $_SESSION[self::FLASH_KEY] = $flashes;
    }

    /**
     * Get flash messages by type
     */
    public static function get(string $type): array
    {
        $flashes = self::getAll();
        $messages = $flashes[$type] ?? [];
        unset($flashes[$type]);
        $_SESSION[self::FLASH_KEY] = $flashes;
        return $messages;
    }

    /**
     * Check if flash messages exist
     */
    public static function has(string $type): bool
    {
        $flashes = self::getAll();
        return !empty($flashes[$type]);
    }

    /**
     * Get all flash messages
     */
    public static function getAll(): array
    {
        return $_SESSION[self::FLASH_KEY] ?? [];
    }

    /**
     * Clear all flash messages
     */
    public static function clear(): void
    {
        unset($_SESSION[self::FLASH_KEY]);
    }

    /**
     * Convenience methods
     */
    public static function success(string $message): void
    {
        self::set('success', $message);
    }

    public static function error(string $message): void
    {
        self::set('error', $message);
    }

    public static function warning(string $message): void
    {
        self::set('warning', $message);
    }

    public static function info(string $message): void
    {
        self::set('info', $message);
    }
}
```

## Session Storage Adapters

### Redis Session Adapter

```php
<?php
/**
 * Redis-based session storage
 */
class RedisSessionAdapter implements SessionHandlerInterface
{
    private $redis;
    private $prefix = 'whmcs_session:';
    private $ttl = 3600;

    public function __construct(array $config)
    {
        $this->redis = new Redis();
        $this->redis->connect($config['host'], $config['port']);

        if (!empty($config['password'])) {
            $this->redis->auth($config['password']);
        }

        if (isset($config['ttl'])) {
            $this->ttl = $config['ttl'];
        }

        if (isset($config['prefix'])) {
            $this->prefix = $config['prefix'];
        }
    }

    public function open($savePath, $sessionName): bool
    {
        return true;
    }

    public function close(): bool
    {
        return true;
    }

    public function read($sessionId): string
    {
        $data = $this->redis->get($this->prefix . $sessionId);
        return $data !== false ? $data : '';
    }

    public function write($sessionId, $data): bool
    {
        $this->redis->setex($this->prefix . $sessionId, $this->ttl, $data);
        return true;
    }

    public function destroy($sessionId): bool
    {
        $this->redis->del($this->prefix . $sessionId);
        return true;
    }

    public function gc($maxLifetime): bool
    {
        // Redis handles expiry automatically
        return true;
    }
}
```

### Memcached Session Adapter

```php
<?php
/**
 * Memcached-based session storage
 */
class MemcachedSessionAdapter implements SessionHandlerInterface
{
    private $memcached;
    private $prefix = 'whmcs_session:';
    private $ttl = 3600;

    public function __construct(array $config)
    {
        $this->memcached = new Memcached();
        $this->memcached->addServer(
            $config['host'] ?? '127.0.0.1',
            $config['port'] ?? 11211
        );

        if (isset($config['ttl'])) {
            $this->ttl = $config['ttl'];
        }
    }

    public function open($savePath, $sessionName): bool
    {
        return true;
    }

    public function close(): bool
    {
        return true;
    }

    public function read($sessionId): string
    {
        $data = $this->memcached->get($this->prefix . $sessionId);
        return $data !== false ? $data : '';
    }

    public function write($sessionId, $data): bool
    {
        return $this->memcached->set($this->prefix . $sessionId, $data, $this->ttl);
    }

    public function destroy($sessionId): bool
    {
        $this->memcached->delete($this->prefix . $sessionId);
        return true;
    }

    public function gc($maxLifetime): bool
    {
        // Memcached handles expiry automatically
        return true;
    }
}
```

## Session Locking

### Atomic Session Updates

```php
<?php
/**
 * Atomic session update manager
 */
class AtomicSessionManager
{
    private $lockTimeout = 10;
    private $maxRetries = 3;

    /**
     * Update session atomically with locking
     */
    public function atomicUpdate(string $key, callable $updateFn, $default = null): mixed
    {
        $retryCount = 0;
        $lockKey = 'lock:' . $key;

        while ($retryCount < $this->maxRetries) {
            // Try to acquire lock
            $lockId = uniqid('', true);
            if ($this->acquireLock($lockKey, $lockId)) {
                try {
                    $currentValue = $_SESSION[$key] ?? $default;
                    $newValue = $updateFn($currentValue);
                    $_SESSION[$key] = $newValue;
                    return $newValue;
                } finally {
                    $this->releaseLock($lockKey, $lockId);
                }
            }

            $retryCount++;
            usleep(rand(10000, 50000)); // Wait 10-50ms
        }

        throw new SessionLockException("Failed to acquire lock for key: $key");
    }

    /**
     * Acquire distributed lock
     */
    private function acquireLock(string $key, string $lockId): bool
    {
        // Using file-based locking for simplicity
        $lockFile = sys_get_temp_dir() . '/' . md5($key) . '.lock';

        $fp = fopen($lockFile, 'c+');
        if (!$fp) {
            return false;
        }

        if (flock($fp, LOCK_EX | LOCK_NB)) {
            // Store lock info
            file_put_contents($lockFile . '.info', json_encode([
                'lock_id' => $lockId,
                'acquired_at' => time(),
            ]));
            return true;
        }

        fclose($fp);
        return false;
    }

    /**
     * Release lock
     */
    private function releaseLock(string $key, string $lockId): void
    {
        $lockFile = sys_get_temp_dir() . '/' . md5($key) . '.lock';

        $fp = fopen($lockFile, 'c+');
        if ($fp) {
            flock($fp, LOCK_UN);
            fclose($fp);
        }

        @unlink($lockFile);
        @unlink($lockFile . '.info');
    }
}

class SessionLockException extends Exception {}
```

## Session Monitoring

### Session Activity Tracker

```php
<?php
/**
 * Session activity monitoring
 */
class SessionActivityTracker
{
    /**
     * Track session activity
     */
    public static function track(int $userId, string $activity, array $context = []): void
    {
        Capsule::table('mod_session_activity')->insert([
            'user_id' => $userId,
            'session_id' => session_id(),
            'activity' => $activity,
            'ip_address' => self::getClientIp(),
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'context' => json_encode($context),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Get active sessions for user
     */
    public static function getActiveSessions(int $userId): array
    {
        $timeout = 30; // 30 minutes
        $cutoff = date('Y-m-d H:i:s', time() - ($timeout * 60));

        return Capsule::table('mod_session_activity')
            ->where('user_id', $userId)
            ->where('created_at', '>', $cutoff)
            ->groupBy('session_id')
            ->get();
    }

    /**
     * Count active users
     */
    public static function countActiveUsers(): int
    {
        $timeout = 30;
        $cutoff = date('Y-m-d H:i:s', time() - ($timeout * 60));

        return Capsule::table('mod_session_activity')
            ->where('created_at', '>', $cutoff)
            ->distinct('user_id')
            ->count();
    }

    private static function getClientIp(): string
    {
        return $_SERVER['HTTP_X_FORWARDED_FOR'] ?? $_SERVER['REMOTE_ADDR'] ?? 'unknown';
    }
}

// Hook for automatic activity tracking
add_hook('ClientAreaPage', 1, function ($vars) {
    if (isset($_SESSION['uid'])) {
        SessionActivityTracker::track($_SESSION['uid'], 'page_view', [
            'uri' => $_SERVER['REQUEST_URI'],
        ]);
    }
});
```

## Best Practices

1. **Use secure session settings** - Enable httponly, secure, and samesite cookies
2. **Regenerate session IDs** - After authentication and periodically
3. **Use session locking** - Prevent race conditions
4. **Implement session timeout** - Auto-expire inactive sessions
5. **Store sessions securely** - Use encrypted storage or trusted backends
6. **Track session activity** - Monitor for suspicious behavior
7. **Clear sensitive data** - Remove data on logout
8. **Use flash messages** - For one-time notifications

## Related Patterns

- [Admin Security](./admin-security.md) - Admin session management
- [Cache Strategies](./cache-strategies.md) - Session caching
- [Session Management in Hooks](./hooks-reference.md) - Hook-based session handling