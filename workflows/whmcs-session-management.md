# WHMCS Session Management Workflow
# Version: 1.0 | Created: 2026-05-28

## Purpose

Comprehensive guide to managing WHMCS sessions including session configuration, storage, security, concurrent session handling, and troubleshooting session issues.

## Prerequisites

- WHMCS installation with admin access
- Understanding of PHP sessions
- Database access for session debugging
- SSL configuration for secure sessions

## Workflow Steps

### Step 1: Configure Session Storage

Set up optimal session storage configuration:

```php
// includes/hooks/session_config.php

/**
 * Configure database session storage
 */
add_hook('SessionStarted', 1, function() {
    // Set session save path to database
    ini_set('session.save_handler', 'database');

    // Session configuration
    ini_set('session.cookie_httponly', true);
    ini_set('session.cookie_secure', isset($_SERVER['HTTPS']));
    ini_set('session.cookie_samesite', 'Strict');
    ini_set('session.use_strict_mode', true);
    ini_set('session.gc_maxlifetime', 7200);
});

/**
 * Create session database table
 */
function createSessionTable(): void
{
    Capsule::schema()->create('whmcs_sessions', function($t) {
        $t->string('id', 128)->primary();
        $t->string('data', 65535)->nullable();
        $t->integer('last_activity');
        $t->string('ip_address', 45)->nullable();
        $t->string('user_agent', 255)->nullable();
        $t->string('user_id', 32)->nullable();
        $t->enum('session_type', ['admin', 'client']);
    });

    // Create index for garbage collection
    Capsule::statement(
        'CREATE INDEX idx_whmcs_sessions_activity ON whmcs_sessions (last_activity)'
    );
}

/**
 * Custom session handler using database
 */
class DatabaseSessionHandler implements SessionHandlerInterface
{
    private PDO $pdo;
    private string $table = 'whmcs_sessions';

    public function __construct(PDO $pdo)
    {
        $this->pdo = $pdo;
    }

    public function open(string $path, string $name): bool
    {
        return true;
    }

    public function close(): bool
    {
        return true;
    }

    public function read(string $id): string
    {
        $stmt = $this->pdo->prepare(
            "SELECT data FROM {$this->table} WHERE id = ? AND last_activity > ?"
        );
        $stmt->execute([$id, time() - $this->getLifetime()]);
        $result = $stmt->fetch(PDO::FETCH_ASSOC);

        return $result ? $result['data'] : '';
    }

    public function write(string $id, string $data): bool
    {
        $stmt = $this->pdo->prepare(
            "INSERT INTO {$this->table} (id, data, last_activity, ip_address, user_agent)
             VALUES (?, ?, ?, ?, ?)
             ON DUPLICATE KEY UPDATE data = ?, last_activity = ?"
        );

        return $stmt->execute([
            $id,
            $data,
            time(),
            $_SERVER['REMOTE_ADDR'] ?? '',
            $_SERVER['HTTP_USER_AGENT'] ?? '',
            $data,
            time(),
        ]);
    }

    public function destroy(string $id): bool
    {
        $stmt = $this->pdo->prepare("DELETE FROM {$this->table} WHERE id = ?");
        return $stmt->execute([$id]);
    }

    public function gc(int $maxlifetime): bool
    {
        $stmt = $this->pdo->prepare(
            "DELETE FROM {$this->table} WHERE last_activity < ?"
        );
        return $stmt->execute([time() - $maxlifetime]);
    }

    private function getLifetime(): int
    {
        return (int) ini_get('session.gc_maxlifetime');
    }
}
```

### Step 2: Implement Session Security

Add security measures to sessions:

```php
// includes/classes/SessionManager.php

class SessionManager
{
    private const FINGERPRINT_KEY = 'session_fingerprint';
    private const SESSION_TIMEOUT = 7200; // 2 hours
    private const ABSOLUTE_TIMEOUT = 28800; // 8 hours

    /**
     * Create session with security measures
     */
    public function createSecureSession(int $userId, string $type = 'client'): array
    {
        // Check for concurrent sessions if limited
        if ($this->isConcurrentSessionLimited()) {
            $this->invalidateOtherSessions($userId, session_id());
        }

        session_regenerate_id(true);

        $_SESSION['user_id'] = $userId;
        $_SESSION['session_type'] = $type;
        $_SESSION['created_at'] = time();
        $_SESSION['last_activity'] = time();
        $_SESSION[self::FINGERPRINT_KEY] = $this->generateFingerprint();

        // Store session in database for tracking
        $this->storeSessionRecord($userId, $type);

        logActivity("Secure session created for user {$userId}");

        return [
            'session_id' => session_id(),
            'created_at' => $_SESSION['created_at'],
            'timeout' => self::SESSION_TIMEOUT,
        ];
    }

    /**
     * Validate an existing session
     */
    public function validateSession(): bool
    {
        // Check if session has required keys
        if (!$this->hasRequiredSessionData()) {
            return false;
        }

        // Check session timeout
        $lastActivity = $_SESSION['last_activity'] ?? 0;
        if (time() - $lastActivity > self::SESSION_TIMEOUT) {
            $this->expireSession();
            return false;
        }

        // Check absolute timeout
        $createdAt = $_SESSION['created_at'] ?? 0;
        if (time() - $createdAt > self::ABSOLUTE_TIMEOUT) {
            $this->expireSession();
            return false;
        }

        // Validate fingerprint
        if (!$this->validateFingerprint()) {
            logActivity("Session fingerprint mismatch - possible hijacking");
            $this->destroySession();
            return false;
        }

        // Update last activity
        $_SESSION['last_activity'] = time();
        $this->updateSessionRecord();

        return true;
    }

    /**
     * Generate browser fingerprint
     */
    private function generateFingerprint(): string
    {
        $components = [
            $_SERVER['HTTP_USER_AGENT'] ?? '',
            $_SERVER['HTTP_ACCEPT_LANGUAGE'] ?? '',
            $_SERVER['HTTP_ACCEPT_ENCODING'] ?? '',
            $_SERVER['HTTP_ACCEPT'] ?? '',
        ];

        return hash('sha256', implode('|', $components));
    }

    /**
     * Validate session fingerprint
     */
    private function validateFingerprint(): bool
    {
        if (!isset($_SESSION[self::FINGERPRINT_KEY])) {
            return false;
        }

        $current = $this->generateFingerprint();
        return hash_equals($_SESSION[self::FINGERPRINT_KEY], $current);
    }

    /**
     * Store session in tracking database
     */
    private function storeSessionRecord(int $userId, string $type): void
    {
        Capsule::table('mod_session_tracking')->insert([
            'session_id'     => session_id(),
            'user_id'        => $userId,
            'session_type'   => $type,
            'ip_address'     => $_SERVER['REMOTE_ADDR'] ?? '',
            'user_agent'      => substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 255),
            'created_at'     => date('Y-m-d H:i:s'),
            'last_activity'  => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Update session activity timestamp
     */
    private function updateSessionRecord(): void
    {
        Capsule::table('mod_session_tracking')
            ->where('session_id', session_id())
            ->update(['last_activity' => date('Y-m-d H:i:s')]);
    }

    /**
     * Check if concurrent sessions should be limited
     */
    private function isConcurrentSessionLimited(): bool
    {
        return Capsule::table('tblconfiguration')
            ->where('setting', 'LimitConcurrentSessions')
            ->first()->value ?? false;
    }

    /**
     * Invalidate other sessions for the same user
     */
    private function invalidateOtherSessions(int $userId, string $currentSessionId): void
    {
        // Find and destroy other sessions
        $otherSessions = Capsule::table('mod_session_tracking')
            ->where('user_id', $userId)
            ->where('session_id', '!=', $currentSessionId)
            ->get();

        foreach ($otherSessions as $session) {
            $this->destroySessionById($session->session_id);
        }

        logActivity("Invalidated other sessions for user {$userId}");
    }

    /**
     * Destroy session by ID
     */
    private function destroySessionById(string $sessionId): void
    {
        Capsule::table('mod_session_tracking')
            ->where('session_id', $sessionId)
            ->delete();

        // Store invalidated session for history
        Capsule::table('mod_session_history')->insert([
            'session_id'   => $sessionId,
            'user_id'      => $_SESSION['user_id'] ?? 0,
            'destroyed_at' => date('Y-m-d H:i:s'),
            'reason'       => 'Concurrent session limit reached',
        ]);
    }
}
```

### Step 3: Handle Admin and Client Sessions

Different session handling for admin and client areas:

```php
// includes/hooks/admin_session.php

/**
 * Admin area session configuration
 */
add_hook('AdminAreaPageHook', 1, function() {
    // Set admin-specific session settings
    if (WebSystem::isAdminArea()) {
        $this->configureAdminSession();

        // Validate admin session
        $sessionManager = new SessionManager();
        if (!$sessionManager->validateSession()) {
            // Redirect to login
            header('Location: ' . $GLOBALS['CONFIG']['SystemURL'] . '/admin/login.php');
            exit;
        }

        // Check for forced re-authentication
        if ($this->requiresReauth()) {
            $this->showReauthPrompt();
        }
    }
});

private function configureAdminSession(): void
{
    // Admin sessions last longer
    ini_set('session.gc_maxlifetime', 14400); // 4 hours

    // Admin requires frequent session regeneration
    if ($this->shouldRegenerateSession()) {
        session_regenerate_id(true);
        $_SESSION['last_regeneration'] = time();
    }
}

private function shouldRegenerateSession(): bool
{
    $lastRegen = $_SESSION['last_regeneration'] ?? 0;
    return (time() - $lastRegen) > 900; // Every 15 minutes
}

/**
 * Client area session configuration
 */
add_hook('ClientAreaPageHook', 1, function() {
    $this->configureClientSession();
});

private function configureClientSession(): void
{
    // Standard session timeout
    ini_set('session.gc_maxlifetime', 7200); // 2 hours

    // Add CSRF token to client sessions
    if (!isset($_SESSION['csrf_token'])) {
        $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
    }
}
```

### Step 4: Debug Session Issues

Create comprehensive session debugging:

```php
// modules/addons/session_debug/session_debug.php

function session_debug_config(): array
{
    return [
        'name'        => 'Session Debug',
        'description' => 'Debug and troubleshoot session issues',
        'version'     => '1.0',
    ];
}

function session_debug_output(array $vars): void
{
    check_token('WHMCS.admin.default');

    $action = $_REQUEST['sub'] ?? 'status';

    switch ($action) {
        case 'status':
            $this->showSessionStatus();
            break;
        case 'logs':
            $this->showSessionLogs();
            break;
        case 'cleanup':
            $this->cleanupSessions();
            break;
        case 'test':
            $this->testSessionFunctionality();
            break;
    }
}

private function showSessionStatus(): void
{
    echo '<div class="session-debug">';
    echo '<h2>Session Debug Status</h2>';

    echo '<table class="datatable">';
    echo '<tr><th>Setting</th><th>Value</th></tr>';
    echo '<tr><td>Session ID</td><td>' . session_id() . '</td></tr>';
    echo '<tr><td>Session Status</td><td>' . (session_status() === PHP_SESSION_ACTIVE ? 'Active' : 'Inactive') . '</td></tr>';
    echo '<tr><td>Session Save Path</td><td>' . ini_get('session.save_path') . '</td></tr>';
    echo '<tr><td>Session Save Handler</td><td>' . ini_get('session.save_handler') . '</td></tr>';
    echo '<tr><td>Cookie Domain</td><td>' . ini_get('session.cookie_domain') . '</td></tr>';
    echo '<tr><td>Cookie Secure</td><td>' . (ini_get('session.cookie_secure') ? 'Yes' : 'No') . '</td></tr>';
    echo '<tr><td>Cookie HTTPOnly</td><td>' . (ini_get('session.cookie_httponly') ? 'Yes' : 'No') . '</td></tr>';
    echo '<tr><td>Session Lifetime</td><td>' . ini_get('session.gc_maxlifetime') . ' seconds</td></tr>';
    echo '</table>';

    echo '<h3>Current Session Data</h3>';
    echo '<pre>' . print_r($_SESSION, true) . '</pre西域';

    echo '<h3>Active Sessions in Database</h3>';
    echo $this->renderActiveSessionsTable();
    echo '</div>';
}

private function showSessionLogs(): void
{
    $logs = Capsule::table('mod_session_tracking')
        ->orderBy('last_activity', 'DESC')
        ->limit(100)
        ->get();

    echo '<table class="datatable">';
    echo '<thead><tr>';
    echo '<th>Session ID</th>';
    echo '<th>User ID</th>';
    echo '<th>Type</th>';
    echo '<th>IP Address</th>';
    echo '<th>Last Activity</th>';
    echo '<th>Actions</th>';
    echo '</tr></thead>';
    echo '<tbody>';

    foreach ($logs as $log) {
        echo '<tr>';
        echo '<td>' . substr($log->session_id, 0, 16) . '...</td>';
        echo '<td>' . $log->user_id . '</td>';
        echo '<td>' . $log->session_type . '</td>';
        echo '<td>' . $log->ip_address . '</td>';
        echo '<td>' . $log->last_activity . '</td>';
        echo '<td>';
        echo '<a href="?module=session_debug&action=destroy&session=' . $log->session_id . '">Destroy</a>';
        echo '</td>';
        echo '</tr>';
    }

    echo '</tbody></table>';
}

private function cleanupSessions(): void
{
    $before = Capsule::table('mod_session_tracking')->count();

    // Delete sessions older than 24 hours
    Capsule::table('mod_session_tracking')
        ->where('last_activity', '<', date('Y-m-d H:i:s', strtotime('-24 hours')))
        ->delete();

    $after = Capsule::table('mod_session_tracking')->count();

    logActivity("Session cleanup: removed " . ($before - $after) . " sessions");

    redir('module=session_debug&status=cleaned');
}

private function testSessionFunctionality(): array
{
    $results = [];

    // Test 1: Session creation
    $_SESSION['test_value'] = 'test_' . time();
    $results[] = ['Test' => 'Write Session', 'Result' => 'OK', 'Value' => $_SESSION['test_value']];

    // Test 2: Session read
    $readValue = $_SESSION['test_value'] ?? null;
    $results[] = ['Test' => 'Read Session', 'Result' => $readValue ? 'OK' : 'FAILED', 'Value' => $readValue];

    // Test 3: Session regeneration
    session_regenerate_id(true);
    $results[] = ['Test' => 'Regenerate ID', 'Result' => 'OK', 'Value' => session_id()];

    // Test 4: Session destroy
    session_destroy();
    $results[] = ['Test' => 'Destroy Session', 'Result' => 'OK', 'Value' => session_status() === PHP_SESSION_NONE ? 'Yes' : 'No'];

    return $results;
}
```

### Step 5: Session Recovery and Troubleshooting

Handle common session problems:

```php
// includes/classes/SessionRecovery.php

class SessionRecovery
{
    /**
     * Attempt to recover a broken session
     */
    public function attemptRecovery(): bool
    {
        // Clear all session cookies
        $this->clearSessionCookies();

        // Regenerate session ID
        session_regenerate_id(true);

        // Check for valid backup session
        if (isset($_COOKIE['backup_session'])) {
            return $this->restoreFromBackup($_COOKIE['backup_session']);
        }

        // Start fresh session
        return true;
    }

    private function clearSessionCookies(): void
    {
        $params = session_get_cookie_params();

        foreach (['REMEMBERME', 'adminid', 'cid'] as $cookie) {
            if (isset($_COOKIE[$cookie])) {
                setcookie($cookie, '', time() - 3600, $params['path'], $params['domain']);
            }
        }
    }

    private function restoreFromBackup(string $backupToken): bool
    {
        $backup = Capsule::table('mod_session_backup')
            ->where('token', $backupToken)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->first();

        if (!$backup) {
            return false;
        }

        $data = json_decode($backup->session_data, true);
        if (!$data) {
            return false;
        }

        foreach ($data as $key => $value) {
            $_SESSION[$key] = $value;
        }

        return true;
    }

    /**
     * Create session backup for recovery
     */
    public function createBackup(): void
    {
        $token = bin2hex(random_bytes(32));

        Capsule::table('mod_session_backup')->insert([
            'session_id'   => session_id(),
            'token'        => $token,
            'session_data' => json_encode($_SESSION),
            'created_at'   => date('Y-m-d H:i:s'),
            'expires_at'   => date('Y-m-d H:i:s', strtotime('+1 hour')),
        ]);

        setcookie('backup_session', $token, time() + 3600, '/', '', isset($_SERVER['HTTPS']), true);
    }

    /**
     * Diagnose session issues
     */
    public function diagnose(): array
    {
        $issues = [];

        // Check session save path
        $savePath = ini_get('session.save_path');
        if (empty($savePath)) {
            $issues[] = 'Session save path not configured';
        }

        // Check if session directory is writable
        if (!is_writable(sys_get_temp_dir())) {
            $issues[] = 'Session directory not writable';
        }

        // Check cookie settings
        if (!ini_get('session.cookie_httponly')) {
            $issues[] = 'Cookie HTTPOnly not enabled (security risk)';
        }

        // Check SSL cookie setting
        if (isset($_SERVER['HTTPS']) && !ini_get('session.cookie_secure')) {
            $issues[] = 'Secure cookie not enabled (security risk)';
        }

        // Check garbage collection
        $gcProbability = ini_get('session.gc_probability');
        $gcDivisor = ini_get('session.gc_divisor');
        if (($gcProbability == 0) || ($gcDivisor == 0)) {
            $issues[] = 'Garbage collection may not run';
        }

        // Check for session fixation vulnerabilities
        if (!ini_get('session.use_strict_mode')) {
            $issues[] = 'Strict session mode not enabled';
        }

        return $issues;
    }
}
```

---

## Best Practices

1. **Use secure cookie settings** - Enable httponly, secure, and samesite flags
2. **Regenerate session IDs** - After login and periodically during session
3. **Implement fingerprinting** - Detect session hijacking attempts
4. **Set reasonable timeouts** - Balance security with user experience
5. **Log session events** - Track session creation, destruction, and failures
6. **Handle concurrent sessions** - Decide policy for multiple sessions per user
7. **Provide recovery options** - Allow users to recover from session errors
8. **Monitor session storage** - Ensure database/file storage doesn't grow too large
9. **Test across browsers** - Verify session works in all supported browsers
10. **Document session flow** - Understand how WHMCS uses sessions

---

## Verification Checklist

- [ ] Session saves to database (if configured)
- [ ] Session cookies have secure flags set
- [ ] Session fingerprinting detects changes
- [ ] Session timeout works correctly
- [ ] Concurrent session handling functions
- [ ] Session debug tool displays correctly
- [ ] Session cleanup removes old sessions
- [ ] Recovery mechanism can restore sessions
- [ ] Admin and client sessions handled separately
- [ ] Session logs capture all events
