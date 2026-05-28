# WHMCS Audit Log Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing comprehensive audit logging.

## When to Use

- Compliance requirements
- Security monitoring
- User activity tracking

## Audit Log Patterns

```php
<?php
class AuditLogger {
    private string $table = 'mod_audit_logs';

    public function log(string $action, array $context = [], string $level = 'info'): void {
        Capsule::table($this->table)->insert([
            'action' => $action,
            'user_id' => $_SESSION['uid'] ?? 0,
            'admin_id' => $_SESSION['adminid'] ?? 0,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'context' => json_encode($context),
            'level' => $level,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function logLogin(int $userId, bool $success, string $reason = ''): void {
        $this->log('login', [
            'user_id' => $userId,
            'success' => $success,
            'reason' => $reason,
        ], $success ? 'info' : 'warning');
    }

    public function logDataChange(string $entity, int $entityId, array $oldData, array $newData): void {
        $this->log('data_change', [
            'entity' => $entity,
            'entity_id' => $entityId,
            'old_data' => $oldData,
            'new_data' => $newData,
        ], 'info');
    }

    public function getRecentLogs(int $limit = 100): array {
        return Capsule::table($this->table)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get();
    }

    public function searchLogs(array $criteria): array {
        $query = Capsule::table($this->table);

        if (!empty($criteria['user_id'])) {
            $query->where('user_id', $criteria['user_id']);
        }

        if (!empty($criteria['action'])) {
            $query->where('action', 'like', '%' . $criteria['action'] . '%');
        }

        if (!empty($criteria['from_date'])) {
            $query->where('created_at', '>=', $criteria['from_date']);
        }

        if (!empty($criteria['to_date'])) {
            $query->where('created_at', '<=', $criteria['to_date']);
        }

        return $query->get();
    }
}
```

---

**Related Skills:**
- whmcs-logging
- whmcs-security-hardening
- whmcs-reporting
