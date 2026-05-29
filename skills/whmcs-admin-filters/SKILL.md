# WHMCS Admin Filters

## Overview
Guide for implementing advanced filtering in WHMCS admin area. Covers filter builders, presets, and dynamic queries.

## Filter System

### Filter Builder

```php
<?php
// /includes/hooks/admin_filters.php

class FilterBuilder {
    private array $filters = [];
    private string $table;
    private array $joins = [];
    
    public function __construct(string $table) {
        $this->table = $table;
    }
    
    public function addFilter(string $field, string $operator, $value, string $type = "and"): self {
        $this->filters[] = [
            "field" => $field,
            "operator" => $operator,
            "value" => $value,
            "type" => $type
        ];
        return $this;
    }
    
    public function addJoin(string $table, string $on, string $type = "left"): self {
        $this->joins[] = [
            "table" => $table,
            "on" => $on,
            "type" => $type
        ];
        return $this;
    }
    
    public function apply($query): mixed {
        foreach ($this->joins as $join) {
            $query->join($join["table"], $join["on"], $join["type"]);
        }
        
        foreach ($this->filters as $index => $filter) {
            $method = $filter["type"] === "or" ? "orWhere" : "where";
            
            switch ($filter["operator"]) {
                case "=":
                case "!=":
                case ">":
                case "<":
                case ">=":
                case "<=":
                    $query->$method($filter["field"], $filter["operator"], $filter["value"]);
                    break;
                    
                case "like":
                    $query->$method($filter["field"], "like", "%" . $filter["value"] . "%");
                    break;
                    
                case "in":
                    $query->$method(function($q) use ($filter) {
                        $q->whereIn($filter["field"], $filter["value"]);
                    });
                    break;
                    
                case "not_in":
                    $query->$method(function($q) use ($filter) {
                        $q->whereNotIn($filter["field"], $filter["value"]);
                    });
                    break;
                    
                case "between":
                    $query->$method(function($q) use ($filter) {
                        $q->whereBetween($filter["field"], $filter["value"]);
                    });
                    break;
                    
                case "null":
                    $query->$method($filter["field"], "IS", null);
                    break;
                    
                case "not_null":
                    $query->$method($filter["field"], "IS NOT", null);
                    break;
            }
        }
        
        return $query;
    }
    
    public function count(): int {
        return count($this->filters);
    }
    
    public function isEmpty(): bool {
        return empty($this->filters);
    }
}
```

### Advanced Filter Hook

```php
add_hook("AdminBuildFilter", 1, function(array $params) {
    $filterBuilder = new FilterBuilder($params["table"]);
    
    // Parse filter params
    foreach ($params["filter_params"] as $key => $value) {
        if ($value === null || $value === "") continue;
        
        switch ($key) {
            case "search":
                $filterBuilder->addFilter(
                    $params["search_field"] ?? "name",
                    "like",
                    $value
                );
                break;
                
            case "status":
                if (is_array($value)) {
                    $filterBuilder->addFilter("status", "in", $value);
                } else {
                    $filterBuilder->addFilter("status", "=", $value);
                }
                break;
                
            case "date_from":
                $filterBuilder->addFilter("created_at", ">=", $value);
                break;
                
            case "date_to":
                $filterBuilder->addFilter("created_at", "<=", $value . " 23:59:59");
                break;
                
            case "amount_min":
                $filterBuilder->addFilter("total", ">=", (float)$value);
                break;
                
            case "amount_max":
                $filterBuilder->addFilter("total", "<=", (float)$value);
                break;
        }
    }
    
    return ["filter_builder" => $filterBuilder];
});
```

### Filter Presets

```php
function getFilterPresets(string $type): array
{
    return [
        "clients" => [
            [
                "name" => "Active Clients",
                "filters" => ["status" => "Active"]
            ],
            [
                "name" => "New This Month",
                "filters" => [
                    "date_from" => date("Y-m-01"),
                    "date_to" => date("Y-m-t")
                ]
            ],
            [
                "name" => "High Value",
                "filters" => ["total_spent_min" => 1000]
            ],
            [
                "name" => "Needs Attention",
                "filters" => ["status" => ["Suspended", "Inactive"]]
            ]
        ],
        "invoices" => [
            [
                "name" => "Overdue",
                "filters" => [
                    "status" => "Unpaid",
                    "due_before" => date("Y-m-d")
                ]
            ],
            [
                "name" => "This Month",
                "filters" => [
                    "date_from" => date("Y-m-01"),
                    "date_to" => date("Y-m-t")
                ]
            ],
            [
                "name" => "Draft",
                "filters" => ["status" => "Draft"]
            ]
        ]
    ];
}
```

## Filter Template

```smarty
<!-- /admin/templates/admin_filters.tpl -->
<div class="filter-panel" id="filter-panel">
    <div class="filter-header">
        <h4>
            <i class="fa fa-filter"></i>
            Filters
            {if $active_filters}
                <span class="badge">{$active_filters_count}</span>
            {/if}
        </h4>
        <button type="button" class="btn-toggle" data-target="filter-panel">
            <i class="fa fa-chevron-{if $filters_visible}up{else}down{/if}"></i>
        </button>
    </div>
    
    <div class="filter-body" {if !$filters_visible}style="display:none"{/if}>
        <form method="get" action="{$filter_url}" class="filter-form">
            <div class="filter-row">
                <div class="filter-group">
                    <label>Search</label>
                    <div class="input-group">
                        <input type="text" name="search" 
                               value="{$filters.search}" 
                               class="form-control"
                               placeholder="Search...">
                        <select name="search_field" class="form-control">
                            <option value="name">Name</option>
                            <option value="email">Email</option>
                            <option value="id">ID</option>
                        </select>
                    </div>
                </div>
                
                <div class="filter-group">
                    <label>Status</label>
                    <select name="status" class="form-control">
                        <option value="">All</option>
                        <option value="Active" {if $filters.status eq 'Active'}selected{/if}>Active</option>
                        <option value="Inactive" {if $filters.status eq 'Inactive'}selected{/if}>Inactive</option>
                        <option value="Suspended" {if $filters.status eq 'Suspended'}selected{/if}>Suspended</option>
                        <option value="Closed" {if $filters.status eq 'Closed'}selected{/if}>Closed</option>
                    </select>
                </div>
                
                <div class="filter-group">
                    <label>Date Range</label>
                    <div class="input-daterange input-group">
                        <input type="date" name="date_from" 
                               value="{$filters.date_from}"
                               class="form-control" placeholder="From">
                        <input type="date" name="date_to" 
                               value="{$filters.date_to}"
                               class="form-control" placeholder="To">
                    </div>
                </div>
            </div>
            
            {if $presets}
                <div class="filter-presets">
                    <label>Quick Filters:</label>
                    {foreach $presets as $preset}
                        <a href="?{http_build_query(array_merge($current_params, $preset.filters))}"
                           class="btn btn-xs {if $preset.active}btn-primary{else}btn-default{/if}">
                            {$preset.name}
                        </a>
                    {/foreach}
                </div>
            {/if}
            
            <div class="filter-actions">
                <button type="submit" class="btn btn-primary btn-sm">
                    Apply Filters
                </button>
                <a href="{$clear_url}" class="btn btn-default btn-sm">
                    Clear All
                </a>
            </div>
        </form>
    </div>
</div>
```

## Best Practices

1. **URL Persistence**: Keep filters in URL for bookmarking
2. **Presets**: Provide common filter presets
3. **Dynamic Queries**: Build queries dynamically
4. **Performance**: Index filtered columns
5. **Clear Feedback**: Show active filter count
6. **Reset**: Easy filter reset
7. **Date Ranges**: Support relative date ranges
8. **Multiple Values**: Allow multiple values per filter
