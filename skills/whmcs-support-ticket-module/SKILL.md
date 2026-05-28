# WHMCS Support Ticket Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building advanced support ticket management modules.

## When to Use

- Creating ticket management addons
- Building FAQ/Knowledge Base modules
- Implementing custom ticket workflows

## Support Ticket Patterns

```php
<?php
// modules/addons/{ticketmodule}/{ticketmodule}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {ticketmodule}_config(): array {
    return [
        'name' => 'Advanced Support Manager',
        'description' => 'Advanced ticket management with automation',
        'version' => '1.0',
        'author' => 'Author',
        'auto_assign' => ['FriendlyName' => 'Auto-assign Tickets', 'Type' => 'yesno'],
        'sla_enabled' => ['FriendlyName' => 'Enable SLA', 'Type' => 'yesno'],
        'first_response_sla' => ['FriendlyName' => 'First Response SLA (hours)', 'Type' => 'text', 'Default' => '4'],
        'resolution_sla' => ['FriendlyName' => 'Resolution SLA (hours)', 'Type' => 'text', 'Default' => '24'],
    ];
}

function {ticketmodule}_activate(): array {
    Capsule::schema()->create('mod_ticket_sla_timers', function($t) {
        $t->increments('id');
        $t->integer('ticket_id')->unsigned();
        $t->string('sla_type', 50);
        $t->timestamp('deadline');
        $t->string('status', 20)->default('pending');
        $t->timestamp('started_at');
        $t->timestamp('completed_at')->nullable();
    });

    Capsule::schema()->create('mod_ticket_knowledge_base', function($t) {
        $t->increments('id');
        $t->string('title', 255);
        $t->text('content');
        $t->string('category', 100);
        $t->string('tags', 255);
        $t->integer('views')->unsigned()->default(0);
        $t->integer('helpful')->unsigned()->default(0);
        $t->integer('not_helpful')->unsigned()->default(0);
        $t->boolean('active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_ticket_feedback', function($t) {
        $t->increments('id');
        $t->integer('ticket_id')->unsigned();
        $t->integer('admin_id')->unsigned();
        $t->integer('rating')->unsigned();
        $t->text('comment')->nullable();
        $t->timestamp('created_at');
    });

    Capsule::schema()->create('mod_ticket_escalations', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->text('condition');
        $t->string('action', 50);
        $t->text('action_data');
        $t->boolean('active')->default(true);
        $t->timestamp('created_at');
    });

    return ['status' => 'success', 'description' => 'Advanced support module activated'];
}

function {ticketmodule}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_ticket_sla_timers');
    Capsule::schema()->dropIfExists('mod_ticket_knowledge_base');
    Capsule::schema()->dropIfExists('mod_ticket_feedback');
    Capsule::schema()->dropIfExists('mod_ticket_escalations');
    return ['status' => 'success', 'description' => 'Support module deactivated'];
}

function {ticketmodule}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';

    echo '<div class="support-module">';
    echo '<h1>Advanced Support Manager</h1>';
    echo '</div>';

    switch ($action) {
        case 'sla':
            $this->showSLADashboard();
            break;
        case 'knowledge':
            $this->showKnowledgeBase();
            break;
        case 'feedback':
            $this->showFeedback();
            break;
        case 'escalations':
            $this->showEscalations();
            break;
        default:
            $this->showDashboard();
    }
}
```

### SLA Timer Management

```php
function startSLATimer(int $ticketId, string $slaType, int $hours): void {
    $deadline = date('Y-m-d H:i:s', strtotime("+{$hours} hours"));

    Capsule::table('mod_ticket_sla_timers')->insert([
        'ticket_id' => $ticketId,
        'sla_type' => $slaType,
        'deadline' => $deadline,
        'status' => 'active',
        'started_at' => date('Y-m-d H:i:s'),
    ]);
}

function checkSLAStatus(): array {
    $overdueTimers = Capsule::table('mod_ticket_sla_timers')
        ->where('status', 'active')
        ->where('deadline', '<', date('Y-m-d H:i:s'))
        ->get();

    $warnings = [];
    foreach ($overdueTimers as $timer) {
        $ticket = Capsule::table('tbltickets')->where('id', $timer->ticket_id)->first();
        $warnings[] = [
            'ticket_id' => $timer->ticket_id,
            'ticket_tid' => $ticket->tid ?? '',
            'title' => $ticket->title ?? '',
            'sla_type' => $timer->sla_type,
            'deadline' => $timer->deadline,
            'minutes_overdue' => (time() - strtotime($timer->deadline)) / 60,
        ];

        // Send escalation notification
        $this->notifySLABreach($timer);
    }

    return $warnings;
}

function completeSLATimer(int $ticketId, string $slaType): void {
    Capsule::table('mod_ticket_sla_timers')
        ->where('ticket_id', $ticketId)
        ->where('sla_type', $slaType)
        ->where('status', 'active')
        ->update([
            'status' => 'completed',
            'completed_at' => date('Y-m-d H:i:s'),
        ]);
}
```

### Knowledge Base

```php
function searchKnowledgeBase(string $query, int $limit = 10): array {
    $articles = Capsule::table('mod_ticket_knowledge_base')
        ->where('active', 1)
        ->where(function($q) use ($query) {
            $q->where('title', 'like', "%$query%")
              ->orWhere('content', 'like', "%$query%")
              ->orWhere('tags', 'like', "%$query%");
        })
        ->orderBy('helpful', 'desc')
        ->limit($limit)
        ->get();

    // Increment view count
    foreach ($articles as $article) {
        Capsule::table('mod_ticket_knowledge_base')
            ->where('id', $article->id)
            ->increment('views');
    }

    return $articles;
}

function submitFeedback(int $ticketId, int $adminId, int $rating, string $comment = ''): void {
    Capsule::table('mod_ticket_feedback')->insert([
        'ticket_id' => $ticketId,
        'admin_id' => $adminId,
        'rating' => $rating,
        'comment' => $comment,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

function getAverageFeedbackRating(array $filters = []): float {
    $query = Capsule::table('mod_ticket_feedback');

    if (!empty($filters['from_date'])) {
        $query->where('created_at', '>=', $filters['from_date']);
    }

    if (!empty($filters['admin_id'])) {
        $query->where('admin_id', $filters['admin_id']);
    }

    return $query->avg('rating') ?? 0;
}
```

### Escalation Engine

```php
function processTicketEscalations(int $ticketId): void {
    $ticket = Capsule::table('tbltickets')->where('id', $ticketId)->first();
    if (!$ticket) return;

    $escalations = Capsule::table('mod_ticket_escalations')
        ->where('active', 1)
        ->get();

    foreach ($escalations as $escalation) {
        if ($this->evaluateEscalationCondition($escalation->condition, $ticket)) {
            $this->executeEscalationAction($escalation->action, $escalation->action_data, $ticket);
        }
    }
}

function evaluateEscalationCondition(string $condition, $ticket): bool {
    // Simple condition evaluator
    $data = [
        'status' => $ticket->status,
        'priority' => $ticket->urgency,
        'created_at' => $ticket->created,
        'waiting_time' => (time() - strtotime($ticket->created)) / 3600,
    ];

    extract($data);
    return eval('return ' . $condition . ';');
}

function executeEscalationAction(string $action, string $actionData, $ticket): void {
    $data = json_decode($actionData, true);

    switch ($action) {
        case 'assign_admin':
            Capsule::table('tbltickets')
                ->where('id', $ticket->id)
                ->update(['adminid' => $data['admin_id']]);
            break;

        case 'change_priority':
            Capsule::table('tbltickets')
                ->where('id', $ticket->id)
                ->update(['urgency' => $data['priority']]);
            break;

        case 'send_email':
            // Implementation for email notification
            break;

        case 'add_tag':
            $currentTags = Capsule::table('tblticket滤s')
                ->where('id', $ticket->id)
                ->value('tags') ?? '';
            $newTags = $currentTags . ',' . $data['tag'];
            Capsule::table('tbltickets')
                ->where('id', $ticket->id)
                ->update(['tags' => $newTags]);
            break;
    }
}
```

### Hook Integration

```php
add_hook('TicketOpen', 1, function($vars) {
    $slaEnabled = Capsule::table('mod_configuration')
        ->where('setting', 'sla_enabled')
        ->value('value');

    if ($slaEnabled) {
        $firstResponse = (int) Capsule::table('mod_configuration')
            ->where('setting', 'first_response_sla')
            ->value('value') ?? 4;

        startSLATimer($vars['ticketid'], 'first_response', $firstResponse);
    }
});

add_hook('TicketReply', 1, function($vars) {
    completeSLATimer($vars['ticketid'], 'first_response');
});

add_hook('TicketClose', 1, function($vars) {
    $resolutionSla = (int) Capsule::table('mod_configuration')
        ->where('setting', 'resolution_sla')
        ->value('value') ?? 24;

    startSLATimer($vars['ticketid'], 'resolution', $resolutionSla);
});

add_hook('TicketClose', 1, function($vars) {
    completeSLATimer($vars['ticketid'], 'resolution');
    processTicketEscalations($vars['ticketid']);
});
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-hooks-development
- whmcs-admin-ui-builder
