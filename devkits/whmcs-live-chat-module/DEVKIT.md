# WHMCS Live Chat Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## Purpose
Create a live chat module for WHMCS that provides real-time customer support through chat integration with third-party providers.

## Module Type
Addon Module with Chat Provider Integration

## Use Case
- Real-time customer support chat
- Chat operator dashboard
- Chat history and transcripts
- Automated chat routing
- Chat widget integration

## DevKit Structure

```
devkits/whmcs-live-chat-module/
├── livechat.php          # Main addon module
├── lib/
│   ├── ChatProvider.php   # Chat provider interface
│   ├── ChatOperator.php   # Operator management
│   └── ChatSession.php    # Chat session handling
├── templates/
│   └── admin.tpl         # Admin templates
├── widget.tpl            # Chat widget
├── hooks.php              # Hook integrations
└── DEVKIT.md            # This file
```

## Main Module Template

```php
<?php
/**
 * WHMCS Live Chat Module: {LiveChat}
 * Live Chat Module Template
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {livechat}_config(): array {
    return [
        'name' => '{Live Chat}',
        'description' => 'Real-time live chat support',
        'version' => '1.0',
        'author' => '{Author Name}',

        'provider' => [
            'FriendlyName' => 'Chat Provider',
            'Type' => 'dropdown',
            'Options' => 'custom,intercom,livechat,tawkto,zendesk',
            'Default' => 'custom',
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
        ],
        'widget_id' => [
            'FriendlyName' => 'Widget ID',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'Your chat widget identifier',
        ],
        'auto_assign' => [
            'FriendlyName' => 'Auto-Assign Chats',
            'Type' => 'yesno',
            'Description' => 'Automatically assign chats to available operators',
        ],
        'require_login' => [
            'FriendlyName' => 'Require Login',
            'Type' => 'yesno',
            'Description' => 'Require customer to login before chatting',
        ],
        'offline_message' => [
            'FriendlyName' => 'Show Offline Form',
            'Type' => 'yesno',
            'Description' => 'Show offline message form when no operators available',
        ],
        'chat_timeout' => [
            'FriendlyName' => 'Chat Timeout (minutes)',
            'Type' => 'text',
            'Size' => '5',
            'Default' => '30',
            'Description' => 'Minutes before inactive chat is closed',
        ],
        'max_conversations' => [
            'FriendlyName' => 'Max Conversations per Operator',
            'Type' => 'text',
            'Size' => '5',
            'Default' => '5',
        ],
    ];
}

function {livechat}_activate(): array {
    try {
        // Operators table
        if (!Capsule::schema()->hasTable('mod_{livechat}_operators')) {
            Capsule::schema()->create('mod_{livechat}_operators', function($t) {
                $t->increments('id');
                $t->integer('admin_id')->unsigned()->unique();
                $t->string('status', 20)->default('offline');
                $t->integer('active_chats')->default(0);
                $t->integer('max_chats')->default(5);
                $t->timestamp('last_activity');
                $t->timestamps();
            });
        }

        // Chat sessions table
        if (!Capsule::schema()->hasTable('mod_{livechat}_sessions')) {
            Capsule::schema()->create('mod_{livechat}_sessions', function($t) {
                $t->increments('id');
                $t->string('session_id', 100)->unique();
                $t->integer('operator_id')->unsigned()->nullable();
                $t->integer('client_id')->unsigned()->nullable();
                $t->string('client_name', 100);
                $t->string('client_email', 255);
                $t->string('status', 20)->default('waiting');
                $t->string('department', 100)->nullable();
                $t->text('custom_data')->nullable();
                $t->timestamp('started_at');
                $t->timestamp('ended_at')->nullable();
                $t->integer('messages_count')->default(0);
                $t->timestamps();

                $t->index('status');
                $t->index('operator_id');
            });
        }

        // Chat messages table
        if (!Capsule::schema()->hasTable('mod_{livechat}_messages')) {
            Capsule::schema()->create('mod_{livechat}_messages', function($t) {
                $t->increments('id');
                $t->integer('session_id')->unsigned();
                $t->integer('sender_id')->unsigned()->nullable();
                $t->string('sender_type', 20); // client, operator, system
                $t->text('message');
                $t->string('message_type', 20)->default('text');
                $t->timestamp('sent_at');
                $t->timestamps();

                $t->index('session_id');
            });
        }

        // Chat departments table
        if (!Capsule::schema()->hasTable('mod_{livechat}_departments')) {
            Capsule::schema()->create('mod_{livechat}_departments', function($t) {
                $t->increments('id');
                $t->string('name', 100);
                $t->text('description')->nullable();
                $t->integer('sort_order')->default(0);
                $t->boolean('is_active')->default(true);
                $t->timestamps();
            });
        }

        // Chat ratings table
        if (!Capsule::schema()->hasTable('mod_{livechat}_ratings')) {
            Capsule::schema()->create('mod_{livechat}_ratings', function($t) {
                $t->increments('id');
                $t->integer('session_id')->unsigned()->unique();
                $t->integer('rating')->unsigned();
                $t->text('comment')->nullable();
                $t->timestamp('created_at');
            });
        }

        return ['status' => 'success', 'description' => '{Live Chat} activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => 'Activation failed: ' . $e->getMessage()];
    }
}

function {livechat}_deactivate(): array {
    try {
        Capsule::schema()->dropIfExists('mod_{livechat}_operators');
        Capsule::schema()->dropIfExists('mod_{livechat}_sessions');
        Capsule::schema()->dropIfExists('mod_{livechat}_messages');
        Capsule::schema()->dropIfExists('mod_{livechat}_departments');
        Capsule::schema()->dropIfExists('mod_{livechat}_ratings');
        return ['status' => 'success'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => 'Deactivation failed'];
    }
}

function {livechat}_output(array $vars): void {
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
        handleChatAction($_POST['action'] ?? '');
    }

    $tab = $_REQUEST['tab'] ?? 'dashboard';
    echo '<div class="livechat-module">';
    echo '<h1><i class="fa fa-comments"></i> Live Chat</h1>';
    echo '<ul class="nav nav-tabs">';
    echo '<li class="' . ($tab === 'dashboard' ? 'active' : '') . '"><a href="?module={livechat}&tab=dashboard">Dashboard</a></li>';
    echo '<li class="' . ($tab === 'operators' ? 'active' : '') . '"><a href="?module={livechat}&tab=operators">Operators</a></li>';
    echo '<li class="' . ($tab === 'history' ? 'active' : '') . '"><a href="?module={livechat}&tab=history">History</a></li>';
    echo '<li class="' . ($tab === 'departments' ? 'active' : '') . '"><a href="?module={livechat}&tab=departments">Departments</a></li>';
    echo '<li class="' . ($tab === 'settings' ? 'active' : '') . '"><a href="?module={livechat}&tab=settings">Settings</a></li>';
    echo '</ul>';
    include __DIR__ . '/templates/admin/' . $tab . '.tpl';
    echo '</div>';
}

function {livechat}_clientarea(array $vars): array {
    return [
        'pagetitle' => 'Live Support Chat',
        'templatefile' => 'templates/chat_widget',
        'vars' => [
            'widget_id' => getWidgetId(),
            'customer' => getCustomerData(),
        ],
    ];
}

function handleChatAction(string $action): void {
    switch ($action) {
        case 'update_operator_status':
            updateOperatorStatus((int)($_POST['operator_id'] ?? 0), $_POST['status'] ?? 'online');
            break;
        case 'close_session':
            closeChatSession((int)($_POST['session_id'] ?? 0));
            break;
        case 'transfer_session':
            transferSession((int)($_POST['session_id'] ?? 0), (int)($_POST['operator_id'] ?? 0));
            break;
        case 'save_department':
            saveDepartment();
            break;
    }
    header('Location: ?module={livechat}&tab=' . ($_POST['redirect_tab'] ?? 'dashboard'));
    exit;
}

function getChatStats(): array {
    return [
        'active_chats' => Capsule::table('mod_{livechat}_sessions')->where('status', 'active')->count(),
        'waiting' => Capsule::table('mod_{livechat}_sessions')->where('status', 'waiting')->count(),
        'today_chats' => Capsule::table('mod_{livechat}_sessions')->whereDate('started_at', date('Y-m-d'))->count(),
        'avg_rating' => Capsule::table('mod_{livechat}_ratings')->avg('rating') ?? 0,
    ];
}

function getActiveSessions(): array {
    return Capsule::table('mod_{livechat}_sessions')
        ->whereIn('status', ['active', 'waiting'])
        ->orderBy('started_at', 'asc')
        ->get()
        ->toArray();
}

function getSessionMessages(int $sessionId): array {
    return Capsule::table('mod_{livechat}_messages')
        ->where('session_id', $sessionId)
        ->orderBy('sent_at', 'asc')
        ->get()
        ->toArray();
}

function closeChatSession(int $sessionId): void {
    Capsule::table('mod_{livechat}_sessions')
        ->where('id', $sessionId)
        ->update([
            'status' => 'closed',
            'ended_at' => date('Y-m-d H:i:s'),
        ]);

    // Update operator chat count
    $session = Capsule::table('mod_{livechat}_sessions')->where('id', $sessionId)->first();
    if ($session && $session->operator_id) {
        Capsule::table('mod_{livechat}_operators')
            ->where('id', $session->operator_id)
            ->decrement('active_chats');
    }
}

function transferSession(int $sessionId, int $operatorId): void {
    Capsule::table('mod_{livechat}_sessions')
        ->where('id', $sessionId)
        ->update(['operator_id' => $operatorId]);

    addSystemMessage($sessionId, "Chat transferred to another operator");
}

function addSystemMessage(int $sessionId, string $message): void {
    Capsule::table('mod_{livechat}_messages')->insert([
        'session_id' => $sessionId,
        'sender_type' => 'system',
        'message' => $message,
        'sent_at' => date('Y-m-d H:i:s'),
    ]);
}

function getWidgetId(): string {
    $settings = Capsule::table('tbladdon_modules')->where('module', '{livechat}')->first();
    $config = $settings ? json_decode($settings->value, true) : [];
    return $config['widget_id'] ?? '';
}

function getCustomerData(): array {
    if (empty($_SESSION['uid'])) {
        return ['logged_in' => false];
    }

    $client = Capsule::table('tblclients')->where('id', $_SESSION['uid'])->first();

    return [
        'logged_in' => true,
        'id' => $client->id,
        'name' => $client->firstname . ' ' . $client->lastname,
        'email' => $client->email,
    ];
}
```

## Chat Provider Interface

```php
<?php
namespace WHMCS\Module\Addon\{LiveChat};

interface ChatProviderInterface {
    public function sendMessage(int $sessionId, string $message, string $senderType): array;
    public function getOnlineOperators(): array;
    public function createSession(array $customerData): array;
    public function closeSession(int $sessionId): array;
    public function getChatHistory(int $sessionId): array;
}
```

## Chat Session Handler

```php
<?php
namespace WHMCS\Module\Addon\{LiveChat};

use WHMCS\Database\Capsule;

class ChatSession {

    public function createSession(array $data): int {
        $sessionId = bin2hex(random_bytes(16));

        return Capsule::table('mod_{livechat}_sessions')->insertGetId([
            'session_id' => $sessionId,
            'client_id' => $data['client_id'] ?? null,
            'client_name' => $data['client_name'] ?? 'Guest',
            'client_email' => $data['client_email'] ?? '',
            'department' => $data['department'] ?? null,
            'custom_data' => json_encode($data['custom_data'] ?? []),
            'status' => 'waiting',
            'started_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function assignOperator(int $sessionId, int $operatorId): bool {
        $settings = getModuleSettings('{livechat}');

        // Check if operator can accept more chats
        $operator = Capsule::table('mod_{livechat}_operators')
            ->where('id', $operatorId)
            ->first();

        if (!$operator || $operator->active_chats >= $operator->max_chats) {
            return false;
        }

        Capsule::table('mod_{livechat}_sessions')
            ->where('id', $sessionId)
            ->update([
                'operator_id' => $operatorId,
                'status' => 'active',
            ]);

        Capsule::table('mod_{livechat}_operators')
            ->where('id', $operatorId)
            ->increment('active_chats');

        // Add system message
        Capsule::table('mod_{livechat}_messages')->insert([
            'session_id' => $sessionId,
            'sender_type' => 'system',
            'message' => 'An operator has joined the chat',
            'sent_at' => date('Y-m-d H:i:s'),
        ]);

        return true;
    }

    public function autoAssign(int $sessionId): bool {
        $settings = getModuleSettings('{livechat}');

        if (($settings['auto_assign'] ?? '') !== 'on') {
            return false;
        }

        // Find available operator
        $operator = Capsule::table('mod_{livechat}_operators')
            ->where('status', 'online')
            ->whereRaw('active_chats < max_chats')
            ->orderBy('active_chats', 'asc')
            ->first();

        if ($operator) {
            return $this->assignOperator($sessionId, $operator->id);
        }

        return false;
    }

    public function sendMessage(int $sessionId, string $message, string $senderType, int $senderId = 0): int {
        $messageId = Capsule::table('mod_{livechat}_messages')->insertGetId([
            'session_id' => $sessionId,
            'sender_id' => $senderId,
            'sender_type' => $senderType,
            'message' => $message,
            'sent_at' => date('Y-m-d H:i:s'),
        ]);

        // Update message count
        Capsule::table('mod_{livechat}_sessions')
            ->where('id', $sessionId)
            ->increment('messages_count');

        // Update operator activity
        if ($senderType === 'operator' && $senderId) {
            Capsule::table('mod_{livechat}_operators')
                ->where('id', $senderId)
                ->update(['last_activity' => date('Y-m-d H:i:s')]);
        }

        return $messageId;
    }

    public function endSession(int $sessionId): void {
        Capsule::table('mod_{livechat}_sessions')
            ->where('id', $sessionId)
            ->update([
                'status' => 'closed',
                'ended_at' => date('Y-m-d H:i:s'),
            ]);

        $session = Capsule::table('mod_{livechat}_sessions')->where('id', $sessionId)->first();

        if ($session && $session->operator_id) {
            Capsule::table('mod_{livechat}_operators')
                ->where('id', $session->operator_id)
                ->decrement('active_chats');
        }

        // Add closing message
        Capsule::table('mod_{livechat}_messages')->insert([
            'session_id' => $sessionId,
            'sender_type' => 'system',
            'message' => 'Chat session ended',
            'sent_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function rateSession(int $sessionId, int $rating, ?string $comment = null): void {
        Capsule::table('mod_{livechat}_ratings')->insert([
            'session_id' => $sessionId,
            'rating' => $rating,
            'comment' => $comment,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function getActiveSessions(): array {
        return Capsule::table('mod_{livechat}_sessions')
            ->whereIn('status', ['waiting', 'active'])
            ->with('messages')
            ->orderBy('started_at', 'asc')
            ->get()
            ->toArray();
    }

    public function getSession(int $sessionId): ?object {
        return Capsule::table('mod_{livechat}_sessions')
            ->where('id', $sessionId)
            ->first();
    }
}
```

## Chat Widget Template

```smarty
<div id="livechat-widget" class="livechat-widget collapsed">
    <div class="livechat-header" onclick="toggleChat()">
        <div class="livechat-title">
            <i class="fa fa-comments"></i>
            <span>Live Chat</span>
        </div>
        <div class="livechat-status">
            <span class="status-indicator online"></span>
            <span class="status-text">Online</span>
        </div>
    </div>

    <div class="livechat-body" style="display: none;">
        <div id="chat-messages" class="chat-messages">
            <div class="chat-welcome">
                <p>Welcome! How can we help you today?</p>
            </div>
        </div>

        <form id="chat-form" class="chat-form">
            <input type="hidden" name="session_id" id="session_id">
            <input type="text" name="name" id="chat-name" placeholder="Your Name" {if $customer.logged_in}value="{$customer.name}" readonly{/if} required>
            <input type="email" name="email" id="chat-email" placeholder="Your Email" {if $customer.logged_in}value="{$customer.email}" readonly{/if} required>
            <textarea name="message" id="chat-message" placeholder="Type your message..." rows="3" required></textarea>
            <button type="submit" class="btn btn-primary">
                <i class="fa fa-paper-plane"></i> Send
            </button>
        </form>
    </div>
</div>

<script>
(function() {
    var widgetId = '{$widget_id}';
    var baseUrl = '{$base_url}';
    var sessionId = null;

    function toggleChat() {
        var widget = document.getElementById('livechat-widget');
        var body = widget.querySelector('.livechat-body');

        if (widget.classList.contains('collapsed')) {
            widget.classList.remove('collapsed');
            body.style.display = 'block';
            initChat();
        } else {
            widget.classList.add('collapsed');
            body.style.display = 'none';
        }
    }

    function initChat() {
        // Start chat session
        fetch(baseUrl + 'modules/addons/{livechat}/api.php?action=start', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                name: document.getElementById('chat-name').value,
                email: document.getElementById('chat-email').value
            })
        })
        .then(r => r.json())
        .then(data => {
            if (data.session_id) {
                sessionId = data.session_id;
                document.getElementById('session_id').value = sessionId;
                pollMessages();
            }
        });
    }

    function pollMessages() {
        if (!sessionId) return;

        setInterval(function() {
            fetch(baseUrl + 'modules/addons/{livechat}/api.php?action=messages&session_id=' + sessionId)
                .then(r => r.json())
                .then(displayMessages);
        }, 3000);
    }

    function displayMessages(messages) {
        var container = document.getElementById('chat-messages');

        messages.forEach(function(msg) {
            if (!document.getElementById('msg-' + msg.id)) {
                var div = document.createElement('div');
                div.id = 'msg-' + msg.id;
                div.className = 'chat-message ' + msg.sender_type;
                div.innerHTML = '<strong>' + msg.sender_name + ':</strong> ' + msg.message;
                container.appendChild(div);
            }
        });

        container.scrollTop = container.scrollHeight;
    }

    document.getElementById('chat-form').addEventListener('submit', function(e) {
        e.preventDefault();

        var formData = new FormData(this);

        fetch(baseUrl + 'modules/addons/{livechat}/api.php?action=send', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                session_id: formData.get('session_id'),
                message: formData.get('message')
            })
        })
        .then(r => r.json())
        .then(data => {
            if (data.success) {
                document.getElementById('chat-message').value = '';
            }
        });
    });

    window.toggleChat = toggleChat;
})();
</script>
```

## Hooks Integration

```php
<?php
/**
 * WHMCS Live Chat Module Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Notify operator of new chat
add_hook('TicketOpen', 1, function(array $vars) {
    $ticketId = $vars['ticketid'];

    // Check if customer came from live chat
    $session = Capsule::table('mod_{livechat}_sessions')
        ->where('client_id', $vars['userid'])
        ->where('status', 'closed')
        ->orderBy('ended_at', 'desc')
        ->first();

    if ($session) {
        logActivity("{LiveChat}: Customer from chat session #{$session->id} created ticket #{$ticketId}");
    }
});

// End chat when service is suspended
add_hook('AfterModuleSuspend', 1, function(array $vars) {
    // Notify active chats about service status
});
```

## Database Schema

### mod_{livechat}_sessions
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| session_id | VARCHAR(100) | Unique session identifier |
| operator_id | INT | Assigned operator |
| client_id | INT | WHMCS client ID |
| client_name | VARCHAR(100) | Customer name |
| client_email | VARCHAR(255) | Customer email |
| status | VARCHAR(20) | waiting/active/closed |
| department | VARCHAR(100) | Chat department |
| started_at | TIMESTAMP | Chat start time |
| ended_at | TIMESTAMP | Chat end time |
| messages_count | INT | Message count |

### mod_{livechat}_messages
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| session_id | INT | Session FK |
| sender_id | INT | Sender (operator/client) |
| sender_type | VARCHAR(20) | client/operator/system |
| message | TEXT | Message content |
| sent_at | TIMESTAMP | Sent time |

## Checklist

```
Pre-Dev:
□ Choose chat provider (custom, Intercom, etc.)
□ Plan operator management
□ Design chat widget
□ Define chat routing

Development:
□ Implement config() with all settings
□ Implement activate() → Create tables
□ Implement deactivate() → Drop tables
□ Create ChatProvider interface
□ Create ChatSession handler
□ Implement operator management
□ Build session creation
□ Create message handling
□ Build chat widget template
□ Create admin dashboard
□ Add API endpoints

Security:
□ Validate all inputs
□ Sanitize chat messages
□ Implement rate limiting
□ Secure operator authentication

Testing:
□ Test chat widget
□ Test session creation
□ Test message sending
□ Test auto-assignment
□ Verify operator dashboard
□ Test chat history
```
