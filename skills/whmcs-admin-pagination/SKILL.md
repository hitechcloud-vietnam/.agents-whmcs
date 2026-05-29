# WHMCS Admin Pagination

## Overview
Guide for implementing pagination in WHMCS admin area. Covers pagination helpers, URL parameters, and custom pagination styles.

## Pagination System

### Pagination Helper

```php
<?php
// /includes/hooks/admin_pagination.php

class PaginationHelper {
    private int $total;
    private int $perPage;
    private int $currentPage;
    private string $baseUrl;
    
    public function __construct(
        int $total,
        int $perPage = 20,
        ?int $currentPage = null
    ) {
        $this->total = $total;
        $this->perPage = $perPage;
        $this->currentPage = $currentPage ?? $this->getCurrentPage();
        $this->baseUrl = $this->getBaseUrl();
    }
    
    private function getCurrentPage(): int {
        return max(1, (int)($_GET["page"] ?? 1));
    }
    
    private function getBaseUrl(): string {
        $url = parse_url($_SERVER["REQUEST_URI"], PHP_URL_PATH);
        $query = $_GET;
        unset($query["page"]);
        if (!empty($query)) {
            $url .= "?" . http_build_query($query);
        }
        return $url;
    }
    
    public function offset(): int {
        return ($this->currentPage - 1) * $this->perPage;
    }
    
    public function totalPages(): int {
        return (int)ceil($this->total / $this->perPage);
    }
    
    public function hasNext(): bool {
        return $this->currentPage < $this->totalPages();
    }
    
    public function hasPrevious(): bool {
        return $this->currentPage > 1;
    }
    
    public function nextPage(): int {
        return min($this->currentPage + 1, $this->totalPages());
    }
    
    public function previousPage(): int {
        return max(1, $this->currentPage - 1);
    }
    
    public function getPageUrl(int $page): string {
        $separator = strpos($this->baseUrl, "?") !== false ? "&" : "?";
        return $this->baseUrl . $separator . "page=" . $page;
    }
    
    public function getRange(int $range = 2): array {
        $pages = [];
        $start = max(1, $this->currentPage - $range);
        $end = min($this->totalPages(), $this->currentPage + $range);
        
        for ($i = $start; $i <= $end; $i++) {
            $pages[] = $i;
        }
        
        return $pages;
    }
    
    public function toArray(): array {
        return [
            "total" => $this->total,
            "per_page" => $this->perPage,
            "current_page" => $this->currentPage,
            "total_pages" => $this->totalPages(),
            "has_next" => $this->hasNext(),
            "has_previous" => $this->hasPrevious(),
            "offset" => $this->offset(),
            "range" => $this->getRange(),
            "first_page" => $this->getPageUrl(1),
            "last_page" => $this->getPageUrl($this->totalPages()),
            "next_page" => $this->getPageUrl($this->nextPage()),
            "previous_page" => $this->getPageUrl($this->previousPage())
        ];
    }
}
```

### Paginated Query

```php
function paginatedQuery(
    string $table,
    array $filters = [],
    int $page = 1,
    int $perPage = 20
): array {
    $query = Capsule::table($table);
    
    // Apply filters
    foreach ($filters as $field => $value) {
        if ($value !== null && $value !== "") {
            if (is_array($value)) {
                $query->whereIn($field, $value);
            } else {
                $query->where($field, $value);
            }
        }
    }
    
    // Get total count
    $total = $query->count();
    
    // Apply pagination
    $data = $query
        ->orderBy("id", "desc")
        ->offset(($page - 1) * $perPage)
        ->limit($perPage)
        ->get();
    
    $pagination = new PaginationHelper($total, $perPage, $page);
    
    return [
        "data" => $data,
        "pagination" => $pagination->toArray()
    ];
}
```

## Pagination Template

```smarty
<!-- /admin/templates/pagination.tpl -->
{if $pagination.total_pages > 1}
<nav class="pagination-nav" aria-label="Page navigation">
    <div class="pagination-info">
        Showing 
        <strong>{($pagination.current_page - 1) * $pagination.per_page + 1}</strong>
        to 
        <strong>{min($pagination.current_page * $pagination.per_page, $pagination.total)}</strong>
        of 
        <strong>{$pagination.total}</strong>
        entries
    </div>
    
    <ul class="pagination">
        {* First Page *}
        <li class="{if !$pagination.has_previous}disabled{/if}">
            <a href="{$pagination.first_page}" aria-label="First">
                <span aria-hidden="true">&laquo;</span>
            </a>
        </li>
        
        {* Previous Page *}
        <li class="{if !$pagination.has_previous}disabled{/if}">
            <a href="{$pagination.previous_page}" aria-label="Previous">
                <span aria-hidden="true">&lsaquo;</span>
            </a>
        </li>
        
        {* Page Numbers *}
        {if $pagination.current_page - 3 > 1}
            <li>
                <a href="{$pagination.getPageUrl(1)}">1</a>
            </li>
            {if $pagination.current_page - 3 > 2}
                <li class="disabled">
                    <span>...</span>
                </li>
            {/if}
        {/if}
        
        {foreach $pagination.range as $page}
            <li class="{if $page == $pagination.current_page}active{/if}">
                <a href="{$pagination.getPageUrl($page)}">
                    {$page}
                </a>
            </li>
        {/foreach}
        
        {if $pagination.current_page + 3 < $pagination.total_pages}
            {if $pagination.current_page + 3 < $pagination.total_pages - 1}
                <li class="disabled">
                    <span>...</span>
                </li>
            {/if}
            <li>
                <a href="{$pagination.getPageUrl($pagination.total_pages)}">
                    {$pagination.total_pages}
                </a>
            </li>
        {/if}
        
        {* Next Page *}
        <li class="{if !$pagination.has_next}disabled{/if}">
            <a href="{$pagination.next_page}" aria-label="Next">
                <span aria-hidden="true">&rsaquo;</span>
            </a>
        </li>
        
        {* Last Page *}
        <li class="{if !$pagination.has_next}disabled{/if}">
            <a href="{$pagination.last_page}" aria-label="Last">
                <span aria-hidden="true">&raquo;</span>
            </a>
        </li>
    </ul>
    
    {* Per Page Selector *}
    <div class="pagination-per-page">
        <label>Show:</label>
        <select name="per_page" onchange="location.href=this.value">
            <option value="{$pagination.getPageUrl(1)|replace:'page=1':'per_page=10'}">10</option>
            <option value="{$pagination.getPageUrl(1)|replace:'page=1':'per_page=25'}">25</option>
            <option value="{$pagination.getPageUrl(1)|replace:'page=1':'per_page=50'}">50</option>
            <option value="{$pagination.getPageUrl(1)|replace:'page=1':'per_page=100'}">100</option>
        </select>
    </div>
</nav>
{/if}
```

### AJAX Pagination

```smarty
<!-- /admin/templates/ajax_pagination.tpl -->
<div class="ajax-pagination" 
     id="ajax-pagination"
     data-url="{$ajax_url}"
     data-container="#results-container">
    
    <div id="results-container">
        {$slot}
    </div>
    
    <div class="pagination-wrapper" {if $pagination.total_pages <= 1}style="display:none"{/if}>
        <ul class="pagination">
            <li data-page="first" class="{if !$pagination.has_previous}disabled{/if}">
                <a href="#">&laquo;</a>
            </li>
            <li data-page="prev" class="{if !$pagination.has_previous}disabled{/if}">
                <a href="#">&lsaquo;</a>
            </li>
            
            <li data-page="next" class="{if !$pagination.has_next}disabled{/if}">
                <a href="#">&rsaquo;</a>
            </li>
            <li data-page="last" class="{if !$pagination.has_next}disabled{/if}">
                <a href="#">&raquo;</a>
            </li>
        </ul>
    </div>
</div>

<script>
(function() {
    var $pagination = $('#ajax-pagination');
    var currentPage = 1;
    
    $pagination.on('click', '.pagination li:not(.disabled) a', function(e) {
        e.preventDefault();
        
        var action = $(this).closest('li').data('page');
        
        switch(action) {
            case 'first': currentPage = 1; break;
            case 'prev': currentPage--; break;
            case 'next': currentPage++; break;
            case 'last': currentPage = {$pagination.total_pages}; break;
            default: currentPage = parseInt($(this).text()); break;
        }
        
        loadPage(currentPage);
    });
    
    function loadPage(page) {
        var url = $pagination.data('url');
        var container = $pagination.data('container');
        
        $.get(url, {page: page}, function(html) {
            $(container).html(html);
            currentPage = page;
            updatePagination();
        });
    }
    
    function updatePagination() {
        $pagination.find('.pagination li').each(function() {
            var $li = $(this);
            var page = $li.data('page');
            
            if (page === 'prev') {
                $li.toggleClass('disabled', currentPage === 1);
            } else if (page === 'next') {
                $li.toggleClass('disabled', currentPage === {$pagination.total_pages});
            }
        });
    }
})();
</script>
```

## Best Practices

1. **SEO-Friendly URLs**: Use clean URL parameters
2. **Preserve Filters**: Maintain filter state in pagination
3. **Per-Page Options**: Let users choose page size
4. **Keyboard Navigation**: Support keyboard pagination
5. **AJAX Loading**: Use AJAX to avoid page reload
6. **Scroll to Top**: Scroll to top on page change
7. **Loading Indicator**: Show loading state
8. **Range Display**: Show current range (1-20 of 100)
