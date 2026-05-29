# WHMCS Domain Backorder System Devkit

## Overview
A domain backordering system that allows clients to place backorders on expiring domains, with automated monitoring, notification services, and competitive backorder management for premium domain acquisition.

## Features
- Domain backorder placement and management
- Real-time expiration monitoring
- Priority queue system based on order time and payment status
- Automated domain capture when available
- Multi-user backorder on same domain
- Backorder success notifications
- Failed capture handling with retry
- Backorder pricing tiers (standard, premium, auction)
- Waitlist management for popular domains
- Transfer assistance after successful capture

## WHMCS Integration Points
- Custom product: `domain_backorder`
- Hook: `DomainExpired`
- Hook: `DomainDeleted`
- Hook: `DomainTransferredAway`
- Cart addon integration

## File Structure
```
whmcs-domain-backorder/
├── DEVKIT.md
├── module.php
├── WHMCS/
│   └── Module/
│       └── Backorder/
│           └── DomainBackorderManager.php
├── includes/
│   ├── CaptureEngine.php
│   ├── MonitoringService.php
│   ├── PriorityQueue.php
│   └── NotificationService.php
├── templates/
│   ├── client/
│   │   ├── backorder_form.tpl
│   │   └── my_backorders.tpl
│   └── admin/
│       └── backorder_management.tpl
├── assets/
│   └── css/backorder.css
├── cron/
│   └── monitor_expired.php
└── api/
    └── backorder.php
```

## Database Schema
```sql
CREATE TABLE mod_backorder_requests (
    id INT AUTO_INCREMENT PRIMARY KEY,
    domain_name VARCHAR(255) NOT NULL,
    user_id INT NOT NULL,
    order_id INT NULL,
    priority INT DEFAULT 0,
    status ENUM('pending', 'captured', 'failed', 'cancelled', 'transferred') DEFAULT 'pending',
    capture_attempts INT DEFAULT 0,
    max_attempts INT DEFAULT 5,
    base_price DECIMAL(10,2) NOT NULL,
    success_fee DECIMAL(10,2) DEFAULT 0.00,
    captured_at TIMESTAMP NULL,
    failed_reason TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_domain (domain_name),
    INDEX idx_user (user_id),
    INDEX idx_status (status)
);

CREATE TABLE mod_backorder_queue (
    id INT AUTO_INCREMENT PRIMARY KEY,
    domain_name VARCHAR(255) NOT NULL,
    priority_rank INT NOT NULL,
    capture_time DATETIME,
    capture_result ENUM('success', 'failed', 'contested') NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_backorder_pricing_tiers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    tier_name VARCHAR(50) NOT NULL,
    base_price DECIMAL(10,2) NOT NULL,
    success_fee_percent DECIMAL(5,2),
    priority_boost INT DEFAULT 0,
    features JSON,
    is_active TINYINT(1) DEFAULT 1
);

CREATE TABLE mod_backorder_notifications (
    id INT AUTO_INCREMENT PRIMARY KEY,
    backorder_id INT NOT NULL,
    notification_type ENUM('status_update', 'capture_success', 'capture_failed', 'transfer_ready') NOT NULL,
    sent_via ENUM('email', 'sms', 'both') DEFAULT 'email',
    sent_at TIMESTAMP,
    content TEXT
);

CREATE TABLE mod_backorder_domains (
    id INT AUTO_INCREMENT PRIMARY KEY,
    domain_name VARCHAR(255) NOT NULL UNIQUE,
    current_status ENUM('expired', 'redemption', 'pending_delete', 'available', 'backordered') DEFAULT 'expired',
    expires_at DATETIME,
    dropped_at DATETIME,
    captured_by_user_id INT NULL,
    capture_cost DECIMAL(10,2),
    whois_updated_at TIMESTAMP,
    INDEX idx_status (current_status),
    INDEX idx_expires (expires_at)
);
```

## Hooks
```php
// Hook: DomainExpired - Add to monitoring queue
add_hook('DomainExpired', 1, function($params) {
    $monitor = new MonitoringService();
    $monitor->addToMonitoring($params['domain']);
});

// Hook: DomainDeleted - Trigger capture attempt
add_hook('DomainDeleted', 1, function($params) {
    $capture = new CaptureEngine();
    $capture->attemptCapture($params['domain']);
});

// Hook: AfterOrderPaid - Upgrade backorder priority
add_hook('AfterOrderPaid', 1, function($params) {
    $queue = new PriorityQueue();
    $queue->upgradePriority($params['order_id']);
});
```

## API Endpoints
```php
POST   /api/backorder/place          - Place new backorder
GET    /api/backorder/status/:domain - Get backorder status
GET    /api/backorder/my             - Get user's backorders
DELETE /api/backorder/cancel/:id      - Cancel backorder
GET    /api/backorder/check/:domain   - Check if domain available
POST   /api/backorder/priority/upgrade - Upgrade backorder priority
```

## Configuration Fields
```php
$configfields = [
    'enable_backorder' => ['Type' => 'yesno'],
    'default_pricing_tier' => ['Type' => 'dropdown'],
    'capture_retry_interval' => ['Type' => 'dropdown', 'Options' => ['1', '5', '10', '30', '60']],
    'max_capture_attempts' => ['Type' => 'text'],
    'monitoring_interval' => ['Type' => 'dropdown'],
    'registrar_api_credentials' => ['Type' => 'password'],
    'auto_capture_enabled' => ['Type' => 'yesno'],
    'success_notification_email' => ['Type' => 'yesno'],
    'payment_hold_days' => ['Type' => 'text']
];
```

## Implementation Example
```php
class CaptureEngine {
    private $registrarAdapter;

    public function attemptCapture($domainName) {
        $queue = new PriorityQueue();
        $backorders = $queue->getNextInQueue($domainName);

        foreach ($backorders as $backorder) {
            try {
                // Check domain availability
                if (!$this->checkAvailability($domainName)) {
                    $this->markUnavailable($domainName);
                    continue;
                }

                // Attempt registration
                $result = $this->registerDomain($domainName, $backorder['user_id']);

                if ($result['success']) {
                    $this->handleSuccessfulCapture($backorder, $result);
                    return true;
                } else {
                    $this->handleFailedCapture($backorder, $result['error']);
                }
            } catch (\Exception $e) {
                Log::error("Capture failed for {$domainName}: " . $e->getMessage());
                $this->retryCapture($backorder);
            }
        }

        return false;
    }

    protected function registerDomain($domain, $userId) {
        $user = User::find($userId);

        return $this->registrarAdapter->register([
            'domain' => $domain,
            'registrant' => $user->registrantData,
            'period' => 1,
            'contact' => $user->contactData
        ]);
    }
}
```

## Testing Checklist
- [ ] Backorder placement
- [ ] Priority queue ordering
- [ ] Capture attempt execution
- [ ] Failed capture retry
- [ ] Multi-backorder competition
- [ ] Notification delivery
- [ ] Transfer after capture
- [ ] Pricing tier application
- [ ] Admin management interface
- [ ] Cron monitoring accuracy

## Security Considerations
- Domain ownership verification before capture
- Payment verification before priority boost
- Rate limiting on capture attempts
- Registrar API credential encryption
- User identity verification for transfers

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- Registrar API access (OpenSRS, Enom, etc.)
- Cron job scheduling for monitoring
- Notification service (email/SMS)