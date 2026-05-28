# WHMCS Session Handling Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for implementing secure session handling in WHMCS custom modules and maintaining session state.

## When to Use

- Building stateful module features
- Managing user sessions
- Implementing secure session tokens
- Session caching and optimization

## Session Handling Patterns

### 1. Session Manager Class

```php
<?php
namespace WHMCS\Session;

class SessionManager {
    private string $sessionPrefix = 'whmcs_';
    private int $defaultLifetime = 7200;

    public function __construct() {
        $this->initializeSession();
    }

    private function initializeSession(): void {
        if (session_status() === PHP_SESSION_NONE) {
            // Configure session before starting
            session_name('WHMPASS');
            session_set_cookie_params([
                'lifetime' => $this->defaultLifetime,
                'path' => '/',
                'secure' => isset($_SERVER['HTTPS']),
                'httponly' => true,
                'samesite' => 'Strict',
            ]);
            session_start();
        }

        // Initialize default session data
        if (!isset($_SESSION[$this->sessionPrefix . 'initialized'])) {
            $_SESSION[$this->sessionPrefix . 'initialized'] = true;
            $_SESSION[$this->sessionPrefix . 'created'] = time();
        }
    }

    public function set(string $key, mixed $value, ?int $lifetime = null): void {
        $sessionKey = $this->sessionPrefix . $key;
        $_SESSION[$sessionKey] = [
            'value' => $value,
            'expires' => $lifetime ? time() + $lifetime : 0,
            'created' => time(),
        ];
    }

    public function get(string $key, mixed $default = null): mixed {
        $sessionKey = $this->sessionPrefix . $key;

        if (!isset($_SESSION[$sessionKey])) {
            return $default;
        }

        $data = $_SESSION[$sessionKey];

        // Check expiration
        if ($data['expires'] > 0 && $data['expires'] < time()) {
            $this->remove($key);
            return $default;
        }

        return $data['value'];
    }

    public function has(string $key): bool {
        $sessionKey = $this->sessionPrefix . $key;

        if (!isset($_SESSION[$sessionKey])) {
            return false;
        }

        $data = $_SESSION[$sessionKey];

        // Check expiration
        if ($data['expires'] > 0 && $data['expires'] < time()) {
            $this->remove($key);
            return false;
        }

        return true;
    }

    public function remove(string $key): void {
        $sessionKey = $this->sessionPrefix . $key;
        unset($_SESSION[$sessionKey]);
    }

    public function flash(string $key, mixed $value): void {
        $this->set('flash_' . $key, $value, 300); // 5 minute expiry
    }

    public function getFlash(string $key, mixed $default = null): mixed {
        $value = $this->get('flash_' . $key, $default);
        $this->remove('flash_' . $key);
        return $value;
    }

    public function regenerate(): void {
        session_regenerate_id(true);
    }

    public function destroy(): void {
        $_SESSION = [];

        if (ini_get('session.use_cookies')) {
            $params = session_get_cookie_params();
            setcookie(session_name(), '', time() - 42000,
                $params['path'], $params['domain'],
                $params['secure'], $params['httponly']
            );
        }

        session_destroy();
    }

    public function getAll(): array {
        $data = [];
        $prefix = $this->sessionPrefix;

        foreach ($_SESSION as $key => $value) {
            if (str_starts_with($key, $prefix)) {
                $data[$key] = $value;
            }
        }

        return $data;
    }
}
```

### 2. Secure Token Session

```php
<?php
namespace WHMCS\Session;

class TokenSession {
    private string $tokenName = 'whmcs_token';
    private int $tokenLength = 32;

    public function generate(): string {
        $token = bin2hex(random_bytes($this->tokenLength));

        $_SESSION[$this->tokenName] = [
            'token' => $token,
            'created' => time(),
            'ip' => $this->getClientIp(),
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        ];

        return $token;
    }

    public function get(): ?string {
        $tokenData = $_SESSION[$this->tokenName] ?? null;

        if (!$tokenData) {
            return $this->generate();
        }

        // Validate token integrity
        if (!$this->validateTokenIntegrity($tokenData)) {
            $this->invalidate();
            return null;
        }

        return $tokenData['token'];
    }

    public function validate(string $token): bool {
        $tokenData = $_SESSION[$this->tokenName] ?? null;

        if (!$tokenData || $tokenData['token'] !== $token) {
            return false;
        }

        return $this->validateTokenIntegrity($tokenData);
    }

    private function validateTokenIntegrity(array $tokenData): bool {
        // Check IP address (optional, can be disabled)
        if (Capsule::table('tblconfiguration')
            ->where('setting', 'SessionValidateIP')
            ->first()->value ?? '1'
        ) {
            if ($tokenData['ip'] !== $this->getClientIp()) {
                return false;
            }
        }

        // Check timeout (30 minutes default)
        $timeout = (int) Capsule::table('tblconfiguration')
            ->where('setting', 'SessionTimeout')
            ->first()->value ?? 1800;

        if (time() - $tokenData['created'] > $timeout) {
            return false;
        }

        return true;
    }

    public function invalidate(): void {
        unset($_SESSION[$this->tokenName]);
    }

    public function refresh(): string {
        $this->invalidate();
        return $this->generate();
    }

    private function getClientIp(): string {
        $headers = ['HTTP_CF_CONNECTING_IP', 'HTTP_X_FORWARDED_FOR', 'HTTP_X_REAL_IP', 'REMOTE_ADDR'];

        foreach ($headers as $header) {
            if (!empty($_SERVER[$header])) {
                $ip = $_SERVER[$header];
                // Handle comma-separated IPs
                if (str_contains($ip, ',')) {
                    $ip = trim(explode(',', $ip)[0]);
                }
                return $ip;
            }
        }

        return 'unknown';
    }

    public function putCsrfToken(): string {
        $token = $this->get();
        $_SESSION['csrf_token'] = $token;
        return $token;
    }

    public function getCsrfToken(): ?string {
        return $_SESSION['csrf_token'] ?? null;
    }

    public function validateCsrfToken(?string $token): bool {
        if (!$token) {
            return false;
        }

        $storedToken = $this->getCsrfToken();

        if (!$storedToken || !hash_equals($storedToken, $token)) {
            return false;
        }

        // Use token once
        $this->invalidate();
        return true;
    }
}
```

### 3. Cached Session Store

```php
<?php
namespace WHMCS\Session;

class CachedSessionStore {
    private string $cachePrefix = 'session_';
    private int $ttl = 3600;
    private ?object $cache = null;

    public function __construct() {
        $this->initializeCache();
    }

    private function initializeCache(): void {
        // Try to use Redis/Memcached if available
        $driver = Capsule::table('tblconfiguration')
            ->where('setting', 'CacheDriver')
            ->first()->value ?? 'file';

        switch ($driver) {
            case 'redis':
                $this->cache = $this->connectRedis();
                break;
            case 'memcached':
                $this->cache = $this->connectMemcached();
                break;
            default:
                $this->cache = $this->connectFileCache();
        }
    }

    private function connectRedis(): object {
        // Redis implementation
        return new class {
            private array $store = [];
            private int $defaultTtl = 3600;

            public function get(string $key) {
                $data = $this->store[$key] ?? null;
                if (!$data) return null;

                if ($data['expires'] && $data['expires'] < time()) {
                    unset($this->store[$key]);
                    return null;
                }

                return $data['value'];
            }

            public function set(string $key, $value, int $ttl = 3600): void {
                $this->store[$key] = [
                    'value' => $value,
                    'expires' => time() + $ttl,
                ];
            }

            public function delete(string $key): void {
                unset($this->store[$key]);
            }

            public function flush(): void {
                $this->store = [];
            }
        };
    }

    private function connectMemcached(): object {
        // Memcached implementation (similar to Redis)
        return new class {
            private array $store = [];

            public function get(string $key) {
                return $this->store[$key] ?? null;
            }

            public function set(string $key, $value, int $ttl = 3600): void {
                $this->store[$key] = $value;
            }

            public function delete(string $key): void {
                unset($this->store[$key]);
            }
        };
    }

    private function connectFileCache(): object {
        // File-based cache fallback
        return new class {
            private string $cacheDir;
            private int $defaultTtl = 3600;

            public function __construct() {
                $this->cacheDir = dirname(__DIR__, 3) . '/cache/sessions/';
                if (!is_dir($this->cacheDir)) {
                    mkdir($this->cacheDir, 0755, true);
                }
            }

            public function get(string $key): mixed {
                $file = $this->cacheDir . md5($key) . '.cache';

                if (!file_exists($file)) {
                    return null;
                }

                $data = unserialize(file_get_contents($file));

                if ($data['expires'] && $data['expires'] < time()) {
                    unlink($file);
                    return null;
                }

                return $data['value'];
            }

            public function set(string $key, $value, int $ttl = 3600): void {
                $file = $this->cacheDir . md5($key) . '.cache';
                file_put_contents($file, serialize([
                    'value' => $value,
                    'expires' => time() + $ttl,
                ]));
            }

            public function delete(string $key): void {
                $file = $this->cacheDir . md5($key) . '.cache';
                if (file_exists($file)) {
                    unlink($file);
                }
            }

            public function flush(): void {
                array_map('unlink', glob($this->cacheDir . '*.cache'));
            }
        };
    }

    public function saveUserSession(int $userId, array $data): void {
        $sessionId = session_id();
        $cacheKey = $this->cachePrefix . 'user_' . $userId;

        $sessionData = array_merge($data, [
            'session_id' => $sessionId,
            'last_activity' => time(),
        ]);

        $this->cache->set($cacheKey, $sessionData, $this->ttl);
    }

    public function getUserSession(int $userId): ?array {
        $cacheKey = $this->cachePrefix . 'user_' . $userId;
        return $this->cache->get($cacheKey);
    }

    public function invalidateUserSession(int $userId): void {
        $cacheKey = $this->cachePrefix . 'user_' . $userId;
        $this->cache->delete($cacheKey);
    }

    public function invalidateAllSessions(): void {
        $this->cache->flush();
    }
}
```

### 4. Session Security

```php
<?php
namespace WHMCS\Session;

class SessionSecurity {
    private array $config;

    public function __construct() {
        $this->loadConfig();
    }

    private function loadConfig(): void {
        $this->config = [
            'validate_ip' => true,
            'validate_ua' => true,
            'cookie_httponly' => true,
            'cookie_secure' => isset($_SERVER['HTTPS']),
            'cookie_samesite' => 'Strict',
            'regenerate_on_login' => true,
            'idle_timeout' => 1800,
            'absolute_timeout' => 86400,
        ];

        foreach ($this->config as $key => &$value) {
            $setting = Capsule::table('tblconfiguration')
                ->where('setting', 'Session' . ucfirst($key))
                ->first();

            if ($setting) {
                $value = $setting->value;
            }
        }
    }

    public function validateRequest(): bool {
        // Check for session fixation
        if (!$this->checkSessionFixation()) {
            return false;
        }

        // Check for idle timeout
        if (!$this->checkIdleTimeout()) {
            return false;
        }

        return true;
    }

    private function checkSessionFixation(): bool {
        // Check that session IP matches current request IP
        if ($this->config['validate_ip'] && isset($_SESSION['clientip'])) {
            $currentIp = $this->getClientIp();
            $sessionIp = $_SESSION['clientip'];

            // Allow for same /24 subnet
            $currentSubnet = implode('.', array_slice(explode('.', $currentIp), 0, 3));
            $sessionSubnet = implode('.', array_slice(explode('.', $sessionIp), 0, 3));

            if ($currentSubnet !== $sessionSubnet) {
                logActivity('Session IP mismatch - possible fixation attempt');
                $this->destroySession();
                return false;
            }
        }

        return true;
    }

    private function checkIdleTimeout(): bool {
        if (!isset($_SESSION['last_activity'])) {
            $_SESSION['last_activity'] = time();
            return true;
        }

        $idleTime = time() - $_SESSION['last_activity'];

        if ($idleTime > $this->config['idle_timeout']) {
            logActivity('Session idle timeout exceeded');
            $this->destroySession();
            return false;
        }

        // Update last activity
        if ($idleTime > 60) {
            $_SESSION['last_activity'] = time();
        }

        // Check absolute timeout (24 hours)
        if (isset($_SESSION['created_at'])) {
            $sessionAge = time() - $_SESSION['created_at'];
            if ($sessionAge > $this->config['absolute_timeout']) {
                logActivity('Session absolute timeout exceeded');
                $this->destroySession();
                return false;
            }
        }

        return true;
    }

    public function secureSession(): void {
        // Regenerate session ID periodically
        if (!isset($_SESSION['last_regenerate'])) {
            $_SESSION['last_regenerate'] = time();
        }

        // Regenerate every 15 minutes
        if (time() - $_SESSION['last_regenerate'] > 900) {
            session_regenerate_id(true);
            $_SESSION['last_regenerate'] = time();
        }

        // Store security markers
        $_SESSION['clientip'] = $this->getClientIp();
        $_SESSION['user_agent'] = $_SERVER['HTTP_USER_AGENT'] ?? '';
        $_SESSION['created_at'] = $_SESSION['created_at'] ?? time();
    }

    public function loginSecurityCheck(): bool {
        // Check for too many failed attempts
        $recentFailures = Capsule::table('mod_auth_failures')
            ->where('ip_address', $this->getClientIp())
            ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-15 minutes')))
            ->count();

        if ($recentFailures >= 5) {
            logActivity("Rate limit exceeded for IP: {$this->getClientIp()}");
            return false;
        }

        // Check for suspicious session activity
        if ($this->detectSuspiciousActivity()) {
            logActivity('Suspicious session activity detected');
            return false;
        }

        return true;
    }

    private function detectSuspiciousActivity(): bool {
        // Check for rapid session creation
        $recentSessions = Capsule::table('mod_session_logs')
            ->where('ip_address', $this->getClientIp())
            ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-1 minute')))
            ->count();

        if ($recentSessions > 3) {
            return true;
        }

        return false;
    }

    private function destroySession(): void {
        $_SESSION = [];
        session_destroy();
    }

    private function getClientIp(): string {
        $headers = ['HTTP_CF_CONNECTING_IP', 'HTTP_X_FORWARDED_FOR', 'HTTP_X_REAL_IP', 'REMOTE_ADDR'];

        foreach ($headers as $header) {
            if (!empty($_SERVER[$header])) {
                return $_SERVER[$header];
            }
        }

        return 'unknown';
    }

    public function logSession(string $action, array $data = []): void {
        Capsule::table('mod_session_logs')->insert([
            'session_id' => session_id(),
            'user_id' => $_SESSION['uid'] ?? null,
            'action' => $action,
            'ip_address' => $this->getClientIp(),
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'data' => json_encode($data),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

### 5. Database Schema

```php
<?php
function createSessionTables(): void {
    Capsule::schema()->create('mod_session_logs', function($t) {
        $t->increments('id');
        $t->string('session_id', 128);
        $t->integer('user_id')->unsigned()->nullable();
        $t->string('action'); // login, logout, refresh, timeout
        $t->string('ip_address', 45);
        $t->string('user_agent')->nullable();
        $t->text('data')->nullable();
        $t->timestamp('created_at');

        $t->index(['user_id', 'created_at']);
        $t->index(['ip_address', 'created_at']);
    });

    Capsule::schema()->create('mod_session_activity', function($t) {
        $t->increments('id');
        $t->string('session_id', 128)->unique();
        $t->integer('user_id')->unsigned()->nullable();
        $t->string('ip_address', 45);
        $t->timestamp('last_activity');
        $t->timestamp('created_at');

        $t->index(['last_activity']);
    });
}
```

### 6. WHMCS Hook Integration

```php
<?php
// hooks.php
add_hook('SessionStart', 1, function($vars) {
    $security = new SessionSecurity();
    $security->secureSession();
    $security->validateRequest();
});

add_hook('ClientLogin', 1, function($vars) {
    $security = new SessionSecurity();
    $security->logSession('login', ['user_id' => $vars['user_id'] ?? $vars['id']]);
});

add_hook('ClientLogout', 1, function($vars) {
    $security = new SessionSecurity();
    $security->logSession('logout', ['user_id' => $vars['user_id'] ?? $vars['id']]);

    // Invalidate cached session
    $cache = new CachedSessionStore();
    $cache->invalidateUserSession($vars['user_id'] ?? $vars['id']);
});

add_hook('DailyCronJob', 1, function($vars) {
    // Clean up expired sessions
    $expired = date('Y-m-d H:i:s', strtotime('-24 hours'));

    Capsule::table('mod_session_activity')
        ->where('last_activity', '<', $expired)
        ->delete();

    // Clean up old session logs (keep 30 days)
    $oldLogs = date('Y-m-d H:i:s', strtotime('-30 days'));

    Capsule::table('mod_session_logs')
        ->where('created_at', '<', $oldLogs)
        ->delete();
});
```

## Checklist

- [ ] Session manager class
- [ ] Secure token generation
- [ ] CSRF token handling
- [ ] Session ID regeneration
- [ ] IP address validation
- [ ] Idle timeout tracking
- [ ] Cached session storage
- [ ] Session security logging
- [ ] Hook integration
- [ ] Session cleanup cron

---

**Related Skills:**
- whmcs-authentication-impl
- whmcs-security-hardening
- whmcs-security-checklist
- whmcs-two-factor-auth
