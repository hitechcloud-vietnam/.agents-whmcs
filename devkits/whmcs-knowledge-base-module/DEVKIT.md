# WHMCS Knowledge Base Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## Purpose
Create a comprehensive knowledge base module for WHMCS that allows you to create, organize, and display helpful articles and documentation.

## Module Type
Addon Module

## Use Case
- Self-service documentation portal
- Product guides and tutorials
- FAQ sections
- Searchable knowledge base
- Article ratings and feedback

## DevKit Structure

```
devkits/whmcs-knowledge-base-module/
├── knowledgebase.php     # Main addon module
├── lib/
│   ├── ArticleManager.php  # Article management
│   ├── CategoryManager.php # Category management
│   └── SearchEngine.php   # Search functionality
├── templates/
│   ├── admin.tpl          # Admin templates
│   └── client.tpl         # Client templates
├── hooks.php              # Hook integrations
└── DEVKIT.md            # This file
```

## Main Module Template

```php
<?php
/**
 * WHMCS Knowledge Base Module: {KnowledgeBase}
 * Knowledge Base Module Template
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {knowledgebase}_config(): array {
    return [
        'name' => '{Knowledge Base}',
        'description' => 'Self-service knowledge base and documentation',
        'version' => '1.0',
        'author' => '{Author Name}',

        'articles_per_page' => [
            'FriendlyName' => 'Articles Per Page',
            'Type' => 'text',
            'Size' => '5',
            'Default' => '10',
        ],
        'enable_ratings' => [
            'FriendlyName' => 'Enable Article Ratings',
            'Type' => 'yesno',
        ],
        'enable_comments' => [
            'FriendlyName' => 'Enable Comments',
            'Type' => 'yesno',
        ],
        'require_login' => [
            'FriendlyName' => 'Require Login to View',
            'Type' => 'yesno',
        ],
        'enable_search' => [
            'FriendlyName' => 'Enable Search',
            'Type' => 'yesno',
        ],
        'featured_articles' => [
            'FriendlyName' => 'Show Featured Articles',
            'Type' => 'yesno',
        ],
        'popular_count' => [
            'FriendlyName' => 'Popular Articles Count',
            'Type' => 'text',
            'Size' => '5',
            'Default' => '5',
        ],
        'enable_attachments' => [
            'FriendlyName' => 'Enable Attachments',
            'Type' => 'yesno',
        ],
    ];
}

function {knowledgebase}_activate(): array {
    try {
        // Categories table
        if (!Capsule::schema()->hasTable('mod_{knowledgebase}_categories')) {
            Capsule::schema()->create('mod_{knowledgebase}_categories', function($t) {
                $t->increments('id');
                $t->string('name', 150);
                $t->text('description')->nullable();
                $t->string('slug', 150)->unique();
                $t->integer('parent_id')->unsigned()->nullable();
                $t->integer('sort_order')->default(0);
                $t->boolean('is_active')->default(true);
                $t->boolean('show_in_menu')->default(true);
                $t->string('icon', 50)->nullable();
                $t->timestamps();

                $t->index('parent_id');
                $t->index('slug');
            });
        }

        // Articles table
        if (!Capsule::schema()->hasTable('mod_{knowledgebase}_articles')) {
            Capsule::schema()->create('mod_{knowledgebase}_articles', function($t) {
                $t->increments('id');
                $t->string('title', 255);
                $t->text('content');
                $t->string('slug', 255)->unique();
                $t->integer('category_id')->unsigned();
                $t->string('meta_title', 255)->nullable();
                $t->text('meta_description')->nullable();
                $t->string('tags', 500)->nullable();
                $t->boolean('is_published')->default(false);
                $t->boolean('is_featured')->default(false);
                $t->boolean('require_login')->default(false);
                $t->integer('views')->default(0);
                $t->decimal('rating', 3, 2)->default(0);
                $t->integer('rating_count')->default(0);
                $t->integer('helpful_yes')->default(0);
                $t->integer('helpful_no')->default(0);
                $t->string('author_name', 100)->nullable();
                $t->timestamp('published_at')->nullable();
                $t->timestamps();

                $t->index('category_id');
                $t->index('slug');
                $t->index('is_published');
                $t->index('is_featured');
            });
        }

        // Article attachments
        if (!Capsule::schema()->hasTable('mod_{knowledgebase}_attachments')) {
            Capsule::schema()->create('mod_{knowledgebase}_attachments', function($t) {
                $t->increments('id');
                $t->integer('article_id')->unsigned();
                $t->string('filename', 255);
                $t->string('filepath', 255);
                $t->bigInteger('filesize')->default(0);
                $t->string('mime_type', 100);
                $t->integer('downloads')->default(0);
                $t->timestamp('created_at');

                $t->index('article_id');
            });
        }

        // Article ratings
        if (!Capsule::schema()->hasTable('mod_{knowledgebase}_ratings')) {
            Capsule::schema()->create('mod_{knowledgebase}_ratings', function($t) {
                $t->increments('id');
                $t->integer('article_id')->unsigned();
                $t->integer('user_id')->unsigned()->nullable();
                $t->string('ip_address', 45)->nullable();
                $t->integer('rating')->unsigned();
                $t->timestamp('created_at');

                $t->unique(['article_id', 'user_id']);
                $t->index('article_id');
            });
        }

        // Article comments
        if (!Capsule::schema()->hasTable('mod_{knowledgebase}_comments')) {
            Capsule::schema()->create('mod_{knowledgebase}_comments', function($t) {
                $t->increments('id');
                $t->integer('article_id')->unsigned();
                $t->integer('parent_id')->unsigned()->nullable();
                $t->integer('user_id')->unsigned()->nullable();
                $t->string('author_name', 100);
                $t->string('author_email', 255)->nullable();
                $t->text('content');
                $t->boolean('is_approved')->default(false);
                $t->timestamp('created_at');

                $t->index('article_id');
                $t->index('parent_id');
            });
        }

        // Insert default categories
        $defaultCategories = [
            ['name' => 'Getting Started', 'slug' => 'getting-started', 'sort_order' => 1],
            ['name' => 'Account & Billing', 'slug' => 'account-billing', 'sort_order' => 2],
            ['name' => 'Products & Services', 'slug' => 'products-services', 'sort_order' => 3],
            ['name' => 'Technical Support', 'slug' => 'technical-support', 'sort_order' => 4],
            ['name' => 'FAQ', 'slug' => 'faq', 'sort_order' => 5],
        ];

        foreach ($defaultCategories as $category) {
            Capsule::table('mod_{knowledgebase}_categories')->insert($category);
        }

        return ['status' => 'success', 'description' => '{Knowledge Base} activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => 'Activation failed: ' . $e->getMessage()];
    }
}

function {knowledgebase}_deactivate(): array {
    try {
        Capsule::schema()->dropIfExists('mod_{knowledgebase}_categories');
        Capsule::schema()->dropIfExists('mod_{knowledgebase}_articles');
        Capsule::schema()->dropIfExists('mod_{knowledgebase}_attachments');
        Capsule::schema()->dropIfExists('mod_{knowledgebase}_ratings');
        Capsule::schema()->dropIfExists('mod_{knowledgebase}_comments');
        return ['status' => 'success'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => 'Deactivation failed'];
    }
}

function {knowledgebase}_output(array $vars): void {
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
        handleKbAction($_POST['action'] ?? '');
    }

    $tab = $_REQUEST['tab'] ?? 'articles';
    $action = $_REQUEST['action'] ?? 'list';

    echo '<div class="knowledgebase-module">';
    echo '<h1><i class="fa fa-book"></i> Knowledge Base</h1>';
    echo '<ul class="nav nav-tabs">';
    echo '<li class="' . ($tab === 'articles' ? 'active' : '') . '"><a href="?module={knowledgebase}&tab=articles">Articles</a></li>';
    echo '<li class="' . ($tab === 'categories' ? 'active' : '') . '"><a href="?module={knowledgebase}&tab=categories">Categories</a></li>';
    echo '<li class="' . ($tab === 'comments' ? 'active' : '') . '"><a href="?module={knowledgebase}&tab=comments">Comments</a></li>';
    echo '<li class="' . ($tab === 'settings' ? 'active' : '') . '"><a href="?module={knowledgebase}&tab=settings">Settings</a></li>';
    echo '</ul>';

    if ($tab === 'articles') {
        include __DIR__ . '/templates/admin/articles.tpl';
    } elseif ($tab === 'categories') {
        include __DIR__ . '/templates/admin/categories.tpl';
    } elseif ($tab === 'comments') {
        include __DIR__ . '/templates/admin/comments.tpl';
    } else {
        include __DIR__ . '/templates/admin/settings.tpl';
    }

    echo '</div>';
}

function {knowledgebase}_clientarea(array $vars): array {
    $action = $_REQUEST['action'] ?? 'home';
    $slug = $_REQUEST['slug'] ?? '';

    return [
        'pagetitle' => 'Knowledge Base',
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'action' => $action,
            'slug' => $slug,
            'categories' => getActiveCategories(),
            'featured_articles' => getFeaturedArticles(),
            'popular_articles' => getPopularArticles(),
        ],
    ];
}

function handleKbAction(string $action): void {
    switch ($action) {
        case 'save_article':
            saveArticle();
            break;
        case 'delete_article':
            deleteArticle((int)($_GET['id'] ?? 0));
            break;
        case 'save_category':
            saveCategory();
            break;
        case 'delete_category':
            deleteCategory((int)($_GET['id'] ?? 0));
            break;
        case 'approve_comment':
            approveComment((int)($_GET['id'] ?? 0));
            break;
        case 'delete_comment':
            deleteComment((int)($_GET['id'] ?? 0));
            break;
    }

    header('Location: ?module={knowledgebase}&tab=' . ($_POST['redirect_tab'] ?? 'articles'));
    exit;
}

function getActiveCategories(): array {
    return Capsule::table('mod_{knowledgebase}_categories')
        ->where('is_active', 1)
        ->orderBy('sort_order')
        ->get()
        ->toArray();
}

function getFeaturedArticles(): array {
    return Capsule::table('mod_{knowledgebase}_articles')
        ->where('is_published', 1)
        ->where('is_featured', 1)
        ->orderBy('published_at', 'desc')
        ->limit(5)
        ->get()
        ->toArray();
}

function getPopularArticles(int $limit = 5): array {
    return Capsule::table('mod_{knowledgebase}_articles')
        ->where('is_published', 1)
        ->orderBy('views', 'desc')
        ->limit($limit)
        ->get()
        ->toArray();
}

function getRecentArticles(int $limit = 10): array {
    return Capsule::table('mod_{knowledgebase}_articles')
        ->where('is_published', 1)
        ->orderBy('published_at', 'desc')
        ->limit($limit)
        ->get()
        ->toArray();
}

function saveArticle(): void {
    $id = (int)($_POST['id'] ?? 0);
    $data = [
        'title' => $_POST['title'] ?? '',
        'slug' => generateSlug($_POST['title'] ?? ''),
        'category_id' => (int)($_POST['category_id'] ?? 0),
        'content' => $_POST['content'] ?? '',
        'meta_title' => $_POST['meta_title'] ?? null,
        'meta_description' => $_POST['meta_description'] ?? null,
        'tags' => $_POST['tags'] ?? null,
        'is_published' => isset($_POST['is_published']) ? 1 : 0,
        'is_featured' => isset($_POST['is_featured']) ? 1 : 0,
        'require_login' => isset($_POST['require_login']) ? 1 : 0,
        'author_name' => $_POST['author_name'] ?? null,
        'published_at' => isset($_POST['is_published']) ? date('Y-m-d H:i:s') : null,
    ];

    if ($id > 0) {
        Capsule::table('mod_{knowledgebase}_articles')
            ->where('id', $id)
            ->update($data);
    } else {
        Capsule::table('mod_{knowledgebase}_articles')->insert($data);
    }
}

function deleteArticle(int $id): void {
    Capsule::table('mod_{knowledgebase}_articles')->where('id', $id)->delete();
    Capsule::table('mod_{knowledgebase}_attachments')->where('article_id', $id)->delete();
    Capsule::table('mod_{knowledgebase}_ratings')->where('article_id', $id)->delete();
    Capsule::table('mod_{knowledgebase}_comments')->where('article_id', $id)->delete();
}

function saveCategory(): void {
    $id = (int)($_POST['id'] ?? 0);
    $data = [
        'name' => $_POST['name'] ?? '',
        'slug' => generateSlug($_POST['name'] ?? ''),
        'description' => $_POST['description'] ?? null,
        'parent_id' => !empty($_POST['parent_id']) ? (int)$_POST['parent_id'] : null,
        'sort_order' => (int)($_POST['sort_order'] ?? 0),
        'is_active' => isset($_POST['is_active']) ? 1 : 0,
        'show_in_menu' => isset($_POST['show_in_menu']) ? 1 : 0,
        'icon' => $_POST['icon'] ?? null,
    ];

    if ($id > 0) {
        Capsule::table('mod_{knowledgebase}_categories')
            ->where('id', $id)
            ->update($data);
    } else {
        Capsule::table('mod_{knowledgebase}_categories')->insert($data);
    }
}

function deleteCategory(int $id): void {
    // Move articles to uncategorized
    Capsule::table('mod_{knowledgebase}_articles')
        ->where('category_id', $id)
        ->update(['category_id' => 0]);

    Capsule::table('mod_{knowledgebase}_categories')->where('id', $id)->delete();
}

function approveComment(int $id): void {
    Capsule::table('mod_{knowledgebase}_comments')
        ->where('id', $id)
        ->update(['is_approved' => 1]);
}

function deleteComment(int $id): void {
    Capsule::table('mod_{knowledgebase}_comments')->where('id', $id)->delete();
}

function generateSlug(string $title): string {
    $slug = strtolower(trim($title));
    $slug = preg_replace('/[^a-z0-9-]/', '-', $slug);
    $slug = preg_replace('/-+/', '-', $slug);

    return trim($slug, '-') . '-' . substr(md5(uniqid()), 0, 6);
}
```

## Article Manager Class

```php
<?php
namespace WHMCS\Module\Addon\{KnowledgeBase};

use WHMCS\Database\Capsule;

class ArticleManager {

    public function getArticle(int $id): ?object {
        return Capsule::table('mod_{knowledgebase}_articles')
            ->where('id', $id)
            ->first();
    }

    public function getArticleBySlug(string $slug): ?object {
        return Capsule::table('mod_{knowledgebase}_articles')
            ->where('slug', $slug)
            ->where('is_published', 1)
            ->first();
    }

    public function getArticlesByCategory(int $categoryId, int $page = 1, int $perPage = 10): array {
        $offset = ($page - 1) * $perPage;

        $articles = Capsule::table('mod_{knowledgebase}_articles')
            ->where('category_id', $categoryId)
            ->where('is_published', 1)
            ->orderBy('title')
            ->limit($perPage)
            ->offset($offset)
            ->get();

        $total = Capsule::table('mod_{knowledgebase}_articles')
            ->where('category_id', $categoryId)
            ->where('is_published', 1)
            ->count();

        return [
            'articles' => $articles->toArray(),
            'total' => $total,
            'page' => $page,
            'per_page' => $perPage,
            'total_pages' => ceil($total / $perPage),
        ];
    }

    public function incrementViews(int $articleId): void {
        Capsule::table('mod_{knowledgebase}_articles')
            ->where('id', $articleId)
            ->increment('views');
    }

    public function rateArticle(int $articleId, int $userId, int $rating): array {
        // Check for existing rating
        $existing = Capsule::table('mod_{knowledgebase}_ratings')
            ->where('article_id', $articleId)
            ->where('user_id', $userId)
            ->first();

        if ($existing) {
            // Update existing rating
            Capsule::table('mod_{knowledgebase}_ratings')
                ->where('id', $existing->id)
                ->update(['rating' => $rating]);

            return ['success' => true, 'message' => 'Rating updated'];
        }

        // Create new rating
        Capsule::table('mod_{knowledgebase}_ratings')->insert([
            'article_id' => $articleId,
            'user_id' => $userId,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
            'rating' => $rating,
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        // Update article rating
        $this->recalculateRating($articleId);

        return ['success' => true, 'message' => 'Rating saved'];
    }

    public function recalculateRating(int $articleId): void {
        $ratings = Capsule::table('mod_{knowledgebase}_ratings')
            ->where('article_id', $articleId)
            ->get();

        $total = $ratings->count();
        $sum = $ratings->sum('rating');
        $avg = $total > 0 ? round($sum / $total, 2) : 0;

        Capsule::table('mod_{knowledgebase}_articles')
            ->where('id', $articleId)
            ->update([
                'rating' => $avg,
                'rating_count' => $total,
            ]);
    }

    public function markHelpful(int $articleId, bool $helpful): void {
        $column = $helpful ? 'helpful_yes' : 'helpful_no';
        Capsule::table('mod_{knowledgebase}_articles')
            ->where('id', $articleId)
            ->increment($column);
    }

    public function getRelatedArticles(int $articleId, int $limit = 5): array {
        $article = $this->getArticle($articleId);

        if (!$article || empty($article->tags)) {
            return [];
        }

        $tags = explode(',', $article->tags);
        $firstTag = trim($tags[0]);

        return Capsule::table('mod_{knowledgebase}_articles')
            ->where('id', '!=', $articleId)
            ->where('is_published', 1)
            ->where('tags', 'LIKE', "%{$firstTag}%")
            ->orderBy('views', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    public function getBreadcrumbs(int $articleId): array {
        $article = $this->getArticle($articleId);

        if (!$article) {
            return [];
        }

        $breadcrumbs = [];

        // Get category hierarchy
        $categoryId = $article->category_id;
        while ($categoryId) {
            $category = Capsule::table('mod_{knowledgebase}_categories')
                ->where('id', $categoryId)
                ->first();

            if ($category) {
                $breadcrumbs[] = $category;
                $categoryId = $category->parent_id;
            } else {
                break;
            }
        }

        return array_reverse($breadcrumbs);
    }
}
```

## Search Engine Class

```php
<?php
namespace WHMCS\Module\Addon\{KnowledgeBase};

use WHMCS\Database\Capsule;

class SearchEngine {

    public function search(string $query, int $page = 1, int $perPage = 10): array {
        $offset = ($page - 1) * $perPage;

        $searchTerms = $this->parseSearchTerms($query);

        $articles = Capsule::table('mod_{knowledgebase}_articles')
            ->where('is_published', 1)
            ->where(function($q) use ($searchTerms) {
                foreach ($searchTerms as $term) {
                    $q->where(function($sub) use ($term) {
                        $sub->where('title', 'LIKE', "%{$term}%")
                            ->orWhere('content', 'LIKE', "%{$term}%")
                            ->orWhere('tags', 'LIKE', "%{$term}%");
                    });
                }
            })
            ->selectRaw('*, MATCH(title, content) AGAINST(? IN NATURAL LANGUAGE MODE) as relevance', [$query])
            ->orderBy('relevance', 'desc')
            ->limit($perPage)
            ->offset($offset)
            ->get();

        $total = Capsule::table('mod_{knowledgebase}_articles')
            ->where('is_published', 1)
            ->where(function($q) use ($searchTerms) {
                foreach ($searchTerms as $term) {
                    $q->where(function($sub) use ($term) {
                        $sub->where('title', 'LIKE', "%{$term}%")
                            ->orWhere('content', 'LIKE', "%{$term}%");
                    });
                }
            })
            ->count();

        return [
            'articles' => $articles->toArray(),
            'total' => $total,
            'query' => $query,
            'page' => $page,
            'per_page' => $perPage,
            'total_pages' => ceil($total / $perPage),
        ];
    }

    public function suggest(string $query, int $limit = 5): array {
        return Capsule::table('mod_{knowledgebase}_articles')
            ->where('is_published', 1)
            ->where('title', 'LIKE', "%{$query}%")
            ->select('id', 'title', 'slug')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    public function getPopularSearches(int $limit = 10): array {
        // Get most viewed articles as "popular"
        return Capsule::table('mod_{knowledgebase}_articles')
            ->where('is_published', 1)
            ->orderBy('views', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    public function getSearchSuggestions(string $query): array {
        // Get matching titles
        $titles = Capsule::table('mod_{knowledgebase}_articles')
            ->where('is_published', 1)
            ->where('title', 'LIKE', "{$query}%")
            ->select('title')
            ->limit(5)
            ->get()
            ->pluck('title')
            ->toArray();

        // Get matching tags
        $tagArticles = Capsule::table('mod_{knowledgebase}_articles')
            ->where('is_published', 1)
            ->where('tags', 'LIKE', "%{$query}%")
            ->select('tags')
            ->limit(5)
            ->get();

        $tags = [];
        foreach ($tagArticles as $article) {
            $articleTags = explode(',', $article->tags);
            foreach ($articleTags as $tag) {
                $tag = trim($tag);
                if (stripos($tag, $query) !== false) {
                    $tags[] = $tag;
                }
            }
        }

        return [
            'titles' => array_unique($titles),
            'tags' => array_unique($tags),
        ];
    }

    private function parseSearchTerms(string $query): array {
        // Remove special characters and split into terms
        $query = preg_replace('/[^\p{L}\p{N}\s]/u', '', $query);
        $terms = array_filter(explode(' ', $query));

        return array_map('trim', $terms);
    }
}
```

## Hooks Integration

```php
<?php
/**
 * WHMCS Knowledge Base Module Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Suggest KB articles in ticket form
add_hook('TicketOpen', 1, function(array $vars) {
    $subject = $vars['subject'] ?? '';

    if (strlen($subject) < 5) {
        return;
    }

    $searchEngine = new \WHMCS\Module\Addon\{KnowledgeBase}\SearchEngine();
    $suggestions = $searchEngine->suggest($subject, 3);

    if (!empty($suggestions)) {
        return [
            'kb_suggestions' => $suggestions,
        ];
    }
});

// Notify admins of new comments
add_hook('AfterCommentSubmitted', 1, function(array $vars) {
    $articleId = $vars['article_id'];
    $article = Capsule::table('mod_{knowledgebase}_articles')
        ->where('id', $articleId)
        ->first();

    if ($article) {
        $admins = Capsule::table('tbladmins')
            ->where('disabled', 0)
            ->get();

        foreach ($admins as $admin) {
            sendEmail(
                $admin->email,
                'NewKbComment',
                [
                    'article_title' => $article->title,
                    'comment_author' => $vars['author_name'],
                    'comment_preview' => substr($vars['content'], 0, 100),
                ]
            );
        }
    }
});

// Update article on product update
add_hook('AfterServiceChangePackage', 1, function(array $vars) {
    // Could link to relevant KB articles for the new product
});
```

## Database Schema

### mod_{knowledgebase}_articles
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| title | VARCHAR(255) | Article title |
| content | TEXT | Article body |
| slug | VARCHAR(255) | URL slug |
| category_id | INT | Category FK |
| meta_title | VARCHAR(255) | SEO title |
| meta_description | TEXT | SEO description |
| tags | VARCHAR(500) | Comma-separated tags |
| is_published | BOOLEAN | Published status |
| is_featured | BOOLEAN | Featured status |
| views | INT | View count |
| rating | DECIMAL(3,2) | Average rating |
| rating_count | INT | Number of ratings |
| helpful_yes | INT | Helpful votes |
| helpful_no | INT | Not helpful votes |
| published_at | TIMESTAMP | Publication date |

### mod_{knowledgebase}_categories
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| name | VARCHAR(150) | Category name |
| description | TEXT | Description |
| slug | VARCHAR(150) | URL slug |
| parent_id | INT | Parent category |
| sort_order | INT | Display order |
| is_active | BOOLEAN | Active status |
| show_in_menu | BOOLEAN | Show in navigation |
| icon | VARCHAR(50) | Icon class |

## Checklist

```
Pre-Dev:
□ Plan category structure
□ Define article templates
□ Design search functionality
□ Plan rating system

Development:
□ Implement config() with all settings
□ Implement activate() → Create tables
□ Implement deactivate() → Drop tables
□ Create ArticleManager class
□ Create CategoryManager class
□ Create SearchEngine class
□ Implement article CRUD
□ Implement category management
□ Add full-text search
□ Create client area templates
□ Add article ratings
□ Add article comments
□ Create admin interface

Security:
□ Sanitize article content
□ Validate all inputs
□ Use check_token() for POST
□ Escape outputs

Testing:
□ Test article creation
□ Test category hierarchy
□ Test search functionality
□ Test article ratings
□ Verify pagination
```
