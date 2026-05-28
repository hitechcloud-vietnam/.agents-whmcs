# WHMCS Product Review Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building product review and rating modules.

## When to Use

- Product review modules
- Rating systems
- Feedback collection

## Review Patterns

```php
<?php
class ProductReview {
    public function submitReview(array $data): array {
        // Validate
        if (!in_array($data['rating'], [1, 2, 3, 4, 5])) {
            return ['error' => 'Rating must be 1-5'];
        }

        // Check for duplicate
        $existing = Capsule::table('mod_reviews')
            ->where('user_id', $data['user_id'])
            ->where('product_id', $data['product_id'])
            ->first();

        if ($existing) {
            return ['error' => 'You have already reviewed this product'];
        }

        // Insert review
        $reviewId = Capsule::table('mod_reviews')->insertGetId([
            'product_id' => $data['product_id'],
            'user_id' => $data['user_id'],
            'rating' => $data['rating'],
            'title' => $data['title'] ?? '',
            'content' => $data['content'],
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        return ['success' => true, 'review_id' => $reviewId];
    }

    public function getProductRating(int $productId): array {
        $stats = Capsule::table('mod_reviews')
            ->where('product_id', $productId)
            ->where('status', 'approved')
            ->selectRaw('
                COUNT(*) as count,
                AVG(rating) as average,
                SUM(CASE WHEN rating = 1 THEN 1 ELSE 0 END) as star1,
                SUM(CASE WHEN rating = 2 THEN 1 ELSE 0 END) as star2,
                SUM(CASE WHEN rating = 3 THEN 1 ELSE 0 END) as star3,
                SUM(CASE WHEN rating = 4 THEN 1 ELSE 0 END) as star4,
                SUM(CASE WHEN rating = 5 THEN 1 ELSE 0 END) as star5
            ')
            ->first();

        return [
            'count' => $stats->count ?? 0,
            'average' => round($stats->average ?? 0, 1),
            'distribution' => [
                5 => $stats->star5 ?? 0,
                4 => $stats->star4 ?? 0,
                3 => $stats->star3 ?? 0,
                2 => $stats->star2 ?? 0,
                1 => $stats->star1 ?? 0,
            ],
        ];
    }
}
```

### Admin Moderation
```php
public function approveReview(int $reviewId): bool {
    Capsule::table('mod_reviews')
        ->where('id', $reviewId)
        ->update([
            'status' => 'approved',
            'approved_at' => date('Y-m-d H:i:s'),
        ]);

    return true;
}
```

---

**Related Skills:**
- whmcs-admin-ui-builder
- whmcs-reporting
