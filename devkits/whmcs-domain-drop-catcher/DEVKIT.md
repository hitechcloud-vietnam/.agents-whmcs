# WHMCS Domain Drop Catcher Devkit

## Overview
A high-performance domain drop catching system with real-time monitoring, automatic registration, and predictive analytics for capturing valuable expiring domains the moment they become available.

## Features
- Real-time WHOIS monitoring for expiring domains
- Millisecond-level capture response time
- Multiple registrar fallbacks for redundancy
- Predictive drop date calculation
- Bulk domain watching for drops
- Priority-based capture queuing
- Captured domain portfolio management
- Drop calendar and notification system
- Drop zone filtering (premium, brandable, niche)
- Historical drop analysis and statistics

## WHMCS Integration Points
- Custom module: `dropcatcher/DomainDropCatcher`
- Hook: `WhoisLookupComplete`
- Hook: `DomainCaptured`
- Server provisioning integration

## File Structure
```
whmcs-domain-drop-catcher/
├── DEVKIT.md
├── module.php
├── WHMCS/
│   └── Module/
│       └── DropCatcher/
│           └── DomainDropCatcher.php
├── includes/
│   ├── DropMonitor.php
│   ├── CaptureEngine.php
│   ├── RegistrarPool.php
│   ├── PredictionEngine.php
│   └── CaptureQueue.php
├── templates/
│   ├── client/
│   │   ├── watchlist.tpl
│   │   └── drops_calendar.tpl
│   └── admin/
│       ├── capture_dashboard.tpl
│       └── registrar_config.tpl
├── services/
│   ├── WhoisService.php
│   └── RdapService.php
├── assets/
│   ├── js/drop_monitor.js
│   └── css/dropcatcher.css
└── cron/
    ├── monitor_drops.php
    └── sync_drops.php
```

## Database Schema
```sql
CREATE TABLE mod_dropcatcher_domains (
    id INT AUTO_INCREMENT PRIMARY KEY,
    domain_name VARCHAR(255) NOT NULL UNIQUE,
    tld VARCHAR(50) NOT NULL,
    registrar VARCHAR(100),
    expiration_date DATE,
    drop_date DATE,
    drop_window_start DATETIME,
    drop_window_end DATETIME,
    predicted_drop_score DECIMAL(5,2),
    current_status ENUM('watched', 'pending_drop', 'capturing', 'captured', 'missed', 'contested') DEFAULT 'watched',
    capture_method VARCHAR(50),
    captured_by_user_id INT NULL,
    capture_timestamp TIMESTAMP NULL,
    capture_cost DECIMAL(10,2),
    auction_id INT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_drop_date (drop_date),
    INDEX idx_status (current_status)
);

CREATE TABLE mod_dropcatcher_watchlist (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    domain_name VARCHAR(255) NOT NULL,
    priority INT DEFAULT 0,
    max_bid DECIMAL(10,2),
    auto_capture TINYINT(1) DEFAULT 1,
    capture_registrar VARCHAR(100),
    notify_before_drop TINYINT(1) DEFAULT 1,
    notify_on_capture TINYINT(1) DEFAULT 1,
    status ENUM('active', 'captured', 'failed', 'cancelled', 'expired') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY unique_watch (user_id, domain_name),
    INDEX idx_priority (priority)
);

CREATE TABLE mod_dropcatcher_capture_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    domain_name VARCHAR(255) NOT NULL,
    capture_start TIMESTAMP,
    capture_end TIMESTAMP,
    capture_duration_ms INT,
    registrar_used VARCHAR(100),
    registrar_response TEXT,
    capture_result ENUM('success', 'failed', 'timeout', 'contested') NOT NULL,
    failure_reason TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_dropcatcher_registrars (
    id INT AUTO_INCREMENT PRIMARY KEY,
    registrar_name VARCHAR(100) NOT NULL,
    api_endpoint VARCHAR(255),
    api_key VARCHAR(255),
    api_secret VARCHAR(255),
    priority INT DEFAULT 0,
    is_active TINYINT(1) DEFAULT 1,
    avg_response_time_ms INT,
    success_rate DECIMAL(5,2),
    last_used_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_dropcatcher_analytics (
    id INT AUTO_INCREMENT PRIMARY KEY,
    domain_name VARCHAR(255) NOT NULL,
    days_to_drop INT,
    drop_hour INT,
    drop_minute INT,
    competition_level INT,
    capture_result ENUM('captured', 'missed', 'contested'),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Hooks
```php
// Hook: WhoisLookupComplete - Track drop candidates
add_hook('WhoisLookupComplete', 1, function($params) {
    $monitor = new DropMonitor();
    $monitor->processWhoisResult($params);
});

// Hook: DomainCaptured - Handle successful captures
add_hook('DomainCaptured', 1, function($params) {
    $catcher = new DomainDropCatcher();
    $catcher->handleSuccessfulCapture($params);
});

// Hook: DomainTransferCompleted - Update analytics
add_hook('DomainTransferCompleted', 1, function($params) {
    $analytics = new AnalyticsEngine();
    $analytics->recordCaptureResult($params);
});
```

## API Endpoints
```php
POST   /api/dropcatcher/watch          - Add domain to watchlist
DELETE /api/dropcatcher/watch/:domain  - Remove from watchlist
GET    /api/dropcatcher/drops          - Get upcoming drops
GET    /api/dropcatcher/calendar       - Get drop calendar
GET    /api/dropcatcher/capture/:domain - Manual capture attempt
GET    /api/dropcatcher/status/:domain - Get capture status
POST   /api/dropcatcher/bulk-watch     - Bulk add domains
GET    /api/dropcatcher/analytics      - Get capture analytics
```

## Configuration Fields
```php
$configfields = [
    'enable_drop_catching' => ['Type' => 'yesno'],
    'capture_mode' => ['Type' => 'dropdown', 'Options' => ['automatic', 'manual', 'hybrid']],
    'max_concurrent_captures' => ['Type' => 'text'],
    'capture_timeout_ms' => ['Type' => 'text'],
    'registrar_priority_1' => ['Type' => 'text'],
    'registrar_priority_2' => ['Type' => 'text'],
    'registrar_priority_3' => ['Type' => 'text'],
    'auto_retry' => ['Type' => 'yesno'],
    'retry_count' => ['Type' => 'text'],
    'prediction_enabled' => ['Type' => 'yesno'],
    'whois_check_interval' => ['Type' => 'dropdown']
];
```

## Implementation Example
```php
class CaptureEngine {
    private $registrarPool;
    private $queue;

    public function executeCapture($domain) {
        $startTime = microtime(true);

        // Get fastest available registrar
        $registrar = $this->registrarPool->getFastestRegistrar();

        try {
            $result = $registrar->register([
                'domain' => $domain,
                'period' => 1,
                'contacts' => $this->getDefaultContacts()
            ]);

            $duration = (microtime(true) - $startTime) * 1000;

            if ($result['success']) {
                $this->logCapture($domain, $registrar->getName(), $duration, 'success');
                return $this->handleSuccess($domain, $result);
            } else {
                throw new CaptureException($result['error']);
            }
        } catch (\Exception $e) {
            // Try fallback registrar
            return $this->tryFallbackRegistrar($domain, $e);
        }
    }

    public function processQueue() {
        $pending = CaptureQueue::getPending(10);

        foreach ($pending as $capture) {
            if ($this->isInDropWindow($capture['domain'])) {
                $this->executeCapture($capture['domain']);
            }
        }
    }
}

class PredictionEngine {
    public function predictDropDate($domain) {
        $historicalData = $this->getHistoricalDrops($domain);

        // Analyze registration period and typical drop patterns
        $predictedDate = $this->calculateDropDate($historicalData);

        // Calculate confidence score
        $confidence = $this->calculateConfidence($historicalData);

        return [
            'drop_date' => $predictedDate,
            'confidence' => $confidence,
            'drop_window' => $this->getDropWindow($predictedDate)
        ];
    }
}
```

## Testing Checklist
- [ ] WHOIS monitoring accuracy
- [ ] Capture speed (millisecond timing)
- [ ] Registrar failover
- [ ] Queue processing efficiency
- [ ] Drop date prediction accuracy
- [ ] Notification timing
- [ ] Analytics recording
- [ ] Admin dashboard updates
- [ ] Bulk watch operations
- [ ] Cost tracking accuracy

## Security Considerations
- Registrar API key encryption at rest
- Rate limiting on capture attempts
- Domain ownership verification
- Payment authorization before capture
- Audit logging for all operations

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- Multiple registrar API access
- RDAP/WHOIS protocol support
- Real-time processing capability
- Redis for queue management (optional)