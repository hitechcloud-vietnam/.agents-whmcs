# WHMCS Reviews Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## Purpose
Create a product/service review module for WHMCS that allows customers to leave reviews and ratings.

## Module Type
Addon Module

## Use Case
- Customer reviews for products/services
- Star ratings system
- Review moderation
- Helpful votes on reviews
- Review analytics

## DevKit Structure

```
devkits/whmcs-reviews-module/
├── reviews.php            # Main addon module
├── lib/
│   ├── ReviewManager.php   # Review management
│   └── RatingCalculator.php # Rating calculations
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
 * WHMCS Reviews Module: {Reviews}
 * Product/Service Review Module Template
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {reviews}_config(): array {
    return [
        'name' => '{Reviews Module}',
        'description' => 'Product and service review system',
        'version' => '1.0',
        'author' => '{Author Name}',

        'require_purchase' => [
            'FriendlyName' => 'Require Purchase',
            'Type' => 'yesno',
            'Description' => 'Only allow reviews from verified purchasers',
        ],
        'require_login' => [
            'FriendlyName' => 'Require Login',
            'Type' => 'yesno',
            'Description' => 'Require login to leave a review',
        ],
        'moderate_reviews' => [
            'FriendlyName' => 'Moderate Reviews',
            'Type' => 'yesno',
            'Description' => 'Require admin approval before publishing',
        ],
        'allow_anonymous' => [
            'FriendlyName' => 'Allow Anonymous',
            'Type' => 'yesno',
            'Description' => 'Allow anonymous reviews (requires login)',
        ],
        'min_rating' => [
            'FriendlyName' => 'Minimum Rating',
            'Type' => 'dropdown',
            'Options' => '1,2,3,4,5',
            'Default' => '1',
            'Description' => 'Minimum star rating allowed',
        ],
        'max_reviews_per_day' => [
            'FriendlyName' => 'Max Reviews Per Day',
            'Type' => 'text',
            'Size' => '5',
            'Default' => '5',
        ],
        'enable_helpful_votes' => [
            'FriendlyName' => 'Enable Helpful Votes',
            'Type' => 'yesno',
        ],
        'display_avg_rating' => [
            'FriendlyName' => 'Display Average Rating',
            'Type' => 'yesno',
            'Description' => 'Show average rating on product pages',
        ],
    ];
}

function {reviews}_activate(): array {
    try {
        // Reviews table
        if (!Capsule::schema()->hasTable('mod_{reviews}_reviews')) {
            Capsule::schema()->create('mod_{reviews}_reviews', function($t) {
                $t->increments('id');
                $t->string('review_type', 20);
                $t->integer('rel_id')->unsigned();
                $t->integer('user_id')->unsigned()->nullable();
                $t->string('user_name', 100)->nullable();
                $t->string('user_email', 255)->nullable();
                $t->integer('rating')->unsigned();
                $t->string('title', 255);
                $t->text('content');
                $t->string('pros', 500)->nullable();
                $t->string('cons', 500)->nullable();
                $t->string('status', 20)->default('pending');
                $t->integer('helpful_yes')->default(0);
                $t->integer('helpful_no')->default(0);
                $t->string('ip_address', 45)->nullable();
                $t->timestamp('created_at');
                $t->timestamp('updated_at')->nullable();
                $t->timestamp('approved_at')->nullable();
                $t->integer('approved_by')->unsigned()->nullable();

                $t->index(['review_type', 'rel_id']);
                $t->index('user_id');
                $t->index('status');
            });
        }

        // Helpful votes table
        if (!Capsule::schema()->hasTable('mod_{reviews}_votes')) {
            Capsule::schema()->create('mod_{reviews}_votes', function($t) {
                $t->increments('id');
                $t->integer('review_id')->unsigned();
                $t->integer('user_id')->unsigned()->nullable();
                $t->string('ip_address', 45)->nullable();
                $t->boolean('is_helpful');
                $t->timestamp('created_at');

                $t->unique(['review_id', 'user_id']);
            });
        }

        // Review responses
        if (!Capsule::schema()->hasTable('mod_{reviews}_responses')) {
            Capsule::schema()->create('mod_{reviews}_responses', function($t) {
                $t->increments('id');
                $t->integer('review_id')->unsigned();
                $t->text('content');
                $t->integer('user_id')->unsigned()->nullable();
                $t->string('author_name', 100);
                $t->timestamp('created_at');
            });
        }

        // Review reports
        if (!Capsule::schema()->hasTable('mod_{reviews}_reports')) {
            Capsule::schema()->create('mod_{reviews}_reports', function($t) {
                $t->increments('id');
                $t->integer('review_id')->unsigned();
                $t->integer('user_id')->unsigned()->nullable();
                $t->string('reason', 50);
                $t->text('details')->nullable();
                $t->string('status', 20)->default('pending');
                $t->timestamp('created_at');
            });
        }

        // Rating cache
        if (!Capsule::schema()->hasTable('mod_{reviews}_rating_cache')) {
            Capsule::schema()->create('mod_{reviews}_rating_cache', function($t) {
                $t->string('review_type', 20);
                $t->integer('rel_id')->unsigned();
                $t->decimal('avg_rating', 3, 2)->default(0);
                $t->integer('total_reviews')->default(0);
                $t->integer('rating_1')->default(0);
                $t->integer('rating_2')->default(0);
                $t->integer('rating_3')->default(0);
                $t->integer('rating_4')->default(0);
                $t->integer('rating_5')->default(0);
                $t->timestamp('updated_at');

                $t->primary(['review_type', 'rel_id']);
            });
        }

        return ['status' => 'success', 'description' => '{Reviews Module} activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => 'Activation failed: ' . $e->getMessage()];
    }
}

function {reviews}_deactivate(): array {
    try {
        Capsule::schema()->dropIfExists('mod_{reviews}_reviews');
        Capsule::schema()->dropIfExists('mod_{reviews}_votes');
        Capsule::schema()->dropIfExists('mod_{reviews}_responses');
        Capsule::schema()->dropIfExists('mod_{reviews}_reports');
        Capsule::schema()->dropIfExists('mod_{reviews}_rating_cache');
        return ['status' => 'success'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => 'Deactivation failed'];
    }
}

function {reviews}_output(array $vars): void {
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
        handleReviewAction($_POST['action'] ?? '');
    }

    $tab = $_REQUEST['tab'] ?? 'overview';
    echo '<div class="reviews-module">';
    echo '<h1><i class="fa fa-star"></i> Reviews Management</h1>';
    echo '<ul class="nav nav-tabs">';
    echo '<li class="' . ($tab === 'overview' ? 'active' : '') . '"><a href="?module={reviews}&tab=overview">Overview</a></li>';
    echo '<li class="' . ($tab === 'pending' ? 'active' : '') . '"><a href="?module={reviews}&tab=pending">Pending</a></li>';
    echo '<li class="' . ($tab === 'approved' ? 'active' : '') . '"><a href="?module={reviews}&tab=approved">Approved</a></li>';
    echo '<li class="' . ($tab === 'reports' ? 'active' : '') . '"><a href="?module={reviews}&tab=reports">Reports</a></li>';
    echo '</ul>';
    include __DIR__ . '/templates/admin/' . $tab . '.tpl';
    echo '</div>';
}

function {reviews}_clientarea(array $vars): array {
    return [
        'pagetitle' => 'Reviews',
        'templatefile' => 'templates/clientarea',
        'requirelogin' => false,
    ];
}

function handleReviewAction(string $action): void {
    switch ($action) {
        case 'approve_review':
            Capsule::table('mod_{reviews}_reviews')
                ->where('id', (int)($_POST['review_id'] ?? 0))
                ->update(['status' => 'approved', 'approved_at' => date('Y-m-d H:i:s')]);
            break;
        case 'reject_review':
            Capsule::table('mod_{reviews}_reviews')
                ->where('id', (int)($_POST['review_id'] ?? 0))
                ->update(['status' => 'rejected']);
            break;
        case 'delete_review':
            Capsule::table('mod_{reviews}_reviews')
                ->where('id', (int)($_POST['review_id'] ?? 0))
                ->delete();
            break;
    }
    header('Location: ?module={reviews}&tab=' . ($_POST['redirect_tab'] ?? 'overview'));
    exit;
}

function getReviewStats(): array {
    return [
        'total' => Capsule::table('mod_{reviews}_reviews')->count(),
        'pending' => Capsule::table('mod_{reviews}_reviews')->where('status', 'pending')->count(),
        'approved' => Capsule::table('mod_{reviews}_reviews')->where('status', 'approved')->count(),
        'avg_rating' => Capsule::table('mod_{reviews}_reviews')->where('status', 'approved')->avg('rating') ?? 0,
    ];
}
```

## Review Manager Class

```php
<?php
namespace WHMCS\Module\Addon\{Reviews};

use WHMCS\Database\Capsule;

class ReviewManager {

    public function createReview(array $data): array {
        $settings = $this->getModuleSettings();

        // Validate rating
        $minRating = (int)($settings['min_rating'] ?? 1);
        if ($data['rating'] < $minRating) {
            return ['success' => false, 'message' => 'Rating below minimum'];
        }

        // Check rate limit
        if ($this->isRateLimited($data['user_id'] ?? 0)) {
            return ['success' => false, 'message' => 'Too many reviews today'];
        }

        // Check if verified purchase required
        if (($settings['require_purchase'] ?? '') === 'on') {
            if (!$this->hasPurchased($data['user_id'] ?? 0, $data['rel_id'] ?? 0)) {
                return ['success' => false, 'message' => 'Purchase required'];
            }
        }

        // Check for existing review
        if ($this->hasReviewed($data['user_id'] ?? 0, $data['rel_id'] ?? 0)) {
            return ['success' => false, 'message' => 'Already reviewed'];
        }

        $reviewData = [
            'review_type' => $data['review_type'] ?? 'product',
            'rel_id' => $data['rel_id'],
            'user_id' => $data['user_id'] ?? null,
            'user_name' => $data['user_name'] ?? null,
            'user_email' => $data['user_email'] ?? null,
            'rating' => $data['rating'],
            'title' => $data['title'],
            'content' => $data['content'],
            'pros' => $data['pros'] ?? null,
            'cons' => $data['cons'] ?? null,
            'status' => ($settings['moderate_reviews'] ?? '') === 'on' ? 'pending' : 'approved',
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
            'created_at' => date('Y-m-d H:i:s'),
        ];

        $reviewId = Capsule::table('mod_{reviews}_reviews')->insertGetId($reviewData);

        if ($reviewData['status'] === 'approved') {
            $this->updateRatingCache($data['review_type'] ?? 'product', $data['rel_id']);
        }

        return ['success' => true, 'review_id' => $reviewId];
    }

    public function getReviews(string $type, int $relId, string $status = 'approved', int $limit = 10): array {
        return Capsule::table('mod_{reviews}_reviews')
            ->where('review_type', $type)
            ->where('rel_id', $relId)
            ->where('status', $status)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    public function voteHelpful(int $reviewId, int $userId, bool $isHelpful): array {
        $existing = Capsule::table('mod_{reviews}_votes')
            ->where('review_id', $reviewId)
            ->where('user_id', $userId)
            ->first();

        if ($existing) {
            Capsule::table('mod_{reviews}_votes')
                ->where('id', $existing->id)
                ->update(['is_helpful' => $isHelpful]);
        } else {
            Capsule::table('mod_{reviews}_votes')->insert([
                'review_id' => $reviewId,
                'user_id' => $userId,
                'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
                'is_helpful' => $isHelpful,
                'created_at' => date('Y-m-d H:i:s'),
            ]);
            $column = $isHelpful ? 'helpful_yes' : 'helpful_no';
            Capsule::table('mod_{reviews}_reviews')
                ->where('id', $reviewId)
                ->increment($column);
        }

        return ['success' => true];
    }

    public function getRatingStats(string $type, int $relId): array {
        $cache = Capsule::table('mod_{reviews}_rating_cache')
            ->where('review_type', $type)
            ->where('rel_id', $relId)
            ->first();

        if (!$cache) {
            return ['total' => 0, 'avg_rating' => 0, 'distribution' => [1 => 0, 2 => 0, 3 => 0, 4 => 0, 5 => 0]];
        }

        return [
            'total' => $cache->total_reviews,
            'avg_rating' => $cache->avg_rating,
            'distribution' => [
                1 => $cache->rating_1,
                2 => $cache->rating_2,
                3 => $cache->rating_3,
                4 => $cache->rating_4,
                5 => $cache->rating_5,
            ],
        ];
    }

    private function isRateLimited(int $userId): bool {
        $maxPerDay = 5;
        return Capsule::table('mod_{reviews}_reviews')
            ->where('user_id', $userId)
            ->whereDate('created_at', date('Y-m-d'))
            ->count() >= $maxPerDay;
    }

    private function hasPurchased(int $userId, int $relId): bool {
        return Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->where('packageid', $relId)
            ->whereIn('domainstatus', ['Active', 'Suspended'])
            ->exists();
    }

    private function hasReviewed(int $userId, int $relId): bool {
        return Capsule::table('mod_{reviews}_reviews')
            ->where('user_id', $userId)
            ->where('rel_id', $relId)
            ->exists();
    }

    private function updateRatingCache(string $type, int $relId): void {
        $reviews = Capsule::table('mod_{reviews}_reviews')
            ->where('review_type', $type)
            ->where('rel_id', $relId)
            ->where('status', 'approved')
            ->get();

        $total = count($reviews);
        $sum = 0;
        $counts = [1 => 0, 2 => 0, 3 => 0, 4 => 0, 5 => 0];

        foreach ($reviews as $r) {
            $sum += $r->rating;
            $counts[$r->rating]++;
        }

        $avg = $total > 0 ? round($sum / $total, 2) : 0;

        Capsule::table('mod_{reviews}_rating_cache')
            ->updateOrInsert(
                ['review_type' => $type, 'rel_id' => $relId],
                [
                    'avg_rating' => $avg, 'total_reviews' => $total,
                    'rating_1' => $counts[1], 'rating_2' => $counts[2],
                    'rating_3' => $counts[3], 'rating_4' => $counts[4], 'rating_5' => $counts[5],
                    'updated_at' => date('Y-m-d H:i:s'),
                ]
            );
    }

    private function getModuleSettings(): array {
        $result = Capsule::table('tbladdon_modules')->where('module', '{reviews}')->first();
        return $result ? json_decode($result->value, true) : [];
    }
}
```

## Hooks Integration

```php
<?php
/**
 * WHMCS Reviews Module Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Send review request after service activation
add_hook('AfterModuleCreate', 1, function(array $vars) {
    $serviceId = $vars['serviceid'];
    $userId = $vars['params']['userid'] ?? 0;

    // Queue review request
    Capsule::table('mod_{reviews}_review_queue')->insert([
        'user_id' => $userId,
        'rel_id' => $serviceId,
        'review_type' => 'service',
        'scheduled_at' => date('Y-m-d H:i:s', strtotime('+7 days')),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
});

// Process review queue daily
add_hook('DailyCronJob', 1, function(array $vars) {
    $pending = Capsule::table('mod_{reviews}_review_queue')
        ->where('status', 'pending')
        ->where('scheduled_at', '<=', date('Y-m-d H:i:s'))
        ->limit(100)
        ->get();

    foreach ($pending as $request) {
        $user = Capsule::table('tblclients')->where('id', $request->user_id)->first();

        if ($user && !$this->hasReviewed($request->user_id, $request->rel_id)) {
            sendEmail($user->email, 'ReviewRequest', [
                'user_id' => $user->id,
                'rel_id' => $request->rel_id,
                'review_type' => $request->review_type,
            ]);
        }

        Capsule::table('mod_{reviews}_review_queue')
            ->where('id', $request->id)
            ->update(['status' => 'sent', 'sent_at' => date('Y-m-d H:i:s')]);
    }
});
```

## Database Schema

### mod_{reviews}_reviews
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| review_type | VARCHAR(20) | product/service/hosting |
| rel_id | INT | Related product/service ID |
| user_id | INT | Reviewer user ID |
| user_name | VARCHAR(100) | Display name |
| rating | INT | 1-5 stars |
| title | VARCHAR(255) | Review title |
| content | TEXT | Review body |
| pros | VARCHAR(500) | Pros |
| cons | VARCHAR(500) | Cons |
| status | VARCHAR(20) | pending/approved/rejected |
| helpful_yes | INT | Helpful votes |
| helpful_no | INT | Not helpful votes |
| created_at | TIMESTAMP | Creation time |

### mod_{reviews}_rating_cache
| Column | Type | Description |
|--------|------|-------------|
| review_type | VARCHAR(20) | Type (primary key) |
| rel_id | INT | ID (primary key) |
| avg_rating | DECIMAL(3,2) | Average rating |
| total_reviews | INT | Total count |
| rating_1-5 | INT | Count per star |

## Checklist

```
Pre-Dev:
□ Define review types
□ Plan moderation workflow
□ Design rating system
□ Plan helpful vote system

Development:
□ Implement config() with all settings
□ Implement activate() → Create tables
□ Implement deactivate() → Drop tables
□ Create ReviewManager class
□ Implement review submission
□ Add review moderation
□ Implement helpful votes
□ Create admin interface
□ Create client templates
□ Add hooks for review requests

Security:
□ Prevent review spam
□ Implement rate limiting
□ Validate inputs
□ Use check_token() for POST

Testing:
□ Test review submission
□ Test moderation workflow
□ Test helpful votes
□ Test rating calculations
□ Verify email notifications
```
