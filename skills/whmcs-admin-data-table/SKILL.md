# WHMCS Admin DataTable Integration

## Overview
Guide for integrating DataTables in WHMCS admin area. Covers DataTables configuration, server-side processing, and custom features.

## DataTables Integration

### Basic Setup

```php
<?php
// /includes/hooks/admin_datatables.php

add_hook("AdminDataTableAssets", 1, function(array $params) {
    return [
        "css" => [
            "/assets/css/dataTables.bootstrap.min.css"
        ],
        "js" => [
            "/assets/js/jquery.dataTables.min.js",
            "/assets/js/dataTables.bootstrap.min.js"
        ]
    ];
});
```

### Server-Side Processing

```php
add_hook("AdminDataTableServerSide", 1, function(array $params) {
    $table = $params["table"];
    $primaryKey = $params["primary_key"] ?? "id";
    
    // Build query
    $query = Capsule::table($table);
    
    // Total records
    $recordsTotal = $query->count();
    
    // Filtered records
    $query = applyDataTableFilters($query, $params);
    $recordsFiltered = $query->count();
    
    // Order
    $query = applyDataTableOrder($query, $params);
    
    // Pagination
    $query->offset($params["start"])
          ->limit($params["length"]);
    
    // Get data
    $data = $query->get();
    
    // Transform data
    $data = transformDataTableData($data, $params);
    
    return [
        "draw" => (int)$params["draw"],
        "recordsTotal" => $recordsTotal,
        "recordsFiltered" => $recordsFiltered,
        "data" => $data
    ];
});

function applyDataTableFilters($query, array $params): mixed
{
    // Column filters
    if (isset($params["columns"]) && is_array($params["columns"])) {
        foreach ($params["columns"] as $index => $column) {
            if (!empty($column["search"]["value"])) {
                $field = $column["data"];
                $value = $column["search"]["value"];
                $searchable = $column["searchable"] ?? true;
                
                if ($searchable === "true") {
                    $query->where($field, "like", "%{$value}%");
                }
            }
        }
    }
    
    // Global search
    if (!empty($params["search"]["value"])) {
        $search = $params["search"]["value"];
        $query->where(function($q) use ($search, $params) {
            foreach ($params["searchable_fields"] as $field) {
                $q->orWhere($field, "like", "%{$search}%");
            }
        });
    }
    
    return $query;
}

function applyDataTableOrder($query, array $params): mixed
{
    if (isset($params["order"]) && is_array($params["order"])) {
        foreach ($params["order"] as $order) {
            $columnIndex = (int)$order["column"];
            $direction = $order["dir"];
            $columnName = $params["columns"][$columnIndex]["data"];
            
            $query->orderBy($columnName, $direction);
        }
    }
    
    return $query;
}
```

### DataTable Template

```smarty
<!-- /admin/templates/datatables_example.tpl -->
<table id="data-table" class="table table-hover">
    <thead>
        <tr>
            <th>
                <input type="checkbox" id="select-all">
            </th>
            <th>ID</th>
            <th>Name</th>
            <th>Email</th>
            <th>Status</th>
            <th>Created</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody></tbody>
</table>

<script>
$(document).ready(function() {
    var table = $('#data-table').DataTable({
        processing: true,
        serverSide: true,
        ajax: {
            url: 'ajax.php?action=list_records',
            type: 'POST',
            data: function(d) {
                d.token = '{$token}';
            }
        },
        columns: [
            {
                data: 'id',
                render: function(data) {
                    return '<input type="checkbox" class="row-checkbox" value="' + data + '">';
                },
                orderable: false,
                searchable: false
            },
            { data: 'id' },
            { data: 'name' },
            { data: 'email' },
            { 
                data: 'status',
                render: function(data) {
                    return '<span class="label label-' + data.toLowerCase() + '">' + data + '</span>';
                }
            },
            { 
                data: 'created_at',
                render: function(data) {
                    return moment(data).format('YYYY-MM-DD HH:mm');
                }
            },
            {
                data: 'id',
                render: function(data, type, row) {
                    return '<a href="edit.php?id=' + data + '" class="btn btn-xs btn-default">Edit</a>' +
                           '<a href="delete.php?id=' + data + '" class="btn btn-xs btn-danger">Delete</a>';
                },
                orderable: false
            }
        ],
        order: [[1, 'desc']],
        pageLength: 25,
        lengthMenu: [[10, 25, 50, 100], [10, 25, 50, 100]],
        language: {
            search: "Search:",
            lengthMenu: "Show _MENU_ entries",
            info: "Showing _START_ to _END_ of _TOTAL_ entries",
            paginate: {
                first: "First",
                last: "Last",
                next: "Next",
                previous: "Previous"
            }
        },
        dom: "<'row'<'col-sm-6'l><'col-sm-6'f>>" +
             "<'row'<'col-sm-12'tr>>" +
             "<'row'<'col-sm-5'i><'col-sm-7'p>>"
    });
    
    // Select all checkbox
    $('#select-all').on('click', function() {
        $('.row-checkbox').prop('checked', this.checked);
    });
    
    // Bulk actions
    $('.bulk-action').on('click', function() {
        var action = $(this).data('action');
        var ids = $('.row-checkbox:checked').map(function() {
            return $(this).val();
        }).get();
        
        if (ids.length === 0) {
            alert('Please select at least one record');
            return;
        }
        
        // Handle bulk action
        $.post('bulk_action.php', {
            action: action,
            ids: ids,
            token: '{$token}'
        }, function(response) {
            table.ajax.reload();
        });
    });
});
</script>
```

### Custom Column Rendering

```php
function transformDataTableData(array $data, array $params): array
{
    return array_map(function($row) use ($params) {
        // Add action buttons
        $row["actions"] = [
            "edit" => [
                "url" => "edit.php?id=" . $row["id"],
                "icon" => "fa-edit",
                "label" => "Edit"
            ],
            "delete" => [
                "url" => "delete.php?id=" . $row["id"],
                "icon" => "fa-trash",
                "label" => "Delete",
                "class" => "text-danger"
            ]
        ];
        
        // Format dates
        if (isset($row["created_at"])) {
            $row["created_at_formatted"] = date("Y-m-d H:i", strtotime($row["created_at"]));
        }
        
        // Add status badge
        if (isset($row["status"])) {
            $row["status_badge"] = '<span class="label label-' . 
                strtolower($row["status"]) . '">' . $row["status"] . '</span>';
        }
        
        return $row;
    }, $data);
}
```

## Best Practices

1. **Server-Side**: Use server-side processing for large datasets
2. **Indexing**: Ensure database columns are indexed
3. **Performance**: Minimize query complexity
4. **Security**: Use prepared statements
5. **UX**: Show loading indicators
6. **Responsive**: Use responsive extension
7. **Export**: Add export buttons (CSV, Excel)
8. **State**: Allow saving table state
