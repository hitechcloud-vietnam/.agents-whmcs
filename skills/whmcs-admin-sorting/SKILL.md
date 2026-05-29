# WHMCS Admin Sorting

## Overview
Guide for implementing column sorting in WHMCS admin area. Covers sortable tables, multi-column sorting, and sort preferences.

## Sorting Implementation

### Sort Helper

```php
<?php
// /includes/hooks/admin_sorting.php

class SortHelper {
    private array $columns = [];
    private array $currentSort = [];
    private array $sortable = [];
    
    public function __construct(array $columns, ?array $currentSort = null) {
        $this->columns = $columns;
        $this->sortable = array_keys($columns);
        
        if ($currentSort) {
            $this->currentSort = $currentSort;
        } else {
            $this->parseSortFromUrl();
        }
    }
    
    private function parseSortFromUrl(): void {
        $sort = $_GET["sort"] ?? "";
        $order = $_GET["order"] ?? "asc";
        
        if ($sort && in_array($sort, $this->sortable)) {
            $this->currentSort = [
                "column" => $sort,
                "order" => strtolower($order) === "desc" ? "desc" : "asc"
            ];
        }
    }
    
    public function apply($query): mixed {
        if (empty($this->currentSort)) {
            return $query->orderBy("id", "desc");
        }
        
        $column = $this->currentSort["column"];
        $order = $this->currentSort["order"];
        
        // Check if column needs special handling
        if (isset($this->columns[$column]["relation"])) {
            return $query
                ->join(
                    $this->columns[$column]["relation"]["table"],
                    $this->columns[$column]["relation"]["on"][0],
                    $this->columns[$column]["relation"]["on"][1]
                )
                ->orderBy($column, $order);
        }
        
        return $query->orderBy($column, $order);
    }
    
    public function getSortUrl(string $column): string {
        if (!in_array($column, $this->sortable)) {
            return "#";
        }
        
        $params = $_GET;
        
        if ($this->currentSort["column"] === $column) {
            // Toggle order
            $params["order"] = $this->currentSort["order"] === "asc" ? "desc" : "asc";
        } else {
            $params["sort"] = $column;
            $params["order"] = "asc";
        }
        
        return "?" . http_build_query($params);
    }
    
    public function getSortClass(string $column): string {
        if ($this->currentSort["column"] !== $column) {
            return "sortable";
        }
        
        return "sortable " . ($this->currentSort["order"] === "asc" ? "sort-asc" : "sort-desc");
    }
    
    public function toArray(): array {
        return $this->currentSort;
    }
}
```

### Sortable Table

```php
add_hook("AdminSortableTable", 1, function(array $params) {
    $columns = [
        "id" => ["label" => "ID"],
        "name" => ["label" => "Name"],
        "email" => ["label" => "Email"],
        "status" => ["label" => "Status"],
        "created_at" => ["label" => "Created"],
        "total" => ["label" => "Total"]
    ];
    
    $sortHelper = new SortHelper($columns);
    
    $query = Capsule::table("tblorders");
    $sortHelper->apply($query);
    
    $data = $query->limit(20)->get();
    
    return [
        "data" => $data,
        "sort" => $sortHelper->toArray()
    ];
});
```

## Sorting Template

```smarty
<!-- /admin/templates/sortable_table.tpl -->
<table class="table sortable-table">
    <thead>
        <tr>
            <th class="{$sort.getSortClass('id')}">
                <a href="{$sort.getSortUrl('id')}">
                    ID
                    <i class="fa fa-sort"></i>
                </a>
            </th>
            <th class="{$sort.getSortClass('name')}">
                <a href="{$sort.getSortUrl('name')}">
                    Name
                    <i class="fa fa-sort"></i>
                </a>
            </th>
            <th class="{$sort.getSortClass('email')}">
                <a href="{$sort.getSortUrl('email')}">
                    Email
                    <i class="fa fa-sort"></i>
                </a>
            </th>
            <th class="{$sort.getSortClass('status')}">
                <a href="{$sort.getSortUrl('status')}">
                    Status
                    <i class="fa fa-sort"></i>
                </a>
            </th>
            <th class="{$sort.getSortClass('created_at')}">
                <a href="{$sort.getSortUrl('created_at')}">
                    Created
                    <i class="fa fa-sort"></i>
                </a>
            </th>
            <th class="{$sort.getSortClass('total')}">
                <a href="{$sort.getSortUrl('total')}">
                    Total
                    <i class="fa fa-sort"></i>
                </a>
            </th>
        </tr>
    </thead>
    <tbody>
        {foreach $data as $row}
            <tr>
                <td>{$row.id}</td>
                <td>{$row.name}</td>
                <td>{$row.email}</td>
                <td>
                    <span class="label label-{$row.status|strtolower}">
                        {$row.status}
                    </span>
                </td>
                <td>{$row.created_at|date_format}</td>
                <td>{$row.total|formatCurrency}</td>
            </tr>
        {/foreach}
    </tbody>
</table>

<style>
.sortable a {
    color: inherit;
    text-decoration: none;
}
.sortable a:hover {
    text-decoration: underline;
}
.sortable a i {
    margin-left: 5px;
    opacity: 0.3;
}
.sort-asc a i,
.sort-desc a i {
    opacity: 1;
}
.sort-asc a i:before {
    content: "\f0de"; /* fa-sort-asc */
}
.sort-desc a i:before {
    content: "\f0dd"; /* fa-sort-desc */
}
</style>
```

## Best Practices

1. **Visual Indicators**: Show sort direction with icons
2. **Default Sort**: Set sensible default sort column
3. **URL Persistence**: Keep sort state in URL
4. **Index Optimization**: Ensure sorted columns are indexed
5. **Multi-Sort**: Support sorting by multiple columns
6. **Preserve Filters**: Maintain filter state with sort
7. **Click Feedback**: Show loading state during sort
8. **Consistent UX**: Use standard sort icons
