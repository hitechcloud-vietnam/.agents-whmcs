# WHMCS Knowledge Base Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building knowledge base/documents modules.

## When to Use

- Self-service documentation
- FAQ systems
- Product guides

## Knowledge Base Patterns

```php
<?php
class KnowledgeBase {
    public function createArticle(array $data): int {
        $articleId = Capsule::table('mod_kb_articles')->insertGetId([
            'title' => $data['title'],
            'slug' => $this->generateSlug($data['title']),
            'content' => $data['content'],
            'category_id' => $data['category_id'],
            'tags' => implode(',', $data['tags'] ?? []),
            'views' => 0,
            'helpful' => 0,
            'not_helpful' => 0,
            'published' => $data['published'] ?? false,
            'published_at' => $data['published'] ? date('Y-m-d H:i:s') : null,
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        // Update search index
        $this->updateSearchIndex($articleId, $data);

        return $articleId;
    }

    public function search(string $query, int $limit = 20): array {
        $articles = Capsule::table('mod_kb_articles')
            ->where('published', 1)
            ->where(function($q) use ($query) {
                $q->where('title', 'like', "%$query%")
                  ->orWhere('content', 'like', "%$query%");
            })
            ->selectRaw(" *,
                MATCH(title, content) AGAINST('$query' IN NATURAL LANGUAGE MODE) as relevance
            ")
            ->orderBy('relevance', 'desc')
            ->limit($limit)
            ->get();

        // Increment view count
        foreach ($articles as $article) {
            Capsule::table('mod_kb_articles')
                ->where('id', $article->id)
                ->increment('views');
        }

        return $articles;
    }

    public function getRelated(int $articleId, int $limit = 5): array {
        $article = Capsule::table('mod_kb_articles')->where('id', $articleId)->first();
        $tags = explode(',', $article->tags);

        return Capsule::table('mod_kb_articles')
            ->where('id', '!=', $articleId)
            ->where('published', 1)
            ->where(function($q) use ($tags) {
                foreach ($tags as $tag) {
                    $q->orWhere('tags', 'like', '%' . trim($tag) . '%');
                }
            })
            ->limit($limit)
            ->get();
    }

    public function rateArticle(int $articleId, bool $helpful): void {
        $field = $helpful ? 'helpful' : 'not_helpful';
        Capsule::table('mod_kb_articles')
            ->where('id', $articleId)
            ->increment($field);
    }
}
```

### Category Management
```php
public function getCategoryTree(): array {
    $categories = Capsule::table('mod_kb_categories')
        ->orderBy('parent_id')
        ->orderBy('sort_order')
        ->get();

    return $this->buildTree($categories);
}

private function buildTree($categories, int $parentId = 0): array {
    $tree = [];
    foreach ($categories as $cat) {
        if ($cat->parent_id == $parentId) {
            $children = $this->buildTree($categories, $cat->id);
            if ($children) {
                $cat->children = $children;
            }
            $tree[] = $cat;
        }
    }
    return $tree;
}
```

---

**Related Skills:**
- whmcs-support-ticket-module
- whmcs-clientarea-builder
