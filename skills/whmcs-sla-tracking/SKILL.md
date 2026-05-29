# WHMCS SLA Tracking Skill

## Purpose
Provides implementation patterns for SLA (Service Level Agreement) monitoring, tracking, reporting, and enforcement in WHMCS hosting environments.

## Implementation Patterns

### SLA Manager
```php
<?php
// includes/SLATracking.class.php

class SLATracker {
    private $db;
    private $cache;
    
    public function __construct() {
        $this->db = console::db();
        $this->cache = console::cache();
    }
    
    // Create SLA definition
    public function createSLA($data) {
        $sla = [
            'name' => $data['name'],
            'description' => $data['description'],
            'response_time_minutes' => $data['response_time'],
            'resolution_time_minutes' => $data['resolution_time'],
            'availability_pct' => $data['availability'],
            'support_hours' => $data['support_hours'] ?? '24x7',
            'credit_policy' => json_encode($data['credit_policy']),
            'tier_level' => $data['tier_level'] ?? 'standard',
            'created_at' => date('Y-m-d H:i:s'),
            'is_active' => 1
        ];
        
        return $this->db->insert('mod_sla_definitions', $sla);
    }
    
    // Assign SLA to client
    public function assignToClient($clientId, $slaId, $effectiveDate = null) {
        $assignment = [
            'client_id' => $clientId,
            'sla_id' => $slaId,
            'assigned_at' => $effectiveDate ?? date('Y-m-d H:i:s'),
            'auto_renew' => $data['auto_renew'] ?? true,
            'billing_cycle' => $data['billing_cycle'] ?? 'monthly'
        ];
        
        $assignmentId = $this->db->insert('mod_sla_assignments', $assignment);
        
        // Initialize SLA period tracking
        $this->initializePeriod($assignmentId);
        
        return $assignmentId;
    }
    
    // Track response time
    public function trackResponseTime($ticketId, $startedAt) {
        $responseMinutes = (time() - strtotime($startedAt)) / 60;
        
        $ticket = $this->getTicket($ticketId);
        $sla = $this->getClientSLA($ticket['client_id']);
        
        $met = $responseMinutes <= $sla['response_time_minutes'];
        
        $this->logSLAMetric('response_time', $ticketId, $responseMinutes, $met);
        
        return [
            'minutes' => $responseMinutes,
            'sla_limit' => $sla['response_time_minutes'],
            'met' => $met
        ];
    }
    
    // Track resolution time
    public function trackResolutionTime($ticketId, $startedAt, $resolvedAt) {
        $resolutionMinutes = (strtotime($resolvedAt) - strtotime($startedAt)) / 60;
        
        $ticket = $this->getTicket($ticketId);
        $sla = $this->getClientSLA($ticket['client_id']);
        
        $met = $resolutionMinutes <= $sla['resolution_time_minutes'];
        
        $this->logSLAMetric('resolution_time', $ticketId, $resolutionMinutes, $met);
        
        return [
            'minutes' => $resolutionMinutes,
            'sla_limit' => $sla['resolution_time_minutes'],
            'met' => $met
        ];
    }
    
    // Calculate uptime percentage
    public function calculateUptime($clientId, $period = 'monthly') {
        $startDate = $this->getPeriodStart($period);
        $endDate = date('Y-m-d H:i:s');
        
        $downtimeEvents = $this->db->select(
            "SELECT SUM(duration_minutes) as total_downtime
             FROM mod_downtime_events
             WHERE client_id = ? AND start_time >= ?",
            [$clientId, $startDate]
        );
        
        $totalMinutes = (strtotime($endDate) - strtotime($startDate)) / 60;
        $downtime = $downtimeEvents['total_downtime'] ?? 0;
        
        $uptime = (($totalMinutes - $downtime) / $totalMinutes) * 100;
        
        $sla = $this->getClientSLA($clientId);
        $slaRequired = $sla['availability_pct'];
        
        return [
            'uptime_pct' => round($uptime, 4),
            'sla_required' => $slaRequired,
            'met' => $uptime >= $slaRequired,
            'downtime_minutes' => $downtime
        ];
    }
    
    // Generate SLA report
    public function generateReport($clientId, $period = 'monthly') {
        $sla = $this->getClientSLA($clientId);
        
        $report = [
            'period' => $period,
            'sla_tier' => $sla['name'],
            'response_time' => $this->getResponseTimeMetrics($clientId, $period),
            'resolution_time' => $this->getResolutionTimeMetrics($clientId, $period),
            'uptime' => $this->calculateUptime($clientId, $period),
            'credits_earned' => $this->calculateCredits($clientId, $period)
        ];
        
        return $report;
    }
}
```

### SLA Monitoring System
```php
class SLAMonitoringSystem {
    private $db;
    
    // Monitor active tickets for SLA compliance
    public function monitorActiveTickets() {
        $expiringTickets = $this->getExpiringTickets();
        
        foreach ($expiringTickets as $ticket) {
            $timeRemaining = $this->getTimeRemaining($ticket);
            
            if ($timeRemaining < 0) {
                $this->handleSLABreach($ticket, 'response');
            } elseif ($timeRemaining < 15) {
                $this->sendUrgentWarning($ticket, $timeRemaining);
            } elseif ($timeRemaining < 30) {
                $this->sendWarning($ticket, $timeRemaining);
            }
        }
    }
    
    private function getExpiringTickets() {
        return $this->db->select(
            "SELECT t.*, c.client_id, s.response_time_minutes, s.resolution_time_minutes
             FROM tbltickets t
             JOIN tblclients c ON t.userid = c.id
             JOIN mod_sla_assignments sa ON sa.client_id = c.id
             JOIN mod_sla_definitions s ON s.id = sa.sla_id
             WHERE t.status IN ('Open', 'Customer Reply')
             AND t.sla_breached = 0
             AND (
                 (t.last_reply = 'client' AND TIMESTAMPDIFF(MINUTE, t.last_reply_time, NOW()) >= s.response_time_minutes * 0.8)
                 OR (t.created_at AND TIMESTAMPDIFF(MINUTE, t.created_at, NOW()) >= s.resolution_time_minutes * 0.9)
             )"
        );
    }
    
    private function handleSLABreach($ticket, $type) {
        $this->db->where('id', $ticket['id'])
            ->update('tickets', [
                'sla_breached' => 1,
                'sla_breach_type' => $type,
                'sla_breach_time' => date('Y-m-d H:i:s')
            ]);
        
        // Trigger notifications
        $this->notifySLABreach($ticket, $type);
        
        // Calculate credit if applicable
        $credit = $this->calculateBreachCredit($ticket);
        $this->applyBreachCredit($ticket['userid'], $credit);
        
        logActivity("SLA breach detected for ticket #{$ticket['id']}", 'critical');
    }
    
    private function calculateBreachCredit($ticket) {
        $sla = $this->getClientSLA($ticket['userid']);
        $policy = json_decode($sla['credit_policy'], true);
        
        $breachType = $ticket['sla_breach_type'];
        $baseCredit = $policy['breaches'][$breachType] ?? 10;
        
        // Calculate based on severity and frequency
        $recentBreaches = $this->countRecentBreaches($ticket['userid'], '7 days');
        $multiplier = min($recentBreaches * 0.1 + 1, 3); // Max 3x multiplier
        
        return $baseCredit * $multiplier;
    }
    
    private function notifySLABreach($ticket, $type) {
        // Notify admin
        $this->sendAdminNotification($ticket, $type);
        
        // Notify client
        $this->sendClientNotification($ticket, $type);
        
        // Log for reporting
        $this->logBreach($ticket, $type);
    }
}
```

### Service Availability Tracker
```php
class ServiceAvailabilityTracker {
    // Record service status events
    public function recordEvent($serviceId, $type, $startTime, $endTime = null) {
        $event = [
            'service_id' => $serviceId,
            'event_type' => $type, // 'downtime', 'degradation', 'maintenance'
            'start_time' => $startTime,
            'end_time' => $endTime,
            'duration_minutes' => $endTime 
                ? (strtotime($endTime) - strtotime($startTime)) / 60 
                : null,
            'logged_at' => date('Y-m-d H:i:s')
        ];
        
        $this->db->insert('mod_service_events', $event);
        
        // Update affected clients
        $service = $this->getService($serviceId);
        $this->updateAffectedClients($service['client_id']);
        
        return $this->db->lastInsertID();
    }
    
    // Mark event as resolved
    public function resolveEvent($eventId, $endTime = null) {
        $event = $this->getEvent($eventId);
        $duration = $endTime 
            ? (strtotime($endTime) - strtotime($event['start_time'])) / 60
            : (time() - strtotime($event['start_time'])) / 60;
        
        $this->db->where('id', $eventId)
            ->update('mod_service_events', [
                'end_time' => $endTime ?? date('Y-m-d H:i:s'),
                'duration_minutes' => $duration,
                'resolved' => 1
            ]);
        
        // Notify clients
        $this->notifyResolution($event);
    }
    
    // Get availability for a service
    public function getServiceAvailability($serviceId, $period = 'monthly') {
        $startDate = $this->getPeriodStart($period);
        
        $events = $this->db->select(
            "SELECT * FROM mod_service_events 
             WHERE service_id = ? AND start_time >= ?",
            [$serviceId, $startDate]
        );
        
        $totalMinutes = $this->getPeriodMinutes($period);
        $downtimeMinutes = array_sum(array_column($events, 'duration_minutes'));
        
        $uptime = (($totalMinutes - $downtimeMinutes) / $totalMinutes) * 100;
        
        return [
            'uptime_pct' => round($uptime, 4),
            'total_downtime_minutes' => $downtimeMinutes,
            'incident_count' => count($events),
            'average_resolution_time' => count($events) > 0 
                ? $downtimeMinutes / count($events) 
                : 0
        ];
    }
    
    // Calculate client-wide availability
    public function getClientAvailability($clientId, $period = 'monthly') {
        $services = $this->getClientServices($clientId);
        
        $totalWeight = 0;
        $weightedUptime = 0;
        
        foreach ($services as $service) {
            $availability = $this->getServiceAvailability($service['id'], $period);
            $weight = $service['weight'] ?? 1;
            
            $totalWeight += $weight;
            $weightedUptime += $availability['uptime_pct'] * $weight;
        }
        
        return $totalWeight > 0 ? $weightedUptime / $totalWeight : 100;
    }
}
```

### SLA Credit Calculator
```php
class SLACreditCalculator {
    public function calculateCredits($clientId, $period = 'monthly') {
        $credits = [];
        
        // Calculate response time credits
        $responseBreaches = $this->getBreaches($clientId, 'response_time', $period);
        $responseCredit = $this->calculateResponseCredits($responseBreaches);
        $credits['response_time'] = $responseCredit;
        
        // Calculate resolution time credits
        $resolutionBreaches = $this->getBreaches($clientId, 'resolution_time', $period);
        $resolutionCredit = $this->calculateResolutionCredits($resolutionBreaches);
        $credits['resolution_time'] = $resolutionCredit;
        
        // Calculate availability credits
        $availability = $this->getAvailabilityMetrics($clientId, $period);
        $availabilityCredit = $this->calculateAvailabilityCredit($availability);
        $credits['availability'] = $availabilityCredit;
        
        $credits['total'] = array_sum($credits);
        
        return $credits;
    }
    
    private function calculateResponseCredits($breaches) {
        $credit = 0;
        
        foreach ($breaches as $breach) {
            $baseCredit = 5; // $5 per response breach
            
            // Factor in severity
            $overage = $breach['actual_minutes'] - $breach['sla_minutes'];
            $severityMultiplier = min(1 + ($overage / $breach['sla_minutes']), 3);
            
            $credit += $baseCredit * $severityMultiplier;
        }
        
        return round($credit, 2);
    }
    
    private function calculateAvailabilityCredit($availability) {
        if ($availability['uptime_pct'] >= $availability['sla_required']) {
            return 0;
        }
        
        // Sliding scale credit
        $shortfall = $availability['sla_required'] - $availability['uptime_pct'];
        
        if ($shortfall <= 0.1) {
            return 5; // 5% credit
        } elseif ($shortfall <= 0.5) {
            return 10; // 10% credit
        } elseif ($shortfall <= 1) {
            return 25; // 25% credit
        } else {
            return 50; // 50% credit
        }
    }
    
    public function applyCredits($clientId, $credits, $description = null) {
        $invoiceId = $this->findOpenInvoice($clientId);
        
        if ($invoiceId) {
            $this->db->insert('tblinvoiceitems', [
                'invoice_id' => $invoiceId,
                'type' => 'Credit',
                'description' => $description ?? 'SLA Credit',
                'amount' => -$credits['total'],
                'created_at' => date('Y-m-d H:i:s')
            ]);
        }
        
        // Or create credit note
        $creditNote = [
            'client_id' => $clientId,
            'amount' => $credits['total'],
            'reason' => 'SLA Breach Credit',
            'applied_to' => $invoiceId ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        return $this->db->insert('mod_sla_credits', $creditNote);
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_sla_definitions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    response_time_minutes INT NOT NULL,
    resolution_time_minutes INT NOT NULL,
    availability_pct DECIMAL(5,2) NOT NULL,
    support_hours VARCHAR(50) DEFAULT '24x7',
    credit_policy JSON,
    tier_level VARCHAR(50),
    created_at DATETIME,
    is_active TINYINT(1) DEFAULT 1
);

CREATE TABLE mod_sla_assignments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    sla_id INT NOT NULL,
    assigned_at DATETIME NOT NULL,
    auto_renew TINYINT(1) DEFAULT 1,
    billing_cycle VARCHAR(20) DEFAULT 'monthly',
    INDEX idx_client (client_id)
);

CREATE TABLE mod_sla_metrics (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    ticket_id INT,
    metric_type ENUM('response_time', 'resolution_time', 'availability') NOT NULL,
    actual_minutes DECIMAL(10,2),
    sla_minutes DECIMAL(10,2),
    met TINYINT(1),
    breach_severity VARCHAR(20),
    recorded_at DATETIME,
    INDEX idx_client_type (client_id, metric_type),
    INDEX idx_period (recorded_at)
);

CREATE TABLE mod_sla_breaches (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    ticket_id INT,
    breach_type VARCHAR(50),
    severity VARCHAR(20),
    minutes_overdue DECIMAL(10,2),
    credit_amount DECIMAL(10,2),
    credited TINYINT(1) DEFAULT 0,
    credited_at DATETIME,
    created_at DATETIME,
    INDEX idx_client (client_id)
);

CREATE TABLE mod_service_events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    event_type ENUM('downtime', 'degradation', 'maintenance', 'other') NOT NULL,
    start_time DATETIME NOT NULL,
    end_time DATETIME,
    duration_minutes INT,
    resolved TINYINT(1) DEFAULT 0,
    logged_at DATETIME,
    INDEX idx_service_time (service_id, start_time)
);

CREATE TABLE mod_downtime_events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    service_id INT,
    start_time DATETIME NOT NULL,
    end_time DATETIME,
    duration_minutes INT,
    cause VARCHAR(255),
    notified TINYINT(1) DEFAULT 0,
    INDEX idx_client (client_id)
);

CREATE TABLE mod_sla_credits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    invoice_id INT,
    amount DECIMAL(10,2) NOT NULL,
    reason VARCHAR(255),
    period_start DATE,
    period_end DATE,
    created_at DATETIME,
    applied_at DATETIME,
    INDEX idx_client (client_id)
);
```

## Usage Examples

### Track Ticket SLA
```php
$tracker = new SLATracker();

// Start tracking when ticket created
$tracker->trackResponseTime($ticketId, $ticket['created_at']);

// When resolved, calculate resolution time
$resolution = $tracker->trackResolutionTime(
    $ticketId, 
    $ticket['created_at'], 
    date('Y-m-d H:i:s')
);

if (!$resolution['met']) {
    echo "SLA breach - resolution exceeded by " . 
        ($resolution['minutes'] - $resolution['sla_limit']) . " minutes";
}
```

### Generate Client SLA Report
```php
$tracker = new SLATracker();
$report = $tracker->generateReport($clientId, 'monthly');

echo "SLA Report for " . date('F Y') . "\n";
echo "Response Time Compliance: " . $report['response_time']['compliance_pct'] . "%\n";
echo "Resolution Time Compliance: " . $report['resolution_time']['compliance_pct'] . "%\n";
echo "Uptime: " . $report['uptime']['uptime_pct'] . "%\n";
echo "Credits Earned: $" . $report['credits_earned']['total'] . "\n";
```

### Monitor SLA Compliance
```php
$monitor = new SLAMonitoringSystem();
$monitor->monitorActiveTickets(); // Run via cron every 5 minutes
```

## Integration Points

- WHMCS support tickets for response/resolution tracking
- Server monitoring modules for availability data
- Invoice system for credit application
- Admin notifications for breach alerts
- Client portal for SLA status visibility