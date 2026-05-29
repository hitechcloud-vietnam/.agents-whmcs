# WHMCS API Pagination

## Skill Description
Implement comprehensive pagination patterns for WHMCS API endpoints including cursor-based pagination, offset pagination, and collection metadata.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- Database query knowledge
- Understanding of pagination algorithms

## Step-by-Step Implementation

### 1. Pagination Builder
```php
<?php
// includes/pagination/PaginationBuilder.php

namespace WHMCS\Module\YourModule\Pagination;

class PaginationBuilder
{
    private int $currentPage;
    private int $perPage;
    private int $totalItems;
    private int $totalPages;
    private int $offset;
    private string $sortBy = 'id';
    private string $sortOrder = 'asc';

    public function __construct(
        int $currentPage = 1,
        int $perPage = 20,
        int $maxPerPage = 100
    ) {
        $this->currentPage = max(1, $currentPage);
        $this->perPage = min($maxPerPage, max(1, $perPage));
    }

    public function setTotalItems(int $total): self
    {
        $this->totalItems = $total;
        $this->totalPages = (int) ceil($total / $this->perPage);
        $this->offset = ($this->currentPage - 1) * $this->perPage;
        return $this;
    }

    public function setSorting(string $sortBy, string $sortOrder = 'asc'): self
    {
        $allowedColumns = ['id', 'created_at', 'updated_at', 'name', 'email'];
        $allowedOrders = ['asc', 'desc'];

        if (in_array($sortBy, $allowedColumns)) {
            $this->sortBy = $sortBy;
        }

        if (in_array(strtolower($sortOrder), $allowedOrders)) {
            $this->sortOrder = strtolower($sortOrder);
        }

        return $this;
    }

    public function getOffset(): int
    {
        return $this->offset;
    }

    public function getLimit(): int
    {
        return $this->perPage;
    }

    public function getOrderBy(): string
    {
        return $this->sortBy . ' ' . strtoupper($this->sortOrder);
    }

    public function getMeta(): array
    {
        return [
            'pagination' => [
                'current_page' => $this->currentPage,
                'per_page' => $this->perPage,
                'total_items' => $this->totalItems,
                'total_pages' => $this->totalPages,
                'has_next' => $this->currentPage < $this->totalPages,
                'has_previous' => $this->currentPage > 1,
                'first_item' => $this->totalItems > 0 ? $this->offset + 1 : 0,
                'last_item' => min($this->offset + $this->perPage, $this->totalItems)
            ],
            'links' => $this->getLinks(),
            'sorting' => [
                'sort_by' => $this->sortBy,
                'sort_order' => $this->sortOrder
            ]
        ];
    }

    private function getLinks(): array
    {
        $baseUrl = $this->getBaseUrl();
        $queryParams = $this->getQueryParams();

        return [
            'self' => $this->buildUrl($baseUrl, array_merge($queryParams, ['page' => $this->currentPage])),
            'first' => $this->buildUrl($baseUrl, array_merge($queryParams, ['page' => 1])),
            'last' => $this->buildUrl($baseUrl, array_merge($queryParams, ['page' => $this->totalPages])),
            'next' => $this->currentPage < $this->totalPages
                ? $this->buildUrl($baseUrl, array_merge($queryParams, ['page' => $this->currentPage + 1]))
                : null,
            'previous' => $this->currentPage > 1
                ? $this->buildUrl($baseUrl, array_merge($queryParams, ['page' => $this->currentPage - 1]))
                : null
        ];
    }

    private function getBaseUrl(): string
    {
        $scheme = isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on' ? 'https' : 'http';
        $host = $_SERVER['HTTP_HOST'] ?? 'localhost';
        $path = parse_url($_SERVER['REQUEST_URI'] ?? '', PHP_URL_PATH);
        return $scheme . '://' . $host . $path;
    }

    private function getQueryParams(): array
    {
        $params = $_GET;
        unset($params['page']); // Remove page from params for link generation
        return $params;
    }

    private function buildUrl(string $base, array $params): string
    {
        if (empty($params)) {
            return $base;
        }
        return $base . '?' . http_build_query($params);
    }

    public static function fromRequest(array $defaultParams = []): self
    {
        $page = isset($_GET['page']) ? (int) $_GET['page'] : ($defaultParams['page'] ?? 1);
        $perPage = isset($_GET['per_page'])
            ? (int) $_GET['per_page']
            : ($defaultParams['per_page'] ?? 20);
        $sortBy = $_GET['sort_by'] ?? ($defaultParams['sort_by'] ?? 'id');
        $sortOrder = $_GET['sort_order'] ?? ($defaultParams['sort_order'] ?? 'asc');

        return (new self($page, $perPage))
            ->setSorting($sortBy, $sortOrder);
    }
}
```

### 2. Cursor Paginator
```php
<?php
// includes/pagination/CursorPaginator.php

namespace WHMCS\Module\YourModule\Pagination;

class CursorPaginator
{
    private string $cursorColumn;
    private $cursorValue;
    private int $perPage;
    private string $sortOrder;
    private array $items = [];
    private bool $hasMore = false;
    private bool $hasPrevious = false;

    public function __construct(
        string $cursorColumn = 'id',
        int $perPage = 20,
        int $maxPerPage = 100
    ) {
        $this->cursorColumn = $cursorColumn;
        $this->perPage = min($maxPerPage, max(1, $perPage));
        $this->sortOrder = 'desc'; // Default to descending for "newest first"
    }

    public function paginate(\Illuminate\Database\Eloquent\Builder $query, ?string $cursor = null): array
    {
        // Decode cursor
        if ($cursor) {
            $this->cursorValue = $this->decodeCursor($cursor);
            $this->hasPrevious = true;
        }

        // Build query
        if ($this->cursorValue !== null) {
            $operator = $this->sortOrder === 'desc' ? '<' : '>';
            $query->where($this->cursorColumn, $operator, $this->cursorValue);
        }

        $query->orderBy($this->cursorColumn, $this->sortOrder);
        $query->limit($this->perPage + 1); // Fetch one extra to check for more

        $this->items = $query->get()->toArray();

        // Check if there are more items
        if (count($this->items) > $this->perPage) {
            array_pop($this->items);
            $this->hasMore = true;
        }

        return $this->buildResponse();
    }

    private function decodeCursor(string $cursor): mixed
    {
        $decoded = base64_decode($cursor);
        $data = json_decode($decoded, true);
        return $data[$this->cursorColumn] ?? null;
    }

    private function encodeCursor(array $item): string
    {
        $data = [$this->cursorColumn => $item[$this->cursorColumn]];
        return base64_encode(json_encode($data));
    }

    private function buildResponse(): array
    {
        $lastItem = end($this->items);

        return [
            'data' => $this->items,
            'meta' => [
                'per_page' => $this->perPage,
                'has_more' => $this->hasMore,
                'has_previous' => $this->hasPrevious,
                'cursor_column' => $this->cursorColumn
            ],
            'cursors' => [
                'next' => $this->hasMore && $lastItem ? $this->encodeCursor($lastItem) : null,
                'previous' => $this->hasPrevious && !empty($this->items)
                    ? $this->encodeCursor($this->items[0])
                    : null
            ]
        ];
    }

    public static function fromRequest(string $cursorParam = 'cursor', int $defaultPerPage = 20): self
    {
        $cursor = $_GET[$cursorParam] ?? null;
        $perPage = isset($_GET['per_page'])
            ? (int) $_GET['per_page']
            : $defaultPerPage;

        $paginator = new self('id', $perPage);

        if ($cursor) {
            $paginator->cursorValue = $cursor;
        }

        return $paginator;
    }
}
```

### 3. Pagination Controller Trait
```php
<?php
// includes/pagination/PaginatesRequests.php

namespace WHMCS\Module\YourModule\Traits;

trait PaginatesRequests
{
    protected function paginate(
        \Illuminate\Database\Eloquent\Builder $query,
        array $options = []
    ): array {
        $pagination = \WHMCS\Module\YourModule\Pagination\PaginationBuilder::fromRequest([
            'page' => $options['default_page'] ?? 1,
            'per_page' => $options['default_per_page'] ?? 20,
            'sort_by' => $options['default_sort_by'] ?? 'id',
            'sort_order' => $options['default_sort_order'] ?? 'asc'
        ]);

        $total = $query->count();
        $pagination->setTotalItems($total);

        $items = $query
            ->orderByRaw($pagination->getOrderBy())
            ->offset($pagination->getOffset())
            ->limit($pagination->getLimit())
            ->get();

        return [
            'data' => $items,
            'meta' => $pagination->getMeta()
        ];
    }

    protected function paginateCursor(
        \Illuminate\Database\Eloquent\Builder $query,
        string $cursorColumn = 'id',
        int $perPage = 20
    ): array {
        $cursor = $_GET['cursor'] ?? null;

        $paginator = new \WHMCS\Module\YourModule\Pagination\CursorPaginator(
            $cursorColumn,
            $perPage
        );

        return $paginator->paginate($query, $cursor);
    }

    protected function getPageInfo(): array
    {
        return [
            'page' => (int) ($_GET['page'] ?? 1),
            'per_page' => (int) ($_GET['per_page'] ?? 20),
            'sort_by' => $_GET['sort_by'] ?? 'id',
            'sort_order' => $_GET['sort_order'] ?? 'asc'
        ];
    }
}
```

### 4. Usage Example
```php
<?php
// Example API controller using pagination

namespace WHMCS\Module\YourModule\Api\Controllers;

use WHMCS\Module\YourModule\Api\ApiController;
use WHMCS\Module\YourModule\Api\ApiResponse;
use WHMCS\Module\YourModule\Traits\PaginatesRequests;

class InvoiceController extends ApiController
{
    use PaginatesRequests;

    public function index(): void
    {
        $this->requireAuth();

        $query = \WHMCS\Billing\Invoice::query();

        // Apply filters
        if ($status = $_GET['status'] ?? null) {
            $query->where('status', $status);
        }

        if ($userId = $_GET['user_id'] ?? null) {
            $query->where('userid', $userId);
        }

        // Date range filter
        if ($fromDate = $_GET['from_date'] ?? null) {
            $query->where('date', '>=', $fromDate);
        }

        if ($toDate = $_GET['to_date'] ?? null) {
            $query->where('date', '<=', $toDate);
        }

        $result = $this->paginate($query, [
            'default_per_page' => 25,
            'default_sort_by' => 'created_at',
            'default_sort_order' => 'desc'
        ]);

        ApiResponse::success($result['data'])
            ->addMeta('pagination', $result['meta']['pagination'])
            ->addMeta('links', $result['meta']['links'])
            ->send();
    }

    public function indexCursor(): void
    {
        $this->requireAuth();

        $query = \WHMCS\Billing\Invoice::query();
        $query->orderBy('id', 'desc');

        $result = $this->paginateCursor($query, 'id', 20);

        ApiResponse::success($result['data'])
            ->addMeta('has_more', $result['meta']['has_more'])
            ->addMeta('cursor_column', $result['meta']['cursor_column'])
            ->send();
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Pagination count queries are slow | Use estimated counts or cursor pagination |
| Invalid page numbers | Clamp page to valid range (1 to totalPages) |
| URL encoding issues with cursors | Base64 encode cursor values |
| Inconsistent ordering | Always include unique column in sort |
| Large offset performance | Use cursor pagination for deep pagination |

## Security Considerations

1. **Limit max per_page** - Prevent expensive queries with large limits
2. **Sanitize sort columns** - Whitelist allowed sort columns
3. **Rate limit paginated requests** - Large result sets are resource intensive
4. **Cache common pages** - Cache first few pages of popular endpoints
5. **Log deep pagination access** - Track suspicious pagination patterns

## Testing Checklist

- [ ] Test pagination with valid page number
- [ ] Test pagination with page beyond total
- [ ] Test pagination with per_page limit
- [ ] Test sorting with allowed columns
- [ ] Test sorting with invalid columns
- [ ] Test cursor pagination forward
- [ ] Test cursor pagination backward
- [ ] Test empty results pagination
- [ ] Test pagination links generation
- [ ] Test maximum per_page enforcement

## Reference Links

- [Pagination Design Best Practices](https://developer.spotify.com/documentation/web-api/#pagination)
- [Cursor-based Pagination](https://dev.twitter.com/overview/api/cursoring)
- [WHMCS Query Builder](https://developers.whmcs.com/advanced/using-the-database/)
