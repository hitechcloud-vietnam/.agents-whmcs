# WHMCS Log Analysis Workflow

## Purpose

Comprehensive guide to analyzing WHMCS logs for troubleshooting, security monitoring, performance optimization, and audit compliance.

## Prerequisites

- WHMCS installation access
- Log file access (SSH or file manager)
- Log aggregation tools (optional)
- Understanding of WHMCS log structures

## Workflow Steps

### Step 1: Log Types and Locations

Understand WHMCS log files:

```php
// Log configuration

return [
    'log_locations' => [
        'activity_log' => [
            'path' => '/storage/logs/activitylog.txt',
            'description' => 'User activity and administrative actions',
            'rotation' => 'daily',
        ],
        'module_log' => [
            'path' => '/storage/logs/module-log.log',
            'description' => 'Provisioning module operations',
            'rotation' => 'weekly',
        ],
        'api_log' => [
            'path' => '/storage/logs/api.log',
            'description' => 'API requests and responses',
            'rotation' => 'daily',
        ],
        'error_log' => [
            'path' => '/storage/logs/error.log',
            'description' => 'PHP errors and warnings',
            'rotation' => 'weekly',
        ],
        'email_log' => [
            'path' => '/storage/logs/emails.log',
            'description' => 'Email sending history',
            'rotation' => 'daily',
        ],
        'ticket_log' => [
            'path' => '/storage/logs/ticketlog.txt',
            'description' => 'Support ticket changes',
            'rotation' => 'weekly',
        ],
        'payment_gateway_log' => [
            'path' => '/storage/logs/gateway-{gateway}.log',
            'description' => 'Payment gateway transactions',
            'rotation' => 'monthly',
        ],
        'cron_log' => [
            'path' => '/storage/logs/crons.log',
            'description' => 'Cron job execution results',
            'rotation' => 'daily',
        ],
    ],
    
    'database_tables' => [
        'tblactivitylog' => 'Activity log database table',
        'tbllog' => 'General logging table',
        'mod_gateway_log' => 'Gateway-specific logs',
        'tblticketlog' => 'Ticket changes log',
    ],
];
```

### Step 2: Log Parsing and Analysis

Create log analysis tools:

```php
// includes/LogAnalyzer.php

class LogAnalyzer
{
    private $logPath;
    
    public function __construct(string $logPath = null)
    {
        $this->logPath = $logPath ?? ROOTDIR . '/storage/logs/activitylog.txt';
    }
    
    /**
     * Parse activity log entries
     */
    public function parseActivityLog(int $limit = 100): array
    {
        if (!file_exists($this->logPath)) {
            return [];
        }
        
        $entries = [];
        $lines = file($this->logPath, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
        
        $lines = array_slice($lines, -$limit);
        
        foreach ($lines as $line) {
            $entry = $this->parseActivityLine($line);
            if ($entry) {
                $entries[] = $entry;
            }
        }
        
        return array_reverse($entries);
    }
    
    /**
     * Parse single activity log line
     */
    private function parseActivityLine(string $line): ?array
    {
        // Format: [timestamp] status: message | userid: X | ip: X.X.X.X
        if (preg_match('/^\[(.+?)\]\s+(.+?):\s+(.+?)(?:\s*\|\s*(.+))?$/i', $line, $matches)) {
            $entry = [
                'timestamp' => $matches[1],
                'status' => $matches[2],
                'message' => $matches[3],
            ];
            
            // Parse additional fields
            if (!empty($matches[4])) {
                $fields = explode('|', $matches[4]);
                foreach ($fields as $field) {
                    if (preg_match('/(\w+):\s*(.+)/', trim($field), $fieldMatch)) {
                        $entry[$fieldMatch[1]] = trim($fieldMatch[2]);
                    }
                }
            }
            
            return $entry;
        }
        
        return null;
    }
    
    /**
     * Search logs by criteria
     */
    public function search(array $criteria): array
    {
        $results = [];
        
        $lines = file($this->logPath, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
        
        foreach ($lines as $line) {
            $entry = $this->parseActivityLine($line);
            
            if (!$entry) continue;
            
            $match = true;
            
            foreach ($criteria as $field => $value) {
                if (!isset($entry[$field])) {
                    $match = false;
                    break;
                }
                
                if (is_array($value)) {
                    // Pattern matching
                    if (!preg_match($value['pattern'], $entry[$field])) {
                        $match = false;
                        break;
                    }
                } elseif ($entry[$field] !== $value) {
                    $match = false;
                    break;
                }
            }
            
            if ($match) {
                $results[] = $entry;
            }
        }
        
        return $results;
    }
    
    /**
     * Get error statistics
     */
    public function getErrorStatistics(DateTime $start, DateTime $end): array
    {
        $stats = [
            'total_entries' => 0,
            'errors' => 0,
            'warnings' => 0,
            'by_type' => [],
            'by_user' => [],
            'by_hour' => [],
        ];
        
        $lines = file($this->logPath, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
        
        foreach ($lines as $line) {
            $entry = $this->parseActivityLine($line);
            
            if (!$entry) continue;
            
            $entryTime = DateTime::createFromFormat('Y-m-d H:i:s', $entry['timestamp']);
            
            if (!$entryTime || $entryTime < $start || $entryTime > $end) {
                continue;
            }
            
            $stats['total_entries']++;
            
            // Categorize
            $status = strtolower($entry['status'] ?? '');
            
            if (strpos($status, 'error') !== false) {
                $stats['errors']++;
                $type = $this->categorizeError($entry['message']);
                $stats['by_type'][$type] = ($stats['by_type'][$type] ?? 0) + 1;
            } elseif (strpos($status, 'warning') !== false) {
                $stats['warnings']++;
            }
            
            // Track by user
            if (!empty($entry['userid'])) {
                $stats['by_user'][$entry['userid']] = 
                    ($stats['by_user'][$entry['userid']] ?? 0) + 1;
            }
            
            // Track by hour
            $hour = $entryTime->format('Y-m-d H:00');
            $stats['by_hour'][$hour] = ($stats['by_hour'][$hour] ?? 0) + 1;
        }
        
        return $stats;
    }
    
    private function categorizeError(string $message): string
    {
        $message = strtolower($message);
        
        if (strpos($message, 'database') !== false || strpos($message, 'sql') !== false) {
            return 'database';
        }
        
        if (strpos($message, 'payment') !== false || strpos($message, 'transaction') !== false) {
            return 'payment';
        }
        
        if (strpos($message, 'email') !== false || strpos($message, 'mail') !== false) {
            return 'email';
        }
        
        if (strpos($message, 'module') !== false) {
            return 'module';
        }
        
        if (strpos($message, 'api') !== false) {
            return 'api';
        }
        
        if (strpos($message, 'login') !== false || strpos($message, 'auth') !== false) {
            return 'authentication';
        }
        
        return 'other';
    }
}
```

### Step 3: Security Log Analysis

Analyze logs for security events:

```php
// includes/SecurityLogAnalyzer.php

class SecurityLogAnalyzer
{
    /**
     * Detect brute force attacks
     */
    public function detectBruteForce(int $threshold = 10, int $windowMinutes = 15): array
    {
        $cutoff = date('Y-m-d H:i:s', strtotime("-{$windowMinutes} minutes"));
        
        // Get failed logins from activity log
        $failedLogins = Capsule::table('tblactivitylog')
            ->where('description', 'LIKE', '%Failed Login Attempt%')
            ->where('datetime', '>=', $cutoff)
            ->selectRaw('description, ipaddr, COUNT(*) as attempts')
            ->groupBy('description', 'ipaddr')
            ->havingRaw('COUNT(*) >= ?', [$threshold])
            ->get();
        
        $attacks = [];
        
        foreach ($failedLogins as $attempt) {
            // Extract IP from description
            preg_match('/(\d+\.\d+\.\d+\.\d+)/', $attempt->description, $matches);
            $ip = $matches[1] ?? $attempt->ipaddr;
            
            // Get all attempts from this IP
            $allAttempts = Capsule::table('tblactivitylog')
                ->where('description', 'LIKE', '%Failed Login Attempt%')
                ->where('datetime', '>=', $cutoff)
                ->where('ipaddr', $ip)
                ->get();
            
            $attacks[] = [
                'ip_address' => $ip,
                'attempt_count' => $attempt->attempts,
                'first_attempt' => $allAttempts->first()->datetime ?? null,
                'last_attempt' => $allAttempts->last()->datetime ?? null,
                'target_users' => $this->getTargetedUsers($allAttempts),
            ];
        }
        
        return $attacks;
    }
    
    /**
     * Detect suspicious IP addresses
     */
    public function detectSuspiciousIPs(): array
    {
        $suspicious = [];
        
        // IPs with high error rates
        $highErrorIPs = Capsule::table('tblactivitylog')
            ->whereRaw("description LIKE '%Error%' OR description LIKE '%Failed%'")
            ->where('datetime', '>=', date('Y-m-d H:i:s', strtotime('-24 hours')))
            ->selectRaw('ipaddr, COUNT(*) as error_count')
            ->groupBy('ipaddr')
            ->havingRaw('COUNT(*) > 50')
            ->get();
        
        foreach ($highErrorIPs as $ip) {
            $suspicious[] = [
                'ip' => $ip->ipaddr,
                'reason' => 'high_error_rate',
                'error_count' => $ip->error_count,
            ];
        }
        
        // IPs from known proxy ranges
        $proxyRanges = ['10.0.0.0/8', '172.16.0.0/12', '192.168.0.0/16'];
        
        $recentLogins = Capsule::table('tblactivitylog')
            ->where('description', 'LIKE', '%Successful Login%')
            ->where('datetime', '>=', date('Y-m-d H:i:s', strtotime('-1 hour')))
            ->get();
        
        foreach ($recentLogins as $login) {
            if ($this->isPrivateIP($login->ipaddr)) {
                $suspicious[] = [
                    'ip' => $login->ipaddr,
                    'reason' => 'private_ip_login',
                    'time' => $login->datetime,
                ];
            }
        }
        
        return $suspicious;
    }
    
    /**
     * Detect privilege escalation attempts
     */
    public function detectPrivilegeEscalation(): array
    {
        $escalations = [];
        
        // Look for admin access from non-admin accounts
        $suspiciousAccess = Capsule::table('tblactivitylog')
            ->where('description', 'LIKE', '%Admin Area Access%')
            ->where('datetime', '>=', date('Y-m-d H:i:s', strtotime('-24 hours')))
            ->get();
        
        foreach ($suspiciousAccess as $access) {
            // Verify user is actually an admin
            $userId = $this->extractUserId($access->description);
            
            if ($userId) {
                $isAdmin = Capsule::table('tbladmins')
                    ->where('id', $userId)
                    ->exists();
                
                if (!$isAdmin) {
                    $escalations[] = [
                        'ip' => $access->ipaddr,
                        'user_id' => $userId,
                        'time' => $access->datetime,
                        'description' => $access->description,
                    ];
                }
            }
        }
        
        return $escalations;
    }
    
    /**
     * Detect data exfiltration attempts
     */
    public function detectDataExfiltration(): array
    {
        $indicators = [];
        
        // Large API requests
        $largeAPIRequests = Capsule::table('mod_api_logs')
            ->where('bytes_sent', '>', 10 * 1024 * 1024) // > 10MB
            ->where('created_at', '>=', date('Y-m-d H:i:s', strtotime('-24 hours')))
            ->get();
        
        foreach ($largeAPIRequests as $request) {
            $indicators[] = [
                'type' => 'large_data_export',
                'ip' => $request->ip_address,
                'user_id' => $request->user_id,
                'bytes' => $request->bytes_sent,
                'endpoint' => $request->endpoint,
            ];
        }
        
        return $indicators;
    }
    
    private function getTargetedUsers($attempts): array
    {
        $users = [];
        
        foreach ($attempts as $attempt) {
            preg_match('/for user (.+?)@/i', $attempt->description, $matches);
            if (!empty($matches[1])) {
                $users[] = $matches[1];
            }
        }
        
        return array_unique($users);
    }
    
    private function isPrivateIP(string $ip): bool
    {
        return !filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE);
    }
    
    private function extractUserId(string $description): ?int
    {
        preg_match('/User ID:\s*(\d+)/', $description, $matches);
        return !empty($matches[1]) ? (int)$matches[1] : null;
    }
}
```

### Step 4: Performance Log Analysis

Analyze logs for performance issues:

```php
// includes/PerformanceLogAnalyzer.php

class PerformanceLogAnalyzer
{
    /**
     * Analyze slow page loads
     */
    public function analyzeSlowPages(int $thresholdMs = 2000): array
    {
        $slowPages = [];
        
        // Read from performance monitoring logs
        $logs = Capsule::table('perf_metrics')
            ->where('metric_name', 'page_load_time')
            ->where('value', '>', $thresholdMs)
            ->where('recorded_at', '>=', date('Y-m-d H:i:s', strtotime('-24 hours')))
            ->orderBy('value', 'desc')
            ->limit(50)
            ->get();
        
        foreach ($logs as $log) {
            $data = json_decode($log->additional_data ?? '{}', true);
            
            $slowPages[] = [
                'page' => $data['page'] ?? 'unknown',
                'load_time_ms' => $log->value,
                'timestamp' => $log->recorded_at,
                'user_id' => $log->user_id,
            ];
        }
        
        return $slowPages;
    }
    
    /**
     * Analyze database query performance
     */
    public function analyzeSlowQueries(int $thresholdMs = 100): array
    {
        $slowQueries = Capsule::table('perf_slow_queries')
            ->where('execution_time_ms', '>', $thresholdMs)
            ->where('created_at', '>=', date('Y-m-d H:i:s', strtotime('-24 hours')))
            ->orderBy('execution_time_ms', 'desc')
            ->limit(100)
            ->get();
        
        // Group by query pattern
        $grouped = [];
        
        foreach ($slowQueries as $query) {
            $pattern = $this->normalizeQuery($query->query);
            
            if (!isset($grouped[$pattern])) {
                $grouped[$pattern] = [
                    'query_pattern' => $pattern,
                    'count' => 0,
                    'total_time_ms' => 0,
                    'max_time_ms' => 0,
                    'examples' => [],
                ];
            }
            
            $grouped[$pattern]['count']++;
            $grouped[$pattern]['total_time_ms'] += $query->execution_time_ms;
            $grouped[$pattern]['max_time_ms'] = max(
                $grouped[$pattern]['max_time_ms'],
                $query->execution_time_ms
            );
            
            if (count($grouped[$pattern]['examples']) < 3) {
                $grouped[$pattern]['examples'][] = [
                    'time_ms' => $query->execution_time_ms,
                    'explain' => json_decode($query->explain_data ?? '{}', true),
                ];
            }
        }
        
        // Calculate average and sort by impact
        foreach ($grouped as &$group) {
            $group['avg_time_ms'] = round($group['total_time_ms'] / $group['count'], 2);
            $group['impact'] = $group['count'] * $group['avg_time_ms'];
        }
        
        usort($grouped, fn($a, $b) => $b['impact'] <=> $a['impact']);
        
        return array_slice($grouped, 0, 20);
    }
    
    /**
     * Analyze cron job performance
     */
    public function analyzeCronPerformance(): array
    {
        $cronStats = [];
        
        $cronLogs = Capsule::table('tblactivitylog')
            ->where('description', 'LIKE', '%Cron Job%')
            ->where('datetime', '>=', date('Y-m-d H:i:s', strtotime('-7 days')))
            ->get();
        
        foreach ($cronLogs as $log) {
            preg_match('/Cron Job: (.+?)(?:\s|$)/', $log->description, $matches);
            $cronName = $matches[1] ?? 'unknown';
            
            if (!isset($cronStats[$cronName])) {
                $cronStats[$cronName] = [
                    'name' => $cronName,
                    'executions' => 0,
                    'failures' => 0,
                    'total_duration_ms' => 0,
                ];
            }
            
            $cronStats[$cronName]['executions']++;
            
            if (strpos($log->description, 'Failed') !== false || 
                strpos($log->description, 'Error') !== false) {
                $cronStats[$cronName]['failures']++;
            }
            
            // Extract duration if present
            preg_match('/Duration:\s*(\d+(?:\.\d+)?)\s*ms/i', $log->description, $durationMatch);
            if (!empty($durationMatch[1])) {
                $cronStats[$cronName]['total_duration_ms'] += (float)$durationMatch[1];
            }
        }
        
        foreach ($cronStats as &$stat) {
            $stat['avg_duration_ms'] = $stat['executions'] > 0 
                ? round($stat['total_duration_ms'] / $stat['executions'], 2)
                : 0;
            $stat['failure_rate'] = $stat['executions'] > 0
                ? round(($stat['failures'] / $stat['executions']) * 100, 2)
                : 0;
        }
        
        return array_values($cronStats);
    }
    
    private function normalizeQuery(string $query): string
    {
        // Replace literal values with placeholders
        $normalized = preg_replace('/\d+/', '?', $query);
        $normalized = preg_replace('/\'[^\']*\'/', '?', $normalized);
        
        // Remove extra whitespace
        $normalized = preg_replace('/\s+/', ' ', $normalized);
        
        return trim($normalized);
    }
}
```

### Step 5: Log Aggregation Setup

Set up centralized log management:

```php
// modules/addons/log_aggregator/aggregator.php

/**
 * Send logs to centralized logging service
 */
class LogAggregator
{
    private $endpoint;
    private $apiKey;
    
    public function __construct(string $endpoint, string $apiKey)
    {
        $this->endpoint = $endpoint;
        $this->apiKey = $apiKey;
    }
    
    /**
     * Send log entry to aggregator
     */
    public function send(array $logEntry): bool
    {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->endpoint . '/api/logs',
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($logEntry),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 5,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return $httpCode >= 200 && $httpCode < 300;
    }
    
    /**
     * Batch send logs
     */
    public function sendBatch(array $logEntries, int $batchSize = 100): array
    {
        $results = ['sent' => 0, 'failed' => 0];
        
        $batches = array_chunk($logEntries, $batchSize);
        
        foreach ($batches as $batch) {
            if ($this->send($batch)) {
                $results['sent'] += count($batch);
            } else {
                $results['failed'] += count($batch);
            }
        }
        
        return $results;
    }
}

/**
 * Hook to capture and aggregate logs
 */
add_hook('AfterLogActivity', 1, function($vars) {
    $aggregator = new LogAggregator(
        getenv('LOG_AGGREGATOR_ENDPOINT'),
        getenv('LOG_AGGREGATOR_API_KEY')
    );
    
    $aggregator->send([
        'timestamp' => $vars['datetime'] ?? date('c'),
        'type' => $vars['type'] ?? 'activity',
        'message' => $vars['description'],
        'user_id' => $vars['userid'] ?? null,
        'ip_address' => $vars['ipaddr'] ?? null,
        'whmcs_version' => \App::VERSION,
    ]);
});

/**
 * Rotate logs to prevent disk space issues
 */
function rotateLogs(int $maxSizeMB = 100, int $retentionDays = 30): void
{
    $logDir = ROOTDIR . '/storage/logs';
    
    // Find large log files
    $logs = glob("{$logDir}/*.log");
    
    foreach ($logs as $log) {
        $sizeMB = filesize($log) / (1024 * 1024);
        
        if ($sizeMB > $maxSizeMB) {
            // Rotate the log
            $rotated = $log . '.' . date('Y-m-d-His');
            rename($log, $rotated);
            
            // Compress rotated log
            exec("gzip -9 {$rotated}");
            
            // Create new empty log
            touch($log);
        }
    }
    
    // Delete old rotated logs
    $cutoff = date('Y-m-d', strtotime("-{$retentionDays} days"));
    $rotatedLogs = glob("{$logDir}/*.log.gz");
    
    foreach ($rotatedLogs as $rotated) {
        if (filemtime($rotated) < strtotime($cutoff)) {
            unlink($rotated);
        }
    }
}
```

## Best Practices

1. **Centralize logs** - Aggregate from multiple sources
2. **Set up alerts** - Notify on critical events
3. **Regular analysis** - Scheduled review of patterns
4. **Log rotation** - Prevent disk space issues
5. **Correlate events** - Link related log entries
6. **Retention policy** - Keep logs appropriately
7. **Secure logs** - Prevent tampering
8. **Performance impact** - Don't over-log

## Common Pitfalls to Avoid

1. **Ignoring logs** - Missing important issues
2. **Too much logging** - Performance impact
3. **No rotation** - Disk space exhaustion
4. **Scattered logs** - Hard to correlate
5. **No alerts** - Problems not detected
6. **Missing context** - Can't debug issues
7. **No retention** - Compliance issues
8. **Plain text passwords** - Security vulnerability
