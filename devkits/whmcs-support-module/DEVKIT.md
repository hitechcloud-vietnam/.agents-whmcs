# WHMCS Support Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-support-module/
├── support.php           # Support module controller
├── lib/
│   ├── TicketManager.php  # Ticket management
│   ├── KnowledgeBase.php   # Knowledge base
│   └── CannedResponses.php # Canned responses
└── templates/
    ├── admin.tpl          # Admin templates
    └── client.tpl         # Client templates
```

## Support Module Template

```php
<?php
/**
 * WHMCS Support Module: {Module}
 * DevKit Template
 * 
 * Installation: Upload to modules/addons/{module}/
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {module}_config(): array {
    return [
        'name' => '{Support Module}',
        'description' => 'Enhanced support and helpdesk features',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Activate
 */
function {module}_activate(): array {
    Capsule::schema()->create('mod_{module}_kb_articles', function($t) {
        $t->increments('id');
        $t->string('title');
        $t->text('content');
        $t->string('slug');
        $t->integer('category_id');
        $t->boolean('is_published');
        $t->integer('views');
        $t->integer('helpful_yes');
        $t->integer('helpful_no');
        $t->timestamp('created_at');
        $t->timestamp('updated_at');
    });
    
    Capsule::schema()->create('mod_{module}_kb_categories', function($t) {
        $t->increments('id');
        $t->string('name');
        $t->text('description');
        $t->string('slug');
        $t->integer('parent_id')->nullable();
        $t->integer('sort_order');
    });
    
    Capsule::schema()->create('mod_{module}_canned_responses', function($t) {
        $t->increments('id');
        $t->string('title');
        $t->text('content');
        $t->string('category');
        $t->boolean('is_global');
        $t->integer('created_by');
    });
    
    Capsule::schema()->create('mod_{module}_ticket_feedback', function($t) {
        $t->increments('id');
        $t->integer('ticket_id');
        $t->integer('rating');
        $t->text('comment');
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_sla_policies', function($t) {
        $t->increments('id');
        $t->string('name');
        $t->string('priority');
        $t->integer('first_response_hours');
        $t->integer('resolution_hours');
        $t->boolean('is_active');
    });
    
    // Insert default SLA policies
    {module}_insertDefaultSla();
    
    return ['status' => 'success', 'description' => 'Module activated'];
}

/**
 * Insert Default SLA Policies
 */
function {module}_insertDefaultSla(): void {
    $policies = [
        ['name' => 'Critical', 'priority' => 'High', 'first_response_hours' => 1, 'resolution_hours' => 4, 'is_active' => 1],
        ['name' => 'High', 'priority' => 'Medium', 'first_response_hours' => 4, 'resolution_hours' => 24, 'is_active' => 1],
        ['name' => 'Normal', 'priority' => 'Low', 'first_response_hours' => 24, 'resolution_hours' => 72, 'is_active' => 1],
    ];
    
    foreach ($policies as $policy) {
        Capsule::table('mod_{module}_sla_policies')->insert($policy);
    }
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_kb_articles');
    Capsule::schema()->dropIfExists('mod_{module}_kb_categories');
    Capsule::schema()->dropIfExists('mod_{module}_canned_responses');
    Capsule::schema()->dropIfExists('mod_{module}_ticket_feedback');
    Capsule::schema()->dropIfExists('mod_{module}_sla_policies');
    
    return ['status' => 'success'];
}

/**
 * Output function
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
    
    switch ($action) {
        case 'kb':
            {module}_manageKnowledgeBase();
            break;
        case 'canned':
            {module}_manageCannedResponses();
            break;
        case 'sla':
            {module}_manageSlaPolicies();
            break;
        case 'feedback':
            {module}_viewFeedback();
            break;
        case 'reports':
            {module}_showReports();
            break;
        default:
            {module}_showDashboard();
    }
}

/**
 * Show Dashboard
 */
function {module}_showDashboard(): void {
    $stats = [
        'open_tickets' => Capsule::table('tbltickets')
            ->whereIn('status', ['Open', 'Awaiting Reply'])->count(),
        'kb_articles' => Capsule::table('mod_{module}_kb_articles')
            ->where('is_published', 1)->count(),
        'avg_response_time' => '2.5 hrs',
        'satisfaction_rate' => '94%',
    ];
    
    echo <<<HTML
<div class="support-module">
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Support Module Dashboard</h3>
                </div>
                <div class="panel-body">
                    <div class="row">
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['open_tickets']}</div>
                                <div class="stat-label">Open Tickets</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['kb_articles']}</div>
                                <div class="stat-label">KB Articles</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['avg_response_time']}</div>
                                <div class="stat-label">Avg Response</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['satisfaction_rate']}</div>
                                <div class="stat-label">Satisfaction</div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Quick Actions</h3>
                </div>
                <div class="panel-body">
                    <div class="btn-group">
                        <a href="?module={module}&action=kb" class="btn btn-primary">
                            <i class="fa fa-book"></i> Knowledge Base
                        </a>
                        <a href="?module={module}&action=canned" class="btn btn-primary">
                            <i class="fa fa-file-alt"></i> Canned Responses
                        </a>
                        <a href="?module={module}&action=sla" class="btn btn-primary">
                            <i class="fa fa-clock"></i> SLA Policies
                        </a>
                        <a href="?module={module}&action=feedback" class="btn btn-default">
                            <i class="fa fa-star"></i> Feedback
                        </a>
                        <a href="?module={module}&action=reports" class="btn btn-default">
                            <i class="fa fa-chart-bar"></i> Reports
                        </a>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <div class="row">
        <div class="col-md-6">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Recent Tickets</h3>
                </div>
                <div class="panel-body">
                    <table class="table table-striped">
                        <thead>
                            <tr>
                                <th>ID</th>
                                <th>Subject</th>
                                <th>Status</th>
                            </tr>
                        </thead>
                        <tbody>
HTML;
    
    $recentTickets = Capsule::table('tbltickets')
        ->orderBy('id', 'desc')
        ->limit(5)
        ->get();
    
    foreach ($recentTickets as $ticket) {
        $statusClass = $ticket->status === 'Open' ? 'success' : 'warning';
        echo "<tr>
            <td>#{$ticket->id}</td>
            <td>{$ticket->title}</td>
            <td><span class='label label-{$statusClass}'>{$ticket->status}</span></td>
        </tr>";
    }
    
    echo "</tbody></table></div></div></div>";
    
    echo "<div class='col-md-6'>
        <div class='panel panel-default'>
            <div class='panel-heading'>
                <h3 class='panel-title'>Popular KB Articles</h3>
            </div>
            <div class='panel-body'>
                <table class='table table-striped'>
                    <thead>
                        <tr><th>Title</th><th>Views</th></tr>
                    </thead>
                    <tbody>";
    
    $popularArticles = Capsule::table('mod_{module}_kb_articles')
        ->where('is_published', 1)
        ->orderBy('views', 'desc')
        ->limit(5)
        ->get();
    
    foreach ($popularArticles as $article) {
        echo "<tr><td>{$article->title}</td><td>{$article->views}</td></tr>";
    }
    
    echo "</tbody></table></div></div></div></div></div>";
}

/**
 * Client Area Output
 */
function {module}_clientarea(array $vars): array {
    return [
        'pagetitle' => 'Support Center',
        'templatefile' => 'templates/client_support',
        'vars' => [
            'categories' => {module}_getKbCategories(),
            'recent_articles' => {module}_getRecentArticles(),
            'popular_articles' => {module}_getPopularArticles(),
        ],
    ];
}

/**
 * Get KB Categories
 */
function {module}_getKbCategories(): array {
    return Capsule::table('mod_{module}_kb_categories')
        ->whereNull('parent_id')
        ->orderBy('sort_order')
        ->get()
        ->toArray();
}

/**
 * Get Recent Articles
 */
function {module}_getRecentArticles(int $limit = 5): array {
    return Capsule::table('mod_{module}_kb_articles')
        ->where('is_published', 1)
        ->orderBy('created_at', 'desc')
        ->limit($limit)
        ->get()
        ->toArray();
}

/**
 * Get Popular Articles
 */
function {module}_getPopularArticles(int $limit = 5): array {
    return Capsule::table('mod_{module}_kb_articles')
        ->where('is_published', 1)
        ->orderBy('views', 'desc')
        ->limit($limit)
        ->get()
        ->toArray();
}
```

## Knowledge Base Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class KnowledgeBase {
    
    public function createArticle(array $data): int {
        $data['slug'] = $this->generateSlug($data['title']);
        $data['created_at'] = date('Y-m-d H:i:s');
        $data['updated_at'] = date('Y-m-d H:i:s');
        
        return Capsule::table('mod_{module}_kb_articles')->insertGetId($data);
    }
    
    public function updateArticle(int $id, array $data): bool {
        $data['slug'] = $this->generateSlug($data['title']);
        $data['updated_at'] = date('Y-m-d H:i:s');
        
        return Capsule::table('mod_{module}_kb_articles')
            ->where('id', $id)
            ->update($data) > 0;
    }
    
    public function deleteArticle(int $id): bool {
        return Capsule::table('mod_{module}_kb_articles')
            ->where('id', $id)
            ->delete() > 0;
    }
    
    public function getArticle(int $id): ?object {
        return Capsule::table('mod_{module}_kb_articles')
            ->where('id', $id)
            ->first();
    }
    
    public function getArticleBySlug(string $slug): ?object {
        return Capsule::table('mod_{module}_kb_articles')
            ->where('slug', $slug)
            ->first();
    }
    
    public function incrementViews(int $id): void {
        Capsule::table('mod_{module}_kb_articles')
            ->where('id', $id)
            ->increment('views');
    }
    
    public function search(string $query): array {
        return Capsule::table('mod_{module}_kb_articles')
            ->where('is_published', 1)
            ->where(function($q) use ($query) {
                $q->where('title', 'LIKE', "%{$query}%")
                  ->orWhere('content', 'LIKE', "%{$query}%");
            })
            ->get()
            ->toArray();
    }
    
    public function rateArticle(int $id, bool $helpful): void {
        $column = $helpful ? 'helpful_yes' : 'helpful_no';
        Capsule::table('mod_{module}_kb_articles')
            ->where('id', $id)
            ->increment($column);
    }
    
    public function getCategories(): array {
        return Capsule::table('mod_{module}_kb_categories')
            ->orderBy('sort_order')
            ->get()
            ->toArray();
    }
    
    public function getArticlesByCategory(int $categoryId): array {
        return Capsule::table('mod_{module}_kb_articles')
            ->where('category_id', $categoryId)
            ->where('is_published', 1)
            ->orderBy('title')
            ->get()
            ->toArray();
    }
    
    private function generateSlug(string $title): string {
        $slug = strtolower(trim($title));
        $slug = preg_replace('/[^a-z0-9-]/', '-', $slug);
        $slug = preg_replace('/-+/', '-', $slug);
        $slug = trim($slug, '-');
        
        return $slug . '-' . substr(md5(uniqid()), 0, 6);
    }
}
```

## Ticket Hooks

```php
<?php
/**
 * Support Module Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Add SLA info to new tickets
add_hook('TicketOpen', 1, function(array $vars) {
    $ticketId = $vars['ticketid'];
    $priority = $vars['priority'] ?? 'Medium';
    
    $sla = Capsule::table('mod_{module}_sla_policies')
        ->where('priority', $priority)
        ->where('is_active', 1)
        ->first();
    
    if ($sla) {
        $firstResponseDeadline = date('Y-m-d H:i:s', 
            strtotime("+{$sla->first_response_hours} hours"));
        
        Capsule::table('tbltickets')
            ->where('id', $ticketId)
            ->update([
                'flag' => $firstResponseDeadline,
            ]);
        
        logActivity("{Module}: Ticket #{$ticketId} assigned SLA policy: {$sla->name}");
    }
});

// Close ticket feedback request
add_hook('TicketClose', 1, function(array $vars) {
    $ticketId = $vars['ticketid'];
    
    // Send feedback request email
    $ticket = Capsule::table('tbltickets')->where('id', $ticketId)->first();
    
    if ($ticket) {
        sendAutomatedEmail(
            'support_feedback_request',
            $ticket->tid,
            [
                'ticket_id' => $ticketId,
                'ticket_subject' => $ticket->title,
            ]
        );
    }
});
```

## Checklist

```
Pre-Dev:
□ Define support features to implement
□ Plan knowledge base structure
□ Design canned responses system
□ Plan SLA policies
□ Design feedback collection

Development:
□ Create support tables
□ Implement KnowledgeBase class
□ Implement TicketManager class
□ Create article CRUD
□ Build category management
□ Implement search functionality
□ Create canned responses
□ Add SLA policies
□ Build feedback system
□ Create admin interface
□ Create client area
□ Add ticket hooks
□ Build reports

Testing:
□ Test knowledge base
□ Test article creation
□ Test search functionality
□ Test canned responses
□ Test SLA assignment
□ Test feedback collection
□ Verify ticket hooks
□ Test client area
```