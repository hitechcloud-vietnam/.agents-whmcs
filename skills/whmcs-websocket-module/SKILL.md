# WHMCS WebSocket Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing real-time WebSocket communication in WHMCS modules.

## When to Use

- Real-time dashboard updates
- Live chat features
- Server status monitoring

## WebSocket Patterns

```php
<?php
// modules/addons/{module}/websocket.php

use Ratchet\MessageComponentInterface;
use Ratchet\ConnectionInterface;

class WebSocketServer implements MessageComponentInterface {
    protected $clients;

    public function __construct() {
        $this->clients = new \SplObjectStorage;
    }

    public function onOpen(ConnectionInterface $conn) {
        $this->clients->attach($conn);
        logActivity('{Module} WebSocket: New connection ' . $conn->resourceId);
    }

    public function onMessage(ConnectionInterface $from, $msg) {
        $data = json_decode($msg, true);

        switch ($data['action'] ?? '') {
            case 'subscribe':
                $this->subscribeClient($from, $data['service_id']);
                break;
            case 'unsubscribe':
                $this->unsubscribeClient($from);
                break;
        }
    }

    public function onClose(ConnectionInterface $conn) {
        $this->clients->detach($conn);
    }

    public function onError(ConnectionInterface $conn, \Exception $e) {
        logActivity('{Module} WebSocket Error: ' . $e->getMessage());
        $conn->close();
    }

    public function broadcast(array $message): void {
        foreach ($this->clients as $client) {
            $client->send(json_encode($message));
        }
    }

    private function subscribeClient($conn, int $serviceId): void {
        $conn->serviceId = $serviceId;
        $conn->send(json_encode(['action' => 'subscribed', 'service_id' => $serviceId]));
    }
}
```

### Client-Side JavaScript
```javascript
const ws = new WebSocket('wss://example.com/ws');

ws.onopen = () => {
    ws.send(JSON.stringify({
        action: 'subscribe',
        service_id: 123
    }));
};

ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    updateDashboard(data);
};
```

---

**Related Skills:**
- whmcs-ajax-patterns
- whmcs-admin-ui-builder
- whmcs-server-builder
