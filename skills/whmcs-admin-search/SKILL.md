# WHMCS Admin Search

## Overview
Guide for implementing advanced search functionality in WHMCS admin area. Covers search indexing, autocomplete, and full-text search.

## Search Implementation

### Global Search

```php
<?php
// /includes/hooks/admin_search.php

add_hook("AdminGlobalSearch", 1, function(array $params) {
    $query = trim($params["q"]);
    
    if (strlen($query) < 2) {
        return ["results" => []];
    }
    
    $results = [];
    
    // Search clients
    $results["clients"] = searchClients($query);
    
    // Search services
    $results["services"] = searchServices($query);
    
    // Search orders
    $results["orders"] = searchOrders($query);
    
    // Search domains
    $results["domains"] = searchDomains($query);
    
    return [
        "results" => $results,
        "total" => count($results["clients"]) + count($results["services"]) + 
                   count($results["orders"]) + count($results["domains"])
    ];
});

function searchClients(string $query): array
{
    return Capsule::table("tblclients")
        ->select([
            "id",
            "firstname",
            "lastname",
            "email",
            "companyname",
            "status"
        ])
        ->where(function($q) use ($query) {
            $q->where("firstname", "like", "%{$query}%")
              ->orWhere("lastname", "like", "%{$query}%")
              ->orWhere("email", "like", "%{$query}%")
              ->orWhere("companyname", "like", "%{$query}%")
              ->orWhere("id", "=", (int)$query);
        })
        ->limit(10)
        ->get()
        ->toArray();
}

function searchServices(string $query): array
{
    return Capsule::table("tblhosting")
        ->join("tblclients", "tblhosting.userid", "=", "tblclients.id")
        ->join("tblproducts", "tblhosting.packageid", "=", "tblproducts.id")
        ->select([
            "tblhosting.id",
            "tblhosting.domain",
            "tblclients.email",
            "tblproducts.name as product_name",
            "tblhosting.domainstatus"
        ])
        ->where(function($q) use ($query) {
            $q->where("tblhosting.domain", "like", "%{$query}%")
              ->orWhere("tblhosting.id", "=", (int)$query);
        })
        ->limit(10)
        ->get()
        ->toArray();
}
```

### Autocomplete

```php
add_hook("AdminSearchAutocomplete", 1, function(array $params) {
    $query = trim($params["q"]);
    $type = $params["type"] ?? "all";
    
    if (strlen($query) < 2) {
        return ["suggestions" => []];
    }
    
    $suggestions = [];
    
    if ($type === "all" || $type === "clients") {
        $clients = Capsule::table("tblclients")
            ->select(["id", "firstname", "lastname", "email"])
            ->where("email", "like", "%{$query}%")
            ->orWhere("firstname", "like", "%{$query}%")
            ->orWhere("lastname", "like", "%{$query}%")
            ->limit(5)
            ->get();
        
        foreach ($clients as $client) {
            $suggestions[] = [
                "type" => "client",
                "value" => $client->email,
                "label" => $client->firstname . " " . $client->lastname,
                "subtext" => "Client #" . $client->id,
                "url" => "clientssummary.php?userid=" . $client->id
            ];
        }
    }
    
    if ($type === "all" || $type === "services") {
        $services = Capsule::table("tblhosting")
            ->select(["id", "domain"])
            ->where("domain", "like", "%{$query}%")
            ->limit(5)
            ->get();
        
        foreach ($services as $service) {
            $suggestions[] = [
                "type" => "service",
                "value" => $service->domain,
                "label" => $service->domain,
                "subtext" => "Service #" . $service->id,
                "url" => "clientsservices.php?userid=" . $service->userid . "&id=" . $service->id
            ];
        }
    }
    
    return ["suggestions" => $suggestions];
});
```

### Full-Text Search

```php
function fullTextSearch(string $query, int $limit = 50): array
{
    // Create temporary search index
    $searchTerms = explode(" ", $query);
    
    $results = [
        "clients" => [],
        "services" => [],
        "orders" => [],
        "invoices" => [],
        "tickets" => []
    ];
    
    // Client search
    foreach ($searchTerms as $term) {
        $clients = Capsule::table("tblclients")
            ->select([
                "id",
                "firstname",
                "lastname",
                "email",
                "companyname",
                "phonenumber",
                "address1",
                "city"
            ])
            ->where(function($q) use ($term) {
                $q->whereRaw(
                    "MATCH(firstname, lastname, email, companyname) AGAINST(? IN BOOLEAN MODE)",
                    [$term . "*"]
                );
            })
            ->limit($limit)
            ->get();
        
        $results["clients"] = array_merge($results["clients"], $clients->toArray());
    }
    
    // Remove duplicates
    $results["clients"] = array_unique($results["clients"], SORT_REGULAR);
    
    return $results;
}
```

## Search Template

```smarty
<!-- /admin/templates/admin_search.tpl -->
<div class="global-search" id="global-search">
    <div class="search-input-wrapper">
        <i class="fa fa-search search-icon"></i>
        <input type="text" 
               name="q" 
               id="global-search-input"
               class="form-control" 
               placeholder="Search clients, services, orders..."
               autocomplete="off">
        <span class="search-shortcut">Ctrl+K</span>
    </div>
    
    <div class="search-results" id="search-results" style="display:none;">
        <div class="results-container">
            <div class="results-loading">
                <i class="fa fa-spinner fa-spin"></i> Searching...
            </div>
        </div>
    </div>
</div>

<script>
$(function() {
    var searchTimeout;
    var $input = $('#global-search-input');
    var $results = $('#search-results');
    
    $input.on('input', function() {
        clearTimeout(searchTimeout);
        var query = $(this).val();
        
        if (query.length < 2) {
            $results.hide();
            return;
        }
        
        searchTimeout = setTimeout(function() {
            performSearch(query);
        }, 300);
    });
    
    function performSearch(query) {
        $results.show();
        
        $.get('ajax.php?action=global_search', {
            q: query
        }, function(response) {
            renderResults(response);
        });
    }
    
    function renderResults(data) {
        if (data.total === 0) {
            $results.find('.results-container').html(
                '<div class="no-results">No results found</div>'
            );
            return;
        }
        
        var html = '';
        
        if (data.results.clients.length > 0) {
            html += '<div class="result-section">' +
                    '<h4>Clients</h4>';
            
            data.results.clients.forEach(function(client) {
                html += '<a href="clientssummary.php?userid=' + client.id + '" ' +
                        'class="result-item">' +
                        '<i class="fa fa-user"></i>' +
                        '<span class="result-title">' + client.firstname + ' ' + client.lastname + '</span>' +
                        '<span class="result-subtitle">' + client.email + '</span>' +
                        '</a>';
            });
            
            html += '</div>';
        }
        
        // Similar sections for services, orders, etc.
        
        $results.find('.results-container').html(html);
    }
    
    // Keyboard shortcut
    $(document).on('keydown', function(e) {
        if ((e.ctrlKey || e.metaKey) && e.key === 'k') {
            e.preventDefault();
            $input.focus();
        }
        
        if (e.key === 'Escape') {
            $results.hide();
            $input.blur();
        }
    });
    
    // Close on click outside
    $(document).on('click', function(e) {
        if (!$(e.target).closest('#global-search').length) {
            $results.hide();
        }
    });
});
</script>

<style>
.global-search {
    position: relative;
    max-width: 400px;
}
.search-input-wrapper {
    position: relative;
}
.search-icon {
    position: absolute;
    left: 12px;
    top: 50%;
    transform: translateY(-50%);
    color: #999;
}
.search-input-wrapper input {
    padding-left: 35px;
    padding-right: 60px;
}
.search-shortcut {
    position: absolute;
    right: 10px;
    top: 50%;
    transform: translateY(-50%);
    background: #eee;
    padding: 2px 6px;
    border-radius: 3px;
    font-size: 11px;
    color: #666;
}
.search-results {
    position: absolute;
    top: 100%;
    left: 0;
    right: 0;
    background: white;
    border: 1px solid #ddd;
    border-radius: 4px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
    z-index: 1000;
    max-height: 400px;
    overflow-y: auto;
    margin-top: 5px;
}
.result-section {
    padding: 10px;
    border-bottom: 1px solid #eee;
}
.result-section h4 {
    margin: 0 0 8px 0;
    font-size: 12px;
    text-transform: uppercase;
    color: #999;
}
.result-item {
    display: flex;
    align-items: center;
    padding: 8px;
    border-radius: 4px;
    color: inherit;
}
.result-item:hover {
    background: #f5f5f5;
}
.result-item i {
    margin-right: 10px;
    color: #999;
}
.result-title {
    flex: 1;
    font-weight: 500;
}
.result-subtitle {
    color: #999;
    font-size: 12px;
}
</style>
```

## Best Practices

1. **Performance**: Use indexed columns for search
2. **Relevance**: Order results by relevance
3. **Debouncing**: Debounce search input
4. **Keyboard Navigation**: Support arrow keys and enter
5. **Caching**: Cache frequent searches
6. **Full-Text**: Use full-text search for large datasets
7. **Fuzzy Matching**: Support typo tolerance
8. **Categorized Results**: Group results by type
