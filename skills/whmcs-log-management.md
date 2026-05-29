# WHMCS Log Management

## Skill Description
Set up centralized logging for WHMCS modules to track events, troubleshoot issues, and maintain audit trails.

## Prerequisites
- WHMCS installation
- Log management system
- Storage space

## Step-by-Step Implementation

### 1. Logger Class
```php
<?php
// includes/logging/Logger.php

namespace WHMCS\Module\YourModule\Logging;

class Logger
{
    private string $channel;
    private string $logPath;
    private array $levels = [
        'emergency' => 0,
        'alert' => 1,
        'critical' => 2,
        'error' => 3,
        'warning' => 4,
        'notice' => 5,
        'info' => 6,
        'debug' => 7
    ];

    public function __construct(string $channel = 'default', string $logPath = null)
    {
        $this->channel = $channel;
        $this->logPath = $logPath ?? dirname(__DIR__) . '/logs';

        if (!is_dir($this->logPath)) {
            mkdir($this->logPath, 0755, true);
        }
    }

    public function log(string $level, string $message, array $context = []): void
    {
        if (!$this->shouldLog($level)) {
            return;
        }

        $logEntry = $this->formatEntry($level, $message, $context);

        $filename = $this->getLogFile($level);
        file_put_contents($filename, $logEntry . PHP_EOL, FILE_APPEND);
    }

    public function emergency(string $message, array $context = []): void
    {
        $this->log('emergency', $message, $context);
    }

    public function error(string $message, array $context = []): void
    {
        $this->log('error', $message, $context);
    }

    public function warning(string $message, array $context = []): void
    {
        $this->log('warning', $message, $context);
    }

    public function info(string $message, array $context = []): void
    {
        $this->log('info', $message, $context);
    }

    public function debug(string $message, array $context = []): void
    {
        $this->log('debug', $message, $context);
    }

    private function shouldLog(string $level): bool
    {
        $minLevel = strtolower(getenv('LOG_LEVEL') ?: 'info');
        return $this->levels[$level] <= $this->levels[$minLevel];
    }

    private function formatEntry(string $level, string $message, array $context): string
    {
        $entry = [
            'timestamp' => date('Y-m-d H:i:s'),
            'level' => strtoupper($level),
            'channel' => $this->channel,
            'message' => $message
        ];

        if (!empty($context)) {
            $entry['context'] = $context;
        }

        return json_encode($entry);
    }

    private function getLogFile(string $level): string
    {
        $date = date('Y-m-d');
        return $this->logPath . "/{$this->channel}_{$level}_{$date}.log";
    }

    public function rotate(int $daysToKeep = 30): int
    {
        $cutoff = strtotime("-{$daysToKeep} days");
        $files = glob($this->logPath . "/*.log");
        $count = 0;

        foreach ($files as $file) {
            if (filemtime($file) < $cutoff) {
                unlink($file);
                $count++;
            }
        }

        return $count;
    }
}
```

### 2. Log Reader
```php
<?php
// includes/logging/LogReader.php

namespace WHMCS\Module\YourModule\Logging;

class LogReader
{
    private string $logPath;

    public function __construct(string $logPath = null)
    {
        $this->logPath = $logPath ?? dirname(__DIR__) . '/logs';
    }

    public function read(string $level = null, int $limit = 100): array
    {
        $files = glob($this->logPath . "/*.log");

        if ($level) {
            $files = array_filter($files, fn($f) => strpos($f, "_{$level}_") !== false);
        }

        $entries = [];

        foreach ($files as $file) {
            $lines = file($file, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);

            foreach (array_slice($lines, -$limit) as $line) {
                $entries[] = json_decode($line, true);
            }
        }

        usort($entries, fn($a, $b) => strtotime($b['timestamp']) - strtotime($a['timestamp']));

        return array_slice($entries, 0, $limit);
    }

    public function search(string $query, int $limit = 100): array
    {
        $entries = $this->read(null, 1000);

        return array_filter($entries, function ($entry) use ($query) {
            return stripos($entry['message'], $query) !== false
                || stripos(json_encode($entry), $query) !== false;
        });
    }

    public function getStats(): array
    {
        $files = glob($this->logPath . "/*.log");

        $stats = [
            'total_files' => count($files),
            'total_size' => 0,
            'oldest' => null,
            'newest' => null
        ];

        foreach ($files as $file) {
            $stats['total_size'] += filesize($file);

            $mtime = filemtime($file);

            if ($stats['oldest'] === null || $mtime < $stats['oldest']) {
                $stats['oldest'] = date('Y-m-d H:i:s', $mtime);
            }

            if ($stats['newest'] === null || $mtime > $stats['newest']) {
                $stats['newest'] = date('Y-m-d H:i:s', $mtime);
            }
        }

        return $stats;
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Large log files | Implement rotation |
| Missing logs | Check permissions |
| Slow queries | Use indexed search |

## Security Considerations

1. **Protect log files** - Set permissions
2. **Sanitize data** - Remove sensitive info
3. **Secure storage** - Encrypt at rest

## Testing Checklist

- [ ] Test logging
- [ ] Test rotation
- [ ] Test search

## Reference Links

- [PSR-3 Logger](https://www.php-fig.org/psr/psr-3/)
