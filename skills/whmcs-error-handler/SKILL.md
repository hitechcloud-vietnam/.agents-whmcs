# WHMCS Error Handler Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing comprehensive error handling in WHMCS modules.

## When to Use

- Building robust error handling systems
- Creating custom error pages
- Implementing logging and alerting

## Error Handler Patterns

```php
<?php
// Error Handler Class
class {Module}ErrorHandler {
    private static array $errors = [];
    private static string $logPath = '';

    public static function register(): void {
        set_error_handler([self::class, 'handleError']);
        set_exception_handler([self::class, 'handleException']);
        register_shutdown_function([self::class, 'shutdownHandler']);
    }

    public static function handleError(int $level, string $message, string $file, int $line): void {
        if (!(error_reporting() & $level)) {
            return;
        }

        $error = [
            'type' => 'error',
            'level' => $level,
            'message' => $message,
            'file' => $file,
            'line' => $line,
            'timestamp' => date('Y-m-d H:i:s'),
        ];

        self::log($error);
        self::$errors[] = $error;
    }

    public static function handleException(\Throwable $e): void {
        $error = [
            'type' => 'exception',
            'message' => $e->getMessage(),
            'file' => $e->getFile(),
            'line' => $e->getLine(),
            'trace' => $e->getTraceAsString(),
            'timestamp' => date('Y-m-d H:i:s'),
        ];

        self::log($error);
        self::$errors[] = $error;

        if (self::isProduction()) {
            self::notifyAdmin($error);
        }
    }

    public static function shutdownHandler(): void {
        $error = error_get_last();
        if ($error && in_array($error['type'], [E_ERROR, E_CORE_ERROR, E_COMPILE_ERROR])) {
            self::log([
                'type' => 'fatal',
                'message' => $error['message'],
                'file' => $error['file'],
                'line' => $error['line'],
                'timestamp' => date('Y-m-d H:i:s'),
            ]);
        }
    }

    private static function log(array $error): void {
        logActivity('{Module} Error: ' . json_encode($error));
    }

    private static function isProduction(): bool {
        return Capsule::table('mod_{module}_settings')
            ->where('key', 'environment')
            ->value('value') !== 'development';
    }

    private static function notifyAdmin(array $error): void {
        // Send admin notification
    }
}
```

---

**Related Skills:**
- whmcs-logging
- whmcs-admin-ui-builder
- whmcs-deployment
