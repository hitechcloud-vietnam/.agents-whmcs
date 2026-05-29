# WHMCS Admin Bulk Actions

## Overview
Guide for implementing bulk actions in WHMCS admin area. Covers batch operations, progress tracking, and undo functionality.

## Bulk Actions Implementation

### Bulk Action Handler

```php
<?php
// /includes/hooks/admin_bulk_actions.php

add_hook("AdminBulkAction", 1, function(array $params) {
    $action = $params["action"];
    $ids = $params["ids"];
    $type = $params["type"];
    
    if (empty($ids)) {
        return ["error" => "No items selected"];
    }
    
    // Validate IDs
    $ids = array_map("intval", $ids);
    
    switch ($action) {
        case "delete":
            return bulkDelete($type, $ids);
        case "update_status":
            return bulkUpdateStatus($type, $ids, $params["new_status"]);
        case "send_email":
            return bulkSendEmail($type, $ids, $params["template_id"]);
        case "export":
            return bulkExport($type, $ids);
        case "assign_group":
            return bulkAssignGroup($type, $ids, $params["group_id"]);
        default:
            return ["error" => "Unknown action"];
    }
});

function bulkDelete(string $type, array $ids): array
{
    $deleted = 0;
    $errors = [];
    
    Capsule::connection()->transaction(function() use ($type, $ids, &$deleted, &$errors) {
        foreach ($ids as $id) {
            try {
                $result = deleteRecord($type, $id);
                if ($result) {
                    $deleted++;
                }
            } catch (Exception $e) {
                $errors[] = "ID {$id}: " . $e->getMessage();
            }
        }
    });
    
    logAdminActivity("Bulk delete: {$deleted} {$type} records deleted");
    
    return [
        "success" => true,
        "deleted" => $deleted,
        "errors" => $errors
    ];
}

function bulkUpdateStatus(string $type, array $ids, string $newStatus): array
{
    $updated = 0;
    
    $tableMap = [
        "clients" => "tblclients",
        "services" => "tblhosting",
        "orders" => "tblorders",
        "invoices" => "tblinvoices"
    ];
    
    $table = $tableMap[$type] ?? null;
    
    if (!$table) {
        return ["error" => "Invalid type"];
    }
    
    // Run in batches
    $batchSize = 100;
    $batches = array_chunk($ids, $batchSize);
    
    foreach ($batches as $batch) {
        Capsule::table($table)
            ->whereIn("id", $batch)
            ->update([
                "status" => $newStatus,
                "updated_at" => date("Y-m-d H:i:s")
            ]);
        
        $updated += count($batch);
    }
    
    // Trigger hooks
    foreach ($ids as $id) {
        run_hook("AfterBulkStatusUpdate", [
            "type" => $type,
            "id" => $id,
            "new_status" => $newStatus
        ]);
    }
    
    logAdminActivity("Bulk status update: {$updated} {$type} to '{$newStatus}'");
    
    return [
        "success" => true,
        "updated" => $updated
    ];
}
```

### Bulk Email

```php
function bulkSendEmail(string $type, array $ids, int $templateId): array
{
    $template = Capsule::table("tblemailtemplates")
        ->where("id", $templateId)
        ->first();
    
    if (!$template) {
        return ["error" => "Email template not found"];
    }
    
    $sent = 0;
    $failed = 0;
    
    foreach ($ids as $id) {
        $client = getClientFromType($type, $id);
        
        if ($client) {
            $result = send_email($template->name, $client->id, []);
            
            if ($result) {
                $sent++;
            } else {
                $failed++;
            }
        }
    }
    
    logAdminActivity("Bulk email: {$sent} sent, {$failed} failed");
    
    return [
        "success" => true,
        "sent" => $sent,
        "failed" => $failed
    ];
}
```

### Progress Tracking

```php
function startBulkOperation(string $action, array $ids, array $params): string
{
    $operationId = bin2hex(random_bytes(16));
    
    Capsule::table("mod_bulk_operations")->insert([
        "operation_id" => $operationId,
        "action" => $action,
        "type" => $params["type"] ?? "",
        "total_items" => count($ids),
        "processed_items" => 0,
        "failed_items" => 0,
        "status" => "in_progress",
        "params" => json_encode($params),
        "admin_id" => $_SESSION["adminid"],
        "started_at" => date("Y-m-d H:i:s")
    ]);
    
    // Store IDs for processing
    Capsule::table("mod_bulk_operation_items")->insertUsing(
        ["operation_id", "item_id", "status"],
        array_map(fn($id) => [$operationId, $id, "pending"], $ids)
    );
    
    return $operationId;
}

function getBulkOperationProgress(string $operationId): array
{
    $operation = Capsule::table("mod_bulk_operations")
        ->where("operation_id", $operationId)
        ->first();
    
    $failed = Capsule::table("mod_bulk_operation_items")
        ->where("operation_id", $operationId)
        ->where("status", "failed")
        ->count();
    
    $processed = Capsule::table("mod_bulk_operation_items")
        ->where("operation_id", $operationId)
        ->where("status", "!=", "pending")
        ->count();
    
    $progress = $operation->total_items > 0 
        ? round(($processed / $operation->total_items) * 100) 
        : 0;
    
    return [
        "status" => $operation->status,
        "progress" => $progress,
        "processed" => $processed,
        "total" => $operation->total_items,
        "failed" => $failed,
        "errors" => getBulkOperationErrors($operationId)
    ];
}
```

## Bulk Actions Template

```smarty
<!-- /admin/templates/bulk_actions.tpl -->
<div class="bulk-actions-container">
    <div class="bulk-actions-header">
        <div class="selected-info">
            <strong><span id="selected-count">0</span></strong> items selected
        </div>
        
        <div class="bulk-actions">
            <select name="bulk_action" id="bulk-action-select" class="form-control">
                <option value="">Select Action...</option>
                <optgroup label="Actions">
                    <option value="send_email">Send Email</option>
                    <option value="export">Export Selected</option>
                </optgroup>
                <optgroup label="Status">
                    <option value="update_status:Active">Set Active</option>
                    <option value="update_status:Suspended">Set Suspended</option>
                    <option value="update_status:Closed">Set Closed</option>
                </optgroup>
                <optgroup label="Management">
                    <option value="assign_group">Assign to Group</option>
                </optgroup>
                <optgroup label="Danger Zone">
                    <option value="delete" class="text-danger">Delete Selected</option>
                </optgroup>
            </select>
            
            <button type="button" id="apply-bulk-action" class="btn btn-primary" disabled>
                Apply
            </button>
        </div>
    </div>
    
    <!-- Progress Modal -->
    <div class="modal fade" id="bulk-progress-modal">
        <div class="modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <h4 class="modal-title">Processing...</h4>
                </div>
                <div class="modal-body">
                    <div class="progress">
                        <div class="progress-bar" role="progressbar" 
                             style="width: 0%">
                            <span class="progress-text">0%</span>
                        </div>
                    </div>
                    <p class="progress-info">
                        Processing <span class="processed">0</span> of <span class="total">0</span>
                    </p>
                    <div class="errors-list" style="display:none">
                        <h5>Errors:</h5>
                        <ul></ul>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>

<script>
$('#bulk-action-select').on('change', function() {
    var action = $(this).val();
    var selected = $('.row-checkbox:checked').length;
    
    $('#apply-bulk-action').prop('disabled', !action || selected === 0);
});

$('#apply-bulk-action').on('click', function() {
    var action = $('#bulk-action-select').val();
    var ids = $('.row-checkbox:checked').map(function() {
        return $(this).val();
    }).get();
    
    if (!action) return;
    
    // Confirm for destructive actions
    if (action === 'delete') {
        if (!confirm('Are you sure you want to delete ' + ids.length + ' items?')) {
            return;
        }
    }
    
    $.post('bulk_action.php', {
        action: action,
        ids: ids,
        token: '{$token}'
    }, function(response) {
        if (response.operation_id) {
            trackProgress(response.operation_id);
        } else {
            alert('Action completed: ' + response.success);
            location.reload();
        }
    });
});

function trackProgress(operationId) {
    $('#bulk-progress-modal').modal('show');
    
    var interval = setInterval(function() {
        $.get('bulk_progress.php?id=' + operationId, function(data) {
            $('.progress-bar').css('width', data.progress + '%');
            $('.progress-text').text(data.progress + '%');
            $('.processed').text(data.processed);
            $('.total').text(data.total);
            
            if (data.status === 'completed' || data.status === 'failed') {
                clearInterval(interval);
                $('#bulk-progress-modal').modal('hide');
                location.reload();
            }
        });
    }, 1000);
}
</script>
```

## Best Practices

1. **Confirmation**: Require confirmation for destructive actions
2. **Progress**: Show progress for long operations
3. **Batch Processing**: Process in batches to avoid timeout
4. **Undo**: Provide undo for reversible actions
5. **Error Handling**: Continue on errors, report at end
6. **Logging**: Log all bulk operations
7. **Permissions**: Check permissions before bulk actions
8. **Validation**: Validate items before processing
