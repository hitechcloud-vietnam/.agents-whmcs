# WHMCS Live Chat Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building live chat modules for WHMCS.

## When to Use

- Customer support modules
- Pre-sale chat
- Real-time communication

## Live Chat Patterns

```php
<?php
class LiveChatModule {
    private array $operators = [];
    private array $sessions = [];

    public function startChat(int $userId, string $message): array {
        $sessionId = $this->createSession($userId);

        return [
            'session_id' => $sessionId,
            'status' => 'waiting',
            'message' => 'You are now in queue. Position: ' . $this->getQueuePosition($sessionId),
        ];
    }

    private function createSession(int $userId): string {
        $sessionId = bin2hex(random_bytes(16));

        $this->sessions[$sessionId] = [
            'user_id' => $userId,
            'started_at' => date('Y-m-d H:i:s'),
            'status' => 'active',
            'operator_id' => null,
            'messages' => [],
        ];

        return $sessionId;
    }

    public function sendMessage(string $sessionId, string $message, string $sender = 'user'): array {
        if (!isset($this->sessions[$sessionId])) {
            return ['error' => 'Session not found'];
        }

        $this->sessions[$sessionId]['messages'][] = [
            'sender' => $sender,
            'message' => $message,
            'timestamp' => date('Y-m-d H:i:s'),
        ];

        // Notify operator
        if ($sender === 'user') {
            $this->notifyOperator($sessionId, $message);
        }

        // Log message
        Capsule::table('mod_chat_messages')->insert([
            'session_id' => $sessionId,
            'sender' => $sender,
            'message' => $message,
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        return ['success' => true];
    }

    public function assignOperator(string $sessionId, int $operatorId): array {
        $this->sessions[$sessionId]['operator_id'] = $operatorId;
        $this->sessions[$sessionId]['assigned_at'] = date('Y-m-d H:i:s');

        return [
            'success' => true,
            'operator' => $this->getOperatorInfo($operatorId),
        ];
    }

    public function endChat(string $sessionId): array {
        $this->sessions[$sessionId]['status'] = 'ended';
        $this->sessions[$sessionId]['ended_at'] = date('Y-m-d H:i:s');

        return ['success' => true];
    }
}
```

### WebSocket Integration
```php
public function onMessage(ConnectionInterface $from, $msg) {
    $data = json_decode($msg, true);

    switch ($data['action']) {
        case 'message':
            $this->sendMessage($data['session_id'], $data['message'], 'user');
            // Broadcast to all in session
            break;

        case 'typing':
            // Broadcast typing indicator
            break;
    }
}
```

---

**Related Skills:**
- whmcs-websocket-module
- whmcs-support-ticket-module
