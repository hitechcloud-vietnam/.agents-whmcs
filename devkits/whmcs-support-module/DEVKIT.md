# WHMCS Support Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## Purpose
Create an enhanced support and helpdesk module for WHMCS with knowledge base, canned responses, and SLA management.

## Module Type
Addon Module

## Use Case
- Enhanced support ticket management
- Knowledge base for self-service
- Canned responses for quick replies
- SLA policy enforcement
- Customer satisfaction tracking

## DevKit Structure

```
devkits/whmcs-support-module/
├── support.php           # Main addon module
├── lib/
│   ├── TicketManager.php  # Ticket management
│   ├── KnowledgeBase.php  # Knowledge base
│   └── CannedResponses.php # Canned responses
├── templates/
│   ├── admin.tpl          # Admin templates
│   └── client.tpl         # Client templates
├── hooks.php             # Hook integrations
└── DEVKIT.md            # This file
```

## Main Module Template

```php
<?php
/**
 * WHMCS Support Module: {Support}
 * Support/Helpdesk Module Template
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {support}_config(): array {
    return [
        'name' => '{Support Module}',
        'description' => 'Enhanced support and helpdesk features',
        'version' => '1.0',
        'author' => '{Author Name}',
        'language' => 'english',

        'enable_kb' => [
            'FriendlyName' => 'Enable Knowledge Base',
            'Type' => 'yesno',
        ],
        'enable_canned' => [
            'FriendlyName' => 'Enable Canned Responses',
            'Type' => 'yesno',
        ],
        'enable_sla' => [
            'FriendlyName' => 'Enable SLA Policies',
            'Type' => 'yesno',
        ],
        'enable_feedback' => [
            'FriendlyName' => 'Enable Feedback',
            'Type' => 'yesno',
        ],
        'auto_close_days' => [
            'FriendlyName' => 'Auto-Close Days',
            'Type' => 'text',
            'Size' => '5',
            'Default' => '7',
            'Description' => 'Days before auto-closing resolved tickets',
        ],
    ];
}

function {support}_activate(): array {
    try {
        // Knowledge Base Articles
        if (!Capsule::schema()->hasTable('mod_{support}_kb_articles')) {
            Capsule::schema()->create('mod_{support}_kb_articles', function($t) {
                $t->increments('id');
                $t->string('title', 255);
                $t->text('content');
                $t->string('slug', 255)->unique();
                $t->integer('category_id')->unsigned()->nullable();
                $t->boolean('is_published')->default(false);
                $t->integer('views')->default(0);
                $t->integer('helpful_yes')->default(0);
                $t->integer('helpful_no')->default(0);
                $t->string('meta_title', 255)->nullable();
                $t->text('meta_description')->nullable();
                $t->timestamps();

                $t->index('category_id');
                $t->index('is_published');
            });
        }

        // Knowledge Base Categories
        if (!Capsule::schema()->hasTable('mod_{support}_kb_categories')) {
            Capsule::schema()->create('mod_{support}_kb_categories', function($t) {
                $t->increments('id');
                $t->string('name', 100);
                $t->text('description')->nullable();
                $t->string('slug', 100)->unique();
                $t->integer('parent_id')->unsigned()->nullable();
                $t->integer('sort_order')->default(0);
                $t->timestamps();

                $t->index('parent_id');
            });
        }

        // Canned Responses
        if (!Capsule::schema()->hasTable('mod_{support}_canned_responses')) {
            Capsule::schema()->create('mod_{support}_canned_responses', function($t) {
                $t->increments('id');
                $t->string('title', 150);
                $t->text('content');
                $t->string('category', 100)->nullable();
                $t->string('shortcode', 50)->nullable();
                $t->boolean('is_global')->default(false);
                $t->integer('use_count')->default(0);
                $t->integer('created_by')->unsigned();
                $t->timestamps();

                $t->index('category');
                $t->index('shortcode');
            });
        }

        // Ticket Feedback
        if (!Capsule::schema()->hasTable('mod_{support}_ticket_feedback')) {
            Capsule::schema()->create('mod_{support}_ticket_feedback', function($t) {
                $t->increments('id');
                $t->integer('ticket_id')->unsigned()->unique();
                $t->integer('rating')->unsigned();
                $t->text('comment')->nullable();
                $t->string('categories', 255)->nullable();
                $t->timestamp('created_at');

                $t->index('rating');
            });
        }

        // SLA Policies
        if (!Capsule::schema()->hasTable('mod_{support}_sla_policies')) {
            Capsule::schema()->create('mod_{support}_sla_policies', function($t) {
                $t->increments('id');
                $t->string('name', 100);
                $t->string('priority', 20);
                $t->integer('first_response_hours');
                $t->integer('resolution_hours');
                $t->boolean('is_active')->default(true);
                $t->timestamps();
            });
        }

        // Insert default SLA policies
        $defaultPolicies = [
            ['name' => 'Critical', 'priority' => 'High', 'first_response_hours' => 1, 'resolution_hours' => 4],
            ['name' => 'High', 'priority' => 'Medium', 'first_response_hours' => 4, 'resolution_hours' => 24],
            ['name' => 'Normal', 'priority' => 'Low', 'first_response_hours' => 24, 'resolution_hours' => 72],
        ];

        foreach ($defaultPolicies as $policy) {
            Capsule::table('mod_{support}_sla_policies')->insert($policy);
        }

        return [
            'status' => 'success',
            'description' => '{Support Module} activated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Activation failed: ' . $e->getMessage(),
        ];
    }
}

function {support}_deactivate(): array {
    try {
        Capsule::schema()->dropIfExists('mod_{support}_kb_articles');
        Capsule::schema()->dropIfExists('mod_{support}_kb_categories');
        Capsule::schema()->dropIfExists('mod_{support}_canned_responses');
        Capsule::schema()->dropIfExists('mod_{support}_ticket_feedback');
        Capsule::schema()->dropIfExists('mod_{support}_sla_policies');

        return [
            'status' => 'success',
            'description' => '{Support Module} deactivated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Deactivation failed: ' . $e->getMessage(),
        ];
    }
}

function {support}_output(array $vars): void {
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
        handleSupportAction($_POST['action'] ?? '');
    }

    $action = $_REQUEST['action'] ?? 'dashboard';
    $tab = $_REQUEST['tab'] ?? 'overview';

    echo '<div class="support-module">';
    echo '<div class="support-header">';
    echo '<h1><i class="fa fa-life-ring"></i> Support Module</h1>';
    echo '</div>';

    // Navigation tabs
    echo '<ul class="nav nav-tabs">';
    echo '<li class="' . ($tab === 'overview' ? 'active' : '') . '"><a href="?module={support}&tab=overview">Overview</a></li>';
    echo '<li class="' . ($tab === 'kb' ? 'active' : '') . '"><a href="?module={support}&tab=kb">Knowledge Base</a></li>';
    echo '<li class="' . ($tab === 'canned' ? 'active' : '') . '"><a href="?module={support}&tab=canned">Canned Responses</a></li>';
    echo '<li class="' . ($tab === 'sla' ? 'active' : '') . '"><a href="?module={support}&tab=sla">SLA Policies</a></li>';
    echo '<li class="' . ($tab === 'feedback' ? 'active' : '') . '"><a href="?module={support}&tab=feedback">Feedback</a></li>';
    echo '</ul>';

    include __DIR__ . '/templates/admin/' . $tab . '.tpl';
    echo '</div>';
}

function {support}_clientarea(array $vars): array {
    return [
        'pagetitle' => 'Support Center',
        'templatefile' => 'templates/clientarea/clientarea',
        'vars' => [
            'categories' => getKbCategories(),
            'popular_articles' => getPopularArticles(5),
            'recent_articles' => getRecentArticles(5),
        ],
        'requirelogin' => true,
    ];
}

// Helper Functions
function handleSupportAction(string $action): void {
    switch ($action) {
        case 'save_article':
            saveArticle();
            break;
        case 'delete_article':
            deleteArticle((int)($_GET['id'] ?? 0));
            break;
        case 'save_canned':
            saveCannedResponse();
            break;
        case 'delete_canned':
            deleteCannedResponse((int)($_GET['id'] ?? 0));
            break;
        case 'save_sla':
            saveSlaPolicy();
            break;
        case 'delete_sla':
            deleteSlaPolicy((int)($_GET['id'] ?? 0));
            break;
    }

    header('Location: ?module={support}&tab=' . ($_POST['redirect_tab'] ?? 'overview'));
    exit;
}

function getKbCategories(): array {
    return Capsule::table('mod_{support}_kb_categories')
        ->orderBy('sort_order')
        ->get()
        ->toArray();
}

function getPopularArticles(int $limit = 10): array {
    return Capsule::table('mod_{support}_kb_articles')
        ->where('is_published', 1)
        ->orderBy('views', 'desc')
        ->limit($limit)
        ->get()
        ->toArray();
}

function getRecentArticles(int $limit = 10): array {
    return Capsule::table('mod_{support}_kb_articles')
        ->where('is_published', 1)
        ->orderBy('created_at', 'desc')
        ->limit($limit)
        ->get()
        ->toArray();
}

function saveArticle(): void {
    $id = (int)($_POST['id'] ?? 0);
    $data = [
        'title' => $_POST['title'] ?? '',
        'slug' => generateSlug($_POST['title'] ?? ''),
        'content' => $_POST['content'] ?? '',
        'category_id' => !empty($_POST['category_id']) ? (int)$_POST['category_id'] : null,
        'is_published' => isset($_POST['is_published']) ? 1 : 0,
        'meta_title' => $_POST['meta_title'] ?? null,
        'meta_description' => $_POST['meta_description'] ?? null,
    ];

    if ($id > 0) {
        Capsule::table('mod_{support}_kb_articles')
            ->where('id', $id)
            ->update($data);
    } else {
        Capsule::table('mod_{support}_kb_articles')->insert($data);
    }
}

function deleteArticle(int $id): void {
    Capsule::table('mod_{support}_kb_articles')
        ->where('id', $id)
        ->delete();
}

function saveCannedResponse(): void {
    $id = (int)($_POST['id'] ?? 0);
    $data = [
        'title' => $_POST['title'] ?? '',
        'content' => $_POST['content'] ?? '',
        'category' => $_POST['category'] ?? '',
        'shortcode' => $_POST['shortcode'] ?? '',
        'is_global' => isset($_POST['is_global']) ? 1 : 0,
    ];

    if ($id > 0) {
        Capsule::table('mod_{support}_canned_responses')
            ->where('id', $id)
            ->update($data);
    } else {
        $data['created_by'] = $_SESSION['adminid'];
        Capsule::table('mod_{support}_canned_responses')->insert($data);
    }
}

function deleteCannedResponse(int $id): void {
    Capsule::table('mod_{support}_canned_responses')
        ->where('id', $id)
        ->delete();
}

function saveSlaPolicy(): void {
    $id = (int)($_POST['id'] ?? 0);
    $data = [
        'name' => $_POST['name'] ?? '',
        'priority' => $_POST['priority'] ?? 'Low',
        'first_response_hours' => (int)($_POST['first_response_hours'] ?? 24),
        'resolution_hours' => (int)($_POST['resolution_hours'] ?? 72),
        'is_active' => isset($_POST['is_active']) ? 1 : 0,
    ];

    if ($id > 0) {
        Capsule::table('mod_{support}_sla_policies')
            ->where('id', $id)
            ->update($data);
    } else {
        Capsule::table('mod_{support}_sla_policies')->insert($data);
    }
}

function deleteSlaPolicy(int $id): void {
    Capsule::table('mod_{support}_sla_policies')
        ->where('id', $id)
        ->delete();
}

function generateSlug(string $title): string {
    $slug = strtolower(trim($title));
    $slug = preg_replace('/[^a-z0-9-]/', '-', $slug);
    $slug = preg_replace('/-+/', '-', $slug);

    return trim($slug, '-') . '-' . substr(md5(uniqid()), 0, 6);
}
```

## Knowledge Base Class

```php
<?php
namespace WHMCS\Module\Addon\{Support};

use WHMCS\Database\Capsule;

class KnowledgeBase {

    public function getCategories(): array {
        return Capsule::table('mod_{support}_kb_categories')
            ->orderBy('sort_order')
            ->get()
            ->toArray();
    }

    public function getCategoryBySlug(string $slug): ?object {
        return Capsule::table('mod_{support}_kb_categories')
            ->where('slug', $slug)
            ->first();
    }

    public function getArticlesByCategory(int $categoryId): array {
        return Capsule::table('mod_{support}_kb_articles')
            ->where('category_id', $categoryId)
            ->where('is_published', 1)
            ->orderBy('title')
            ->get()
            ->toArray();
    }

    public function getArticle(int $id): ?object {
        return Capsule::table('mod_{support}_kb_articles')
            ->where('id', $id)
            ->first();
    }

    public function getArticleBySlug(string $slug): ?object {
        return Capsule::table('mod_{support}_kb_articles')
            ->where('slug', $slug)
            ->where('is_published', 1)
            ->first();
    }

    public function search(string $query): array {
        return Capsule::table('mod_{support}_kb_articles')
            ->where('is_published', 1)
            ->where(function($q) use ($query) {
                $q->where('title', 'LIKE', "%{$query}%")
                  ->orWhere('content', 'LIKE', "%{$query}%");
            })
            ->orderBy('views', 'desc')
            ->get()
            ->toArray();
    }

    public function incrementViews(int $id): void {
        Capsule::table('mod_{support}_kb_articles')
            ->where('id', $id)
            ->increment('views');
    }

    public function rateArticle(int $id, bool $helpful): void {
        $column = $helpful ? 'helpful_yes' : 'helpful_no';
        Capsule::table('mod_{support}_kb_articles')
            ->where('id', $id)
            ->increment($column);
    }

    public function getRelatedArticles(int $articleId, int $limit = 5): array {
        $article = $this->getArticle($articleId);
        if (!$article) {
            return [];
        }

        return Capsule::table('mod_{support}_kb_articles')
            ->where('id', '!=', $articleId)
            ->where('is_published', 1)
            ->where(function($q) use ($article) {
                $q->where('category_id', $article->category_id)
                  ->orWhere('title', 'LIKE', "%" . explode(' ', $article->title)[0] . "%");
            })
            ->limit($limit)
            ->get()
            ->toArray();
    }
}
```

## Canned Responses Class

```php
<?php
namespace WHMCS\Module\Addon\{Support};

use WHMCS\Database\Capsule;

class CannedResponses {

    public function getAll(): array {
        return Capsule::table('mod_{support}_canned_responses')
            ->orderBy('title')
            ->get()
            ->toArray();
    }

    public function getByCategory(string $category): array {
        return Capsule::table('mod_{support}_canned_responses')
            ->where('category', $category)
            ->orderBy('title')
            ->get()
            ->toArray();
    }

    public function getByShortcode(string $shortcode): ?object {
        return Capsule::table('mod_{support}_canned_responses')
            ->where('shortcode', $shortcode)
            ->first();
    }

    public function getGlobal(): array {
        return Capsule::table('mod_{support}_canned_responses')
            ->where('is_global', 1)
            ->orderBy('title')
            ->get()
            ->toArray();
    }

    public function getByAdmin(int $adminId): array {
        return Capsule::table('mod_{support}_canned_responses')
            ->where('created_by', $adminId)
            ->orderBy('title')
            ->get()
            ->toArray();
    }

    public function incrementUsage(int $id): void {
        Capsule::table('mod_{support}_canned_responses')
            ->where('id', $id)
            ->increment('use_count');
    }

    public function getCategories(): array {
        return Capsule::table('mod_{support}_canned_responses')
            ->select('category')
            ->whereNotNull('category')
            ->where('category', '!=', '')
            ->groupBy('category')
            ->orderBy('category')
            ->get()
            ->toArray();
    }

    public function parseShortcodes(string $content, array $vars = []): string {
        $shortcodes = [
            '{client_name}' => $vars['client_name'] ?? '',
            '{ticket_id}' => $vars['ticket_id'] ?? '',
            '{ticket_subject}' => $vars['ticket_subject'] ?? '',
            '{admin_name}' => $_SESSION['adminname'] ?? '',
            '{date}' => date('Y-m-d'),
            '{company_name}' => Capsule::table('tblconfiguration')
                ->where('setting', 'CompanyName')
                ->value('value') ?? '',
        ];

        return str_replace(array_keys($shortcodes), array_values($shortcodes), $content);
    }
}
```

## SLA Policy Class

```php
<?php
namespace WHMCS\Module\Addon\{Support};

use WHMCS\Database\Capsule;

class SlaPolicyManager {

    public function getAll(): array {
        return Capsule::table('mod_{support}_sla_policies')
            ->orderBy('first_response_hours')
            ->get()
            ->toArray();
    }

    public function getActive(): array {
        return Capsule::table('mod_{support}_sla_policies')
            ->where('is_active', 1)
            ->orderBy('first_response_hours')
            ->get()
            ->toArray();
    }

    public function getByPriority(string $priority): ?object {
        return Capsule::table('mod_{support}_sla_policies')
            ->where('priority', $priority)
            ->where('is_active', 1)
            ->first();
    }

    public function calculateDeadline(int $ticketId, string $priority): ?array {
        $policy = $this->getByPriority($priority);

        if (!$policy) {
            return null;
        }

        $ticket = Capsule::table('tbltickets')
            ->where('id', $ticketId)
            ->first();

        if (!$ticket) {
            return null;
        }

        $createdAt = strtotime($ticket->created);

        return [
            'first_response_deadline' => date('Y-m-d H:i:s', $createdAt + ($policy->first_response_hours * 3600)),
            'resolution_deadline' => date('Y-m-d H:i:s', $createdAt + ($policy->resolution_hours * 3600)),
            'policy_name' => $policy->name,
            'first_response_hours' => $policy->first_response_hours,
            'resolution_hours' => $policy->resolution_hours,
        ];
    }

    public function isFirstResponseOverdue(int $ticketId): bool {
        $ticket = Capsule::table('tbltickets')
            ->where('id', $ticketId)
            ->first();

        if (!$ticket || $ticket->status === 'Closed') {
            return false;
        }

        $sla = $this->calculateDeadline($ticketId, $ticket->priority);
        if (!$sla) {
            return false;
        }

        $hasReply = Capsule::table('tblticketreplies')
            ->where('tid', $ticketId)
            ->where('admin', '!=', '0')
            ->exists();

        if (!$hasReply) {
            return time() > strtotime($sla['first_response_deadline']);
        }

        return false;
    }

    public function isResolutionOverdue(int $ticketId): bool {
        $ticket = Capsule::table('tbltickets')
            ->where('id', $ticketId)
            ->first();

        if (!$ticket || $ticket->status === 'Closed') {
            return false;
        }

        $sla = $this->calculateDeadline($ticketId, $ticket->priority);
        if (!$sla) {
            return false;
        }

        return time() > strtotime($sla['resolution_deadline']);
    }
}
```

## Hooks Integration

```php
<?php
/**
 * WHMCS Support Module Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Assign SLA to new tickets
add_hook('TicketOpen', 1, function(array $vars) {
    $ticketId = $vars['ticketid'];
    $priority = $vars['priority'] ?? 'Medium';

    $policyManager = new \WHMCS\Module\Addon\{Support}\SlaPolicyManager();
    $sla = $policyManager->calculateDeadline($ticketId, $priority);

    if ($sla) {
        Capsule::table('tbltickets')
            ->where('id', $ticketId)
            ->update([
                'flag' => $sla['first_response_deadline'],
            ]);

        logActivity("{Support}: Ticket #{$ticketId} SLA - First response due: {$sla['first_response_deadline']}");
    }
});

// Request feedback on ticket close
add_hook('TicketClose', 1, function(array $vars) {
    $ticketId = $vars['ticketid'];

    // Send feedback request via email template
    $ticket = Capsule::table('tbltickets')->where('id', $ticketId)->first();

    if ($ticket) {
        sendEmail(
            $ticket->userid,
            'SupportTicketFeedback',
            [
                'ticket_id' => $ticketId,
                'ticket_tid' => $ticket->tid,
                'ticket_subject' => $ticket->subject,
            ]
        );
    }
});

// Log ticket stats for SLA reporting
add_hook('TicketReply', 1, function(array $vars) {
    $ticketId = $vars['ticketid'];

    $policyManager = new \WHMCS\Module\Addon\{Support}\SlaPolicyManager();
    if ($policyManager->isFirstResponseOverdue($ticketId)) {
        logActivity("{Support}: First response SLA breached for ticket #{$ticketId}");
    }
});

// Daily SLA check
add_hook('DailyCronJob', 1, function(array $vars) {
    $overdueFirstResponse = Capsule::table('tbltickets')
        ->whereIn('status', ['Open', 'Awaiting Reply'])
        ->get();

    foreach ($overdueFirstResponse as $ticket) {
        $policyManager = new \WHMCS\Module\Addon\{Support}\SlaPolicyManager();

        if ($policyManager->isFirstResponseOverdue($ticket->id)) {
            // Alert admins about overdue tickets
            $admins = Capsule::table('tbladmins')->where('disabled', 0)->get();

            foreach ($admins as $admin) {
                sendEmail(
                    $admin->email,
                    'SlaOverdueAlert',
                    [
                        'ticket_id' => $ticket->id,
                        'ticket_subject' => $ticket->subject,
                        'overdue_type' => 'First Response',
                    ]
                );
            }
        }
    }
});
```

## Database Schema

### mod_{support}_kb_articles
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| title | VARCHAR(255) | Article title |
| content | TEXT | Article content |
| slug | VARCHAR(255) | URL-friendly slug |
| category_id | INT | Category FK |
| is_published | BOOLEAN | Published status |
| views | INT | View counter |
| helpful_yes | INT | Helpful votes |
| helpful_no | INT | Not helpful votes |
| meta_title | VARCHAR(255) | SEO title |
| meta_description | TEXT | SEO description |

### mod_{support}_sla_policies
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| name | VARCHAR(100) | Policy name |
| priority | VARCHAR(20) | Priority level |
| first_response_hours | INT | Hours for first response |
| resolution_hours | INT | Hours for resolution |
| is_active | BOOLEAN | Active status |

## Checklist

```
Pre-Dev:
□ Define support features needed
□ Plan knowledge base structure
□ Design canned responses categories
□ Plan SLA policies
□ Design feedback collection

Development:
□ Implement config() with all settings
□ Implement activate() → Create all tables
□ Implement deactivate() → Drop tables
□ Implement KnowledgeBase class
□ Implement CannedResponses class
□ Implement SlaPolicyManager class
□ Create admin interface with tabs
□ Create client area template
□ Add ticket hooks for SLA
□ Implement search functionality
□ Add feedback request on close

Security:
□ Validate all inputs
□ Sanitize article content
□ Use check_token() for POST
□ Escape outputs

Testing:
□ Test knowledge base CRUD
□ Test article search
□ Test canned responses
□ Test SLA policy assignment
□ Test feedback collection
□ Verify ticket hooks
```
