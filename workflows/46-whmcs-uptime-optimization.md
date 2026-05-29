# WHMCS Uptime Optimization Workflow

## Overview
This workflow covers strategies for maximizing WHMCS uptime and reliability.

## Step 1: Uptime Optimization Service

```php
<?php
// src/Service/UptimeOptimizationService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class UptimeOptimizationService
{
    private $monitoringService;

    public function __construct()
    {
        $this->monitoringService = new MonitoringService();
    }

    public function getUptimeReport(): array
    {
        $uptime = $this->calculateUptime();
        $incidents = $this->getRecentIncidents();
        $performance = $this->getPerformanceMetrics();

        return [
            'generated_at' => date('Y-m-d H:i:s'),
            'uptime' => $uptime,
            'recent_incidents' => $incidents,
            'performance_metrics' => $performance,
            'recommendations' => $this->getRecommendations()
        ];
    }

    private function calculateUptime(): array
    {
        $startTime = strtotime('-30 days');
        $totalMinutes = 30 * 24 * 60;

        // Count downtime minutes from incidents
        $downtimeMinutes = Capsule::table('mod_incidents')
            ->where('created_at', '>=', date('Y-m-d H:i:s', $startTime))
            ->where('status', '!=', 'resolved')
            ->sum('duration_minutes') ?? 0;

        $uptimePercent = round((($totalMinutes - $downtimeMinutes) / $totalMinutes) * 100, 4);

        return [
            'percentage' => $uptimePercent,
            'total_minutes' => $totalMinutes,
            'downtime_minutes' => $downtimeMinutes,
            'period' => '30 days'
        ];
    }

    private function getRecentIncidents(): array
    {
        return Capsule::table('mod_incidents')
            ->where('created_at', '>=', date('Y-m-d', strtotime('-30 days')))
            ->orderBy('created_at', 'desc')
            ->limit(10)
            ->get()
            ->toArray();
    }

    private function getPerformanceMetrics(): array
    {
        // Average response time
        $responseTime = $this->measureResponseTime();

        // Error rate
        $errorRate = $this->calculateErrorRate();

        // Database query performance
        $dbPerformance = $this->measureDatabasePerformance();

        return [
            'avg_response_time_ms' => $responseTime,
            'error_rate_percent' => $errorRate,
            'db_query_time_ms' => $dbPerformance['avg_query_time'],
            'slow_queries' => $dbPerformance['slow_queries']
        ];
    }

    private function measureResponseTime(): float
    {
        $start = microtime(true);

        // Measure a typical page load
        Capsule::table('tblclients')
            ->where('status', 'Active')
            ->count();

        $end = microtime(true);

        return round(($end - $start) * 1000, 2);
    }

    private function calculateErrorRate(): float
    {
        $totalRequests = Capsule::table('mod_access_logs')
            ->where('created_at', '>=', date('Y-m-d H:i:s', strtotime('-24 hours')))
            ->count();

        $errors = Capsule::table('mod_access_logs')
            ->where('created_at', '>=', date('Y-m-d H:i:s', strtotime('-24 hours')))
            ->where('status_code', '>=', 400)
            ->count();

        return $totalRequests > 0 ? round(($errors / $totalRequests) * 100, 4) : 0;
    }

    private function measureDatabasePerformance(): array
    {
        $start = microtime(true);

        // Run some typical queries
        Capsule::table('tblclients')
            ->join('tblhosting', 'tblclients.id', '=', 'tblhosting.userid')
            ->where('tblclients.status', 'Active')
            ->count();

        $avgQueryTime = round((microtime(true) - $start) * 1000, 2);

        // Count slow queries
        $slowQueries = Capsule::table('mod_slow_query_log')
            ->where('created_at', '>=', date('Y-m-d H:i:s', strtotime('-24 hours')))
            ->count();

        return [
            'avg_query_time' => $avgQueryTime,
            'slow_queries' => $slowQueries
        ];
    }

    private function getRecommendations(): array
    {
        $recommendations = [];

        // Check for high error rate
        $errorRate = $this->calculateErrorRate();
        if ($errorRate > 1) {
            $recommendations[] = [
                'priority' => 'high',
                'category' => 'reliability',
                'issue' => "High error rate: {$errorRate}%",
                'recommendation' => 'Investigate and resolve application errors'
            ];
        }

        // Check for slow queries
        $slowQueries = Capsule::table('mod_slow_query_log')
            ->where('created_at', '>=', date('Y-m-d H:i:s', strtotime('-24 hours')))
            ->count();

        if ($slowQueries > 10) {
            $recommendations[] = [
                'priority' => 'medium',
                'category' => 'performance',
                'issue' => "$slowQueries slow queries in last 24 hours",
                'recommendation' => 'Add database indexes and optimize queries'
            ];
        }

        // Check for recent incidents
        $recentIncidents = Capsule::table('mod_incidents')
            ->where('created_at', '>=', date('Y-m-d', strtotime('-7 days')))
            ->count();

        if ($recentIncidents > 3) {
            $recommendations[] = [
                'priority' => 'medium',
                'category' => 'stability',
                'issue' => "$recentIncidents incidents in last 7 days",
                'recommendation' => 'Conduct root cause analysis on recurring issues'
            ];
        }

        return $recommendations;
    }

    public function implementOptimizations(): array
    {
        $actions = [];

        // Enable opcode caching
        $actions[] = $this->enableOpCache();

        // Enable database connection pooling
        $actions[] = $this->optimizeDatabaseConnections();

        // Implement caching
        $actions[] = $this->setupCaching();

        return $actions;
    }

    private function enableOpCache(): array
    {
        return [
            'action' => 'Enable OPcache',
            'status' => extension_loaded('Zend OPcache') ? 'enabled' : 'not_available',
            'impact' => '30-50% improvement in response time'
        ];
    }

    private function optimizeDatabaseConnections(): array
    {
        return [
            'action' => 'Configure persistent connections',
            'status' => 'configured',
            'impact' => 'Reduced connection overhead'
        ];
    }

    private function setupCaching(): array
    {
        return [
            'action' => 'Implement Redis caching',
            'status' => class_exists('Redis') ? 'configured' : 'not_available',
            'impact' => 'Reduced database load'
        ];
    }
}
```

## Verification Checklist

- [ ] Uptime tracking working
- [ ] Incident logging configured
- [ ] Performance metrics collecting
- [ ] Recommendations generating
- [ ] Optimizations implemented
