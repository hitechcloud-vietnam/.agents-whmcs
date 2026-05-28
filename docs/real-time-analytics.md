# Real-time Analytics Patterns

Real-time analytics patterns enable immediate insight into WHMCS operations through streaming data processing and live dashboards.

## Streaming Architecture

### Event Stream

```php
<?php
/**
 * Event stream for real-time analytics
 */
class EventStream
{
    private string $streamName;
    private array $partitions = [];
    private int $retentionDays = 7;

    public function __construct(string $streamName)
    {
        $this->streamName = $streamName;
    }

    /**
     * Publish an event to the stream
     */
    public function publish(string $eventType, array $data, ?string $partitionKey = null): string
    {
        $event = [
            'event_id' => $this->generateEventId(),
            'stream' => $this->streamName,
            'event_type' => $eventType,
            'partition_key' => $partitionKey ?? $this->defaultPartitionKey($data),
            'data' => $data,
            'timestamp' => microtime(true),
            'processed' => false,
        ];

        // Store in stream table
        Capsule::table('mod_event_stream')->insert($event);

        // Trigger async processing
        $this->triggerProcessing($event['event_id']);

        return $event['event_id'];
    }

    /**
     * Subscribe to event stream
     */
    public function subscribe(string $consumerGroup, callable $handler, array $eventTypes = []): void
    {
        $offset = $this->getConsumerOffset($consumerGroup);

        $query = Capsule::table('mod_event_stream')
            ->where('id', '>', $offset)
            ->where('processed', false)
            ->where('stream', $this->streamName)
            ->orderBy('id', 'asc')
            ->limit(100);

        if (!empty($eventTypes)) {
            $query->whereIn('event_type', $eventTypes);
        }

        $events = $query->get();

        foreach ($events as $event) {
            try {
                $handler($event);
                $this->acknowledgeEvent($consumerGroup, $event->id);
            } catch (\Throwable $e) {
                $this->handleFailedEvent($event, $e);
            }
        }
    }

    /**
     * Get stream statistics
     */
    public function getStats(): array
    {
        $total = Capsule::table('mod_event_stream')
            ->where('stream', $this->streamName)
            ->count();

        $unprocessed = Capsule::table('mod_event_stream')
            ->where('stream', $this->streamName)
            ->where('processed', false)
            ->count();

        $lastHour = Capsule::table('mod_event_stream')
            ->where('stream', $this->streamName)
            ->where('timestamp', '>', microtime(true) - 3600)
            ->count();

        return [
            'stream' => $this->streamName,
            'total_events' => $total,
            'unprocessed' => $unprocessed,
            'last_hour_count' => $lastHour,
        ];
    }

    private function generateEventId(): string
    {
        return sprintf('%016x-%04x-%04x',
            time(),
            mt_rand(0, 0xffff),
            mt_rand(0, 0xffff)
        );
    }

    private function defaultPartitionKey(array $data): string
    {
        if (isset($data['client_id'])) {
            return 'client:' . $data['client_id'];
        }

        if (isset($data['userid'])) {
            return 'client:' . $data['userid'];
        }

        return 'default';
    }

    private function getConsumerOffset(string $consumerGroup): int
    {
        $offset = Capsule::table('mod_stream_offsets')
            ->where('consumer_group', $consumerGroup)
            ->where('stream', $this->streamName)
            ->value('offset');

        return $offset ?? 0;
    }

    private function acknowledgeEvent(string $consumerGroup, int $eventId): void
    {
        Capsule::table('mod_stream_offsets')
            ->updateOrInsert(
                ['consumer_group' => $consumerGroup, 'stream' => $this->streamName],
                ['offset' => $eventId]
            );

        Capsule::table('mod_event_stream')
            ->where('id', $eventId)
            ->update(['processed' => true]);
    }

    private function handleFailedEvent($event, \Throwable $e): void
    {
        Capsule::table('mod_event_stream')->where('id', $event->id)->update([
            'error' => $e->getMessage(),
            'retry_count' => ($event->retry_count ?? 0) + 1,
        ]);

        logActivity("Stream processing failed for event {$event->id}: " . $e->getMessage());
    }

    private function triggerProcessing(string $eventId): void
    {
        // Store processing job
        Capsule::table('mod_stream_processing')->insert([
            'event_id' => $eventId,
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

## Real-time Aggregations

### Streaming Aggregations

```php
<?php
<?php
/**
 * Real-time aggregation counter
 */
class StreamingCounter
{
    private string $name;
    private string $aggregationType;
    private array $groupBy = [];
    private int $windowSeconds;
    private string $table;
    private int $lastFlush = 0;

    public function __construct(string $name, string $aggregationType = 'COUNT', int $windowSeconds = 60)
    {
        $this->name = $name;
        $this->aggregationType = $aggregationType;
        $this->windowSeconds = $windowSeconds;
        $this->table = 'mod_realtime_aggregations';
    }

    public function groupBy(array $columns): self
    {
        $this->groupBy = $columns;
        return $this;
    }

    /**
     * Increment the counter
     */
    public function increment(array $dimensions = [], float $value = 1): void
    {
        $windowKey = $this->getWindowKey();
        $dimensionKey = $this->getDimensionKey($dimensions);

        $exists = Capsule::table($this->table)
            ->where('metric_name', $this->name)
            ->where('window_key', $windowKey)
            ->where('dimension_key', $dimensionKey)
            ->exists();

        if ($exists) {
            Capsule::table($this->table)
                ->where('metric_name', $this->name)
                ->where('window_key', $windowKey)
                ->where('dimension_key', $dimensionKey)
                ->increment('value', $value);
        } else {
            Capsule::table($this->table)->insert([
                'metric_name' => $this->name,
                'window_key' => $windowKey,
                'dimension_key' => $dimensionKey,
                'dimensions' => json_encode($dimensions),
                'value' => $value,
                'created_at' => date('Y-m-d H:i:s'),
            ]);
        }
    }

    /**
     * Get current value
     */
    public function getValue(array $dimensions = []): float
    {
        $windowKey = $this->getWindowKey();
        $dimensionKey = $this->getDimensionKey($dimensions);

        $result = Capsule::table($this->table)
            ->where('metric_name', $this->name)
            ->where('window_key', $windowKey)
            ->where('dimension_key', $dimensionKey)
            ->value('value');

        return $result ?? 0;
    }

    /**
     * Get aggregated value over time range
     */
    public function getHistory(int $minutes = 60, array $dimensions = []): array
    {
        $cutoff = microtime(true) - ($minutes * 60);
        $dimensionKey = empty($dimensions) ? '' : $this->getDimensionKey($dimensions);

        $query = Capsule::table($this->table)
            ->where('metric_name', $this->name)
            ->where('created_at', '>', date('Y-m-d H:i:s', $cutoff));

        if ($dimensionKey) {
            $query->where('dimension_key', $dimensionKey);
        }

        return $query
            ->orderBy('created_at', 'asc')
            ->get();
    }

    private function getWindowKey(): string
    {
        $window = floor(microtime(true) / $this->windowSeconds) * $this->windowSeconds;
        return sprintf('%012d', (int) $window);
    }

    private function getDimensionKey(array $dimensions): string
    {
        ksort($dimensions);
        return md5(json_encode($dimensions));
    }

    /**
     * Clean up old windows
     */
    public function cleanup(int $retentionMinutes = 1440): void
    {
        $cutoff = microtime(true) - ($retentionMinutes * 60);

        Capsule::table($this->table)
            ->where('created_at', '<', date('Y-m-d H:i:s', $cutoff))
            ->delete();
    }
}

/**
 * Unique counter (HyperLogLog approximation)
 */
class UniqueCounter
{
    private string $name;
    private int $precision;
    private array $registers = [];
    private int $windowSeconds;

    public function __construct(string $name, int $precision = 12, int $windowSeconds = 60)
    {
        $this->name = $name;
        $this->precision = $precision;
        $this->windowSeconds = $windowSeconds;
        $this->initRegisters();
    }

    private function initRegisters(): void
    {
        $this->registers = array_fill(0, (1 << $this->precision), 0);
    }

    public function add(string $identifier): void
    {
        $hash = crc32($identifier);
        $index = $hash & ((1 << $this->precision) - 1);
        $zeroCount = $this->countTrailingZeros($hash >> $this->precision);

        $this->registers[$index] = max($this->registers[$index], $zeroCount);
    }

    public function count(): int
    {
        $sum = 0;

        foreach ($this->registers as $register) {
            $sum += pow(2, -$register);
        }

        $m = 1 << $this->precision;
        $alpha = 0.673;

        if ($m == 16) {
            $alpha = 0.697;
        } elseif ($m == 32) {
            $alpha = 0.709;
        } elseif ($m >= 128) {
            $alpha = 0.7213;
        }

        $estimate = $alpha * $m * $m / $sum;

        if ($estimate < 2.5 * $m) {
            $zeros = count(array_filter($this->registers, fn($r) => $r === 0));
            if ($zeros > 0) {
                $estimate = $m * log($m / $zeros);
            }
        }

        return (int) round($estimate);
    }

    private function countTrailingZeros(int $value): int
    {
        $count = 0;

        while (($value & 1) === 0 && $count < 32) {
            $value >>= 1;
            $count++;
        }

        return $count;
    }
}
```

## Live Dashboards

### Dashboard Data Provider

```php
<?php
<?php
/**
 * Real-time dashboard data provider
 */
class DashboardProvider
{
    /**
     * Get dashboard metrics
     */
    public function getMetrics(): array
    {
        return [
            'realtime' => $this->getRealtimeMetrics(),
            'today' => $this->getTodayMetrics(),
            'trends' => $this->getTrendData(),
            'alerts' => $this->getActiveAlerts(),
        ];
    }

    private function getRealtimeMetrics(): array
    {
        $minuteAgo = date('Y-m-d H:i:s', strtotime('-1 minute'));

        return [
            'active_users' => $this->getActiveUsers(),
            'requests_last_minute' => $this->getRequestCount($minuteAgo),
            'orders_last_minute' => $this->getOrderCount($minuteAgo),
            'new_signups_last_minute' => $this->getSignupCount($minuteAgo),
            'queue_depth' => $this->getQueueDepth(),
        ];
    }

    private function getTodayMetrics(): array
    {
        $today = date('Y-m-d');
        $todayStart = $today . ' 00:00:00';
        $todayEnd = $today . ' 23:59:59';

        return [
            'revenue_today' => $this->getRevenue($todayStart, $todayEnd),
            'orders_today' => $this->getOrders($todayStart, $todayEnd),
            'new_clients_today' => $this->getNewClients($todayStart, $todayEnd),
            'invoices_generated' => $this->getInvoicesGenerated($todayStart, $todayEnd),
            'tickets_opened' => $this->getTicketsOpened($todayStart, $todayEnd),
        ];
    }

    private function getTrendData(): array
    {
        $data = [];
        $labels = [];

        for ($i = 6; $i >= 0; $i--) {
            $date = date('Y-m-d', strtotime("-{$i} days"));
            $labels[] = date('D', strtotime("-{$i} days"));

            $data['revenue'][] = $this->getRevenue(
                $date . ' 00:00:00',
                $date . ' 23:59:59'
            );

            $data['orders'][] = $this->getOrders(
                $date . ' 00:00:00',
                $date . ' 23:59:59'
            );

            $data['new_clients'][] = $this->getNewClients(
                $date . ' 00:00:00',
                $date . ' 23:59:59'
            );
        }

        return [
            'labels' => $labels,
            'datasets' => $data,
        ];
    }

    private function getActiveUsers(): int
    {
        $cutoff = date('Y-m-d H:i:s', strtotime('-5 minutes'));

        return Capsule::table('tblactivitylog')
            ->where('date', '>', $cutoff)
            ->distinct('userid')
            ->count('userid');
    }

    private function getRequestCount(string $since): int
    {
        return Capsule::table('mod_event_stream')
            ->where('event_type', 'api_request')
            ->where('timestamp', '>', strtotime($since))
            ->count();
    }

    private function getOrderCount(string $since): int
    {
        return Capsule::table('tblorders')
            ->where('date', '>', $since)
            ->count();
    }

    private function getSignupCount(string $since): int
    {
        return Capsule::table('tblclients')
            ->where('datecreated', '>', $since)
            ->count();
    }

    private function getQueueDepth(): int
    {
        return Capsule::table('mod_job_queue')
            ->whereIn('status', ['pending', 'processing'])
            ->count();
    }

    private function getRevenue(string $start, string $end): float
    {
        $result = Capsule::table('tblorders')
            ->where('date', '>=', $start)
            ->where('date', '<=', $end)
            ->where('status', 'Completed')
            ->sum('total');

        return (float) ($result ?? 0);
    }

    private function getOrders(string $start, string $end): int
    {
        return Capsule::table('tblorders')
            ->where('date', '>=', $start)
            ->where('date', '<=', $end)
            ->count();
    }

    private function getNewClients(string $start, string $end): int
    {
        return Capsule::table('tblclients')
            ->where('datecreated', '>=', $start)
            ->where('datecreated', '<=', $end)
            ->count();
    }

    private function getInvoicesGenerated(string $start, string $end): int
    {
        return Capsule::table('tblinvoiceitems')
            ->where('invoiced', '>=', strtotime($start))
            ->where('invoiced', '<=', strtotime($end))
            ->count();
    }

    private function getTicketsOpened(string $start, string $end): int
    {
        return Capsule::table('tbltickets')
            ->where('date', '>=', strtotime($start))
            ->where('date', '<=', strtotime($end))
            ->count();
    }

    private function getActiveAlerts(): array
    {
        return Capsule::table('mod_dashboard_alerts')
            ->where('acknowledged', false)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->orderBy('severity', 'desc')
            ->limit(10)
            ->get();
    }
}
```

## Real-time Charts

### Chart Data Generators

```php
<?php
<?php
/**
 * Chart data generators for real-time dashboards
 */
class ChartDataGenerator
{
    /**
     * Generate time series data
     */
    public static function timeSeries(
        string $metric,
        string $timestampColumn,
        string $valueColumn,
        int $intervalMinutes = 60,
        int $dataPoints = 24
    ): array {
        $labels = [];
        $values = [];
        $intervalSeconds = $intervalMinutes * 60;

        for ($i = $dataPoints - 1; $i >= 0; $i--) {
            $timestamp = floor((time() - ($i * $intervalSeconds)) / $intervalSeconds) * $intervalSeconds;
            $labels[] = date('H:i', $timestamp);

            // Get value for this interval
            $value = self::getIntervalValue($metric, $timestampColumn, $valueColumn, $timestamp, $intervalSeconds);
            $values[] = $value;
        }

        return [
            'labels' => $labels,
            'values' => $values,
        ];
    }

    /**
     * Generate pie chart data
     */
    public static function pieChart(string $table, string $groupColumn, string $valueColumn, array $filters = []): array
    {
        $query = Capsule::table($table)
            ->selectRaw("{$groupColumn} as label, SUM({$valueColumn}) as value")
            ->groupBy($groupColumn);

        foreach ($filters as $column => $value) {
            $query->where($column, $value);
        }

        $results = $query->get();

        return [
            'labels' => array_column($results, 'label'),
            'values' => array_map('floatval', array_column($results, 'value')),
        ];
    }

    /**
     * Generate bar chart data
     */
    public static function barChart(
        string $metric,
        string $groupByColumn,
        int $limit = 10
    ): array {
        $query = Capsule::table('mod_realtime_aggregations')
            ->selectRaw("dimensions->'$.{$groupByColumn}' as label, SUM(value) as value")
            ->where('metric_name', $metric)
            ->groupBy($groupByColumn)
            ->orderByRaw('SUM(value) DESC')
            ->limit($limit);

        $results = $query->get();

        return [
            'labels' => array_column($results, 'label'),
            'values' => array_map('floatval', array_column($results, 'value')),
        ];
    }

    /**
     * Generate funnel data
     */
    public static function funnel(string $table, array $stages): array
    {
        $data = [];

        foreach ($stages as $stage) {
            $count = Capsule::table($table)
                ->where($stage['column'], $stage['operator'] ?? '=', $stage['value'] ?? true)
                ->count();

            $data[] = [
                'stage' => $stage['label'],
                'count' => $count,
            ];
        }

        return $data;
    }

    private static function getIntervalValue(
        string $metric,
        string $timestampColumn,
        string $valueColumn,
        int $timestamp,
        int $intervalSeconds
    ): float {
        $start = date('Y-m-d H:i:s', $timestamp);
        $end = date('Y-m-d H:i:s', $timestamp + $intervalSeconds);

        // This is a simplified example - actual implementation depends on your data model
        $result = Capsule::table('mod_event_stream')
            ->where('event_type', $metric)
            ->where('timestamp', '>=', $timestamp)
            ->where('timestamp', '<', $timestamp + $intervalSeconds)
            ->count();

        return (float) $result;
    }
}

/**
 * Revenue chart generator
 */
class RevenueChartGenerator
{
    public static function hourly(): array
    {
        $labels = [];
        $values = [];

        for ($hour = 23; $hour >= 0; $hour--) {
            $start = strtotime(date('Y-m-d') . " {$hour}:00:00");
            $end = $start + 3600;

            $revenue = Capsule::table('tblorders')
                ->where('date', '>=', date('Y-m-d H:i:s', $start))
                ->where('date', '<', date('Y-m-d H:i:s', $end))
                ->where('status', 'Completed')
                ->sum('total');

            $labels[] = date('H:00', $start);
            $values[] = (float) ($revenue ?? 0);
        }

        return [
            'type' => 'bar',
            'labels' => $labels,
            'datasets' => [['label' => 'Revenue', 'data' => $values]],
        ];
    }

    public static function byProduct(): array
    {
        $products = Capsule::table('tblorders')
            ->selectRaw('productname as label, SUM(total) as value')
            ->where('status', 'Completed')
            ->where('date', '>=', date('Y-m-d', strtotime('-30 days')))
            ->groupBy('productname')
            ->orderByRaw('SUM(total) DESC')
            ->limit(10)
            ->get();

        return [
            'type' => 'doughnut',
            'labels' => array_column($products, 'label'),
            'datasets' => [['data' => array_map('floatval', array_column($products, 'value'))]],
        ];
    }
}
```

## WebSocket Updates

### Real-time Push

```php
<?php
<?php
/**
 * Real-time event pusher
 */
class RealtimePusher
{
    private array $channels = [];
    private array $subscribers = [];

    /**
     * Broadcast to a channel
     */
    public function broadcast(string $channel, string $event, array $data): void
    {
        $message = json_encode([
            'channel' => $channel,
            'event' => $event,
            'data' => $data,
            'timestamp' => microtime(true),
        ]);

        // Store in broadcast table for processing
        Capsule::table('mod_realtime_broadcasts')->insert([
            'channel' => $channel,
            'event' => $event,
            'data' => json_encode($data),
            'message' => $message,
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        // Notify connected clients (requires separate WebSocket server)
        $this->notifyClients($channel, $message);
    }

    /**
     * Subscribe a client to a channel
     */
    public function subscribe(string $clientId, string $channel): void
    {
        $key = "{$clientId}:{$channel}";

        Capsule::table('mod_realtime_subscriptions')->updateOrInsert(
            ['client_id' => $clientId, 'channel' => $channel],
            ['subscribed_at' => date('Y-m-d H:i:s')]
        );
    }

    /**
     * Unsubscribe a client from a channel
     */
    public function unsubscribe(string $clientId, string $channel): void
    {
        Capsule::table('mod_realtime_subscriptions')
            ->where('client_id', $clientId)
            ->where('channel', $channel)
            ->delete();
    }

    private function notifyClients(string $channel, string $message): void
    {
        // This would connect to a WebSocket server
        // For example, using Redis pub/sub or a dedicated service
    }
}

/**
 * WHMCS real-time events
 */
class WhmcsRealtimeEvents
{
    public static function orderPlaced(int $orderId): void
    {
        $order = Capsule::table('tblorders')->where('id', $orderId)->first();

        $pusher = new RealtimePusher();
        $pusher->broadcast('dashboard', 'order.placed', [
            'order_id' => $orderId,
            'client_id' => $order->userid,
            'total' => $order->total,
            'timestamp' => time(),
        ]);
    }

    public static function paymentReceived(int $invoiceId, float $amount): void
    {
        $pusher = new RealtimePusher();
        $pusher->broadcast('dashboard', 'payment.received', [
            'invoice_id' => $invoiceId,
            'amount' => $amount,
            'timestamp' => time(),
        ]);

        // Also broadcast to revenue channel
        $pusher->broadcast('revenue', 'update', [
            'amount' => $amount,
            'timestamp' => time(),
        ]);
    }

    public static function clientCreated(int $clientId): void
    {
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();

        $pusher = new RealtimePusher();
        $pusher->broadcast('dashboard', 'client.created', [
            'client_id' => $clientId,
            'name' => $client->firstname . ' ' . $client->lastname,
            'timestamp' => time(),
        ]);
    }
}

/**
 * Hook into WHMCS events for real-time updates
 */
add_hook('AfterModuleCreate', 1, function ($vars) {
    // Notify of service provisioning
});

add_hook('InvoicePaid', 1, function ($vars) {
    WhmcsRealtimeEvents::paymentReceived($vars['invoice_id'], $vars['amount'] ?? 0);
});

add_hook('ClientAdd', 1, function ($vars) {
    WhmcsRealtimeEvents::clientCreated($vars['client_id']);
});
```

## Best Practices

1. **Use appropriate windows** - Choose aggregation windows based on use case
2. **Implement backpressure** - Handle slow consumers gracefully
3. **Clean up old data** - Prune historical stream data regularly
4. **Batch writes** - Combine events for efficiency
5. **Use efficient data structures** - HyperLogLog for unique counts
6. **Monitor latency** - Track end-to-end processing time
7. **Provide fallbacks** - Handle WebSocket disconnections
8. **Secure channels** - Authenticate subscribers

## Related Patterns

- [Event Sourcing](./event-sourcing.md) - Event-driven architecture
- [Message Queue](./message-queue.md) - Async processing
- [Observability Patterns](./observability-patterns.md) - Metrics collection
