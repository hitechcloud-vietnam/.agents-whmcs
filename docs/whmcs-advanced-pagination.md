# WHMCS Pagination

Complete guide to pagination patterns.

## Overview

Implement efficient pagination for large datasets.

## Pagination Helper

```php
<?php
/**
 * Pagination helper
 */
class Pagination
{
    /**
     * Paginate query
     */
    public static function paginate(Builder $query, int $page = 1, int $perPage = 20): array
    {
        $total = $query->count();
        $offset = ($page - 1) * $perPage;
        
        $items = $query->offset($offset)->limit($perPage)->get();
        
        return [
            'data' => $items,
            'pagination' => [
                'current_page' => $page,
                'per_page' => $perPage,
                'total' => $total,
                'total_pages' => ceil($total / $perPage),
                'has_next' => $page < ceil($total / $perPage),
                'has_prev' => $page > 1,
            ],
        ];
    }
    
    /**
     * Generate pagination links
     */
    public static function links(array $pagination, string $baseUrl): string
    {
        $html = '<nav class="pagination">';
        
        if ($pagination['has_prev']) {
            $html .= '<a href="' . $baseUrl . '?page=' . ($pagination['current_page'] - 1) . '">Previous</a>';
        }
        
        for ($i = 1; $i <= $pagination['total_pages']; $i++) {
            $active = $i === $pagination['current_page'] ? ' class="active"' : '';
            $html .= '<a href="' . $baseUrl . '?page=' . $i . '"' . $active . '>' . $i . '</a>';
        }
        
        if ($pagination['has_next']) {
            $html .= '<a href="' . $baseUrl . '?page=' . ($pagination['current_page'] + 1) . '">Next</a>';
        }
        
        $html .= '</nav>';
        
        return $html;
    }
}
```

## Best Practices

1. **Limit results** - Never load all data
2. **Index properly** - Use indexed columns for ordering
3. **Cache counts** - Cache total counts
4. **Cursor pagination** - For very large datasets
5. **AJAX loading** - Load more on scroll
6. **SEO** - Use proper pagination markup

## Related Documentation

- [whmcs-advanced-database.md](whmcs-advanced-database.md)
- [whmcs-advanced-performance.md](whmcs-advanced-performance.md)
