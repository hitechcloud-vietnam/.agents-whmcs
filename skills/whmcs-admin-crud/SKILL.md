# WHMCS Admin CRUD Operations

## Overview
Guide for implementing CRUD (Create, Read, Update, Delete) operations in WHMCS admin area. Covers forms, data handling, and validation.

## CRUD Implementation

### Create Operation

```php
<?php
// /includes/hooks/admin_crud.php

add_hook("AdminCreateRecord", 1, function(array $params) {
    $table = $params["table"];
    $data = $params["data"];
    
    // Validate required fields
    $errors = validateRequiredFields($table, $data);
    if (!empty($errors)) {
        return ["error" => implode(", ", $errors)];
    }
    
    // Sanitize data
    $sanitized = sanitizeInput($data);
    
    // Add timestamps
    $sanitized["created_at"] = date("Y-m-d H:i:s");
    $sanitized["created_by"] = $_SESSION["adminid"];
    
    try {
        $id = Capsule::table($table)->insertGetId($sanitized);
        
        // Log action
        logAdminActivity("Created record in {$table}", $id);
        
        return ["success" => true, "id" => $id];
    } catch (Exception $e) {
        return ["error" => "Failed to create record: " . $e->getMessage()];
    }
});
```

### Read Operation

```php
function getRecords(
    string $table,
    array $filters = [],
    int $page = 1,
    int $perPage = 20,
    ?string $orderBy = null,
    string $orderDir = "asc"
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
    
    // Count total
    $total = $query->count();
    
    // Order and paginate
    $query->orderBy($orderBy ?? "id", $orderDir)
          ->offset(($page - 1) * $perPage)
          ->limit($perPage);
    
    return [
        "data" => $query->get(),
        "total" => $total,
        "page" => $page,
        "per_page" => $perPage,
        "total_pages" => ceil($total / $perPage)
    ];
}

function getRecord(string $table, int $id): ?object
{
    return Capsule::table($table)->where("id", $id)->first();
}
```

### Update Operation

```php
add_hook("AdminUpdateRecord", 1, function(array $params) {
    $table = $params["table"];
    $id = $params["id"];
    $data = $params["data"];
    
    // Check record exists
    $existing = Capsule::table($table)->where("id", $id)->first();
    if (!$existing) {
        return ["error" => "Record not found"];
    }
    
    // Validate data
    $errors = validateForUpdate($table, $data);
    if (!empty($errors)) {
        return ["error" => implode(", ", $errors)];
    }
    
    // Sanitize and add update timestamp
    $sanitized = sanitizeInput($data);
    $sanitized["updated_at"] = date("Y-m-d H:i:s");
    $sanitized["updated_by"] = $_SESSION["adminid"];
    
    try {
        Capsule::table($table)->where("id", $id)->update($sanitized);
        
        // Log action
        logAdminActivity("Updated record in {$table}", $id);
        
        return ["success" => true];
    } catch (Exception $e) {
        return ["error" => "Failed to update record: " . $e->getMessage()];
    }
});
```

### Delete Operation

```php
add_hook("AdminDeleteRecord", 1, function(array $params) {
    $table = $params["table"];
    $id = $params["id"];
    
    // Check for soft delete (mark as deleted)
    if (columnExists($table, "deleted_at")) {
        Capsule::table($table)
            ->where("id", $id)
            ->update([
                "deleted_at" => date("Y-m-d H:i:s"),
                "deleted_by" => $_SESSION["adminid"]
            ]);
        
        logAdminActivity("Soft deleted record from {$table}", $id);
    } else {
        // Hard delete
        Capsule::table($table)->where("id", $id)->delete();
        logAdminActivity("Permanently deleted record from {$table}", $id);
    }
    
    return ["success" => true];
});
```

### Validation Helper

```php
function validateRequiredFields(string $table, array $data): array
{
    $errors = [];
    $required = getRequiredFields($table);
    
    foreach ($required as $field) {
        if (!isset($data[$field]) || trim($data[$field]) === "") {
            $errors[] = ucfirst(str_replace("_", " ", $field)) . " is required";
        }
    }
    
    return $errors;
}

function validateForUpdate(string $table, array $data): array
{
    $errors = [];
    
    // Unique field validation
    $uniqueFields = getUniqueFields($table);
    foreach ($uniqueFields as $field) {
        if (isset($data[$field])) {
            $exists = Capsule::table($table)
                ->where($field, $data[$field])
                ->where("id", "!=", $data["id"])
                ->exists();
            
            if ($exists) {
                $errors[] = ucfirst($field) . " already exists";
            }
        }
    }
    
    return $errors;
}

function sanitizeInput(array $data): array
{
    $sanitized = [];
    
    foreach ($data as $key => $value) {
        // Skip system fields
        if (in_array($key, ["id", "created_at", "created_by"])) {
            continue;
        }
        
        if (is_string($value)) {
            $sanitized[$key] = htmlspecialchars(trim($value), ENT_QUOTES, "UTF-8");
        } else {
            $sanitized[$key] = $value;
        }
    }
    
    return $sanitized;
}
```

### CRUD Form Template

```smarty
<!-- /admin/templates/crud_form.tpl -->
<form method="post" action="{$form_action}" class="admin-crud-form">
    <input type="hidden" name="token" value="{$token}">
    <input type="hidden" name="id" value="{$record.id|default:''}">
    
    <div class="form-panel">
        <h3>Record Details</h3>
        
        {foreach $fields as $field}
            <div class="form-group {if $field.required}required{/if}">
                <label for="{$field.name}">
                    {$field.label}
                    {if $field.required}<span class="text-danger">*</span>{/if}
                </label>
                
                {switch $field.type}
                    {case "text"}
                        <input type="text" 
                               name="{$field.name}" 
                               id="{$field.name}"
                               value="{$record.{$field.name}|default:''}"
                               class="form-control"
                               {if $field.required}required{/if}>
                    
                    {case "textarea"}
                        <textarea name="{$field.name}" 
                                  id="{$field.name}"
                                  class="form-control"
                                  rows="4"
                                  {if $field.required}required{/if}>
                            {$record.{$field.name}|default:''}
                        </textarea>
                    
                    {case "select"}
                        <select name="{$field.name}" 
                                id="{$field.name}"
                                class="form-control"
                                {if $field.required}required{/if}>
                            <option value="">Select...</option>
                            {foreach $field.options as $option}
                                <option value="{$option.value}"
                                        {if $record.{$field.name} == $option.value}selected{/if}>
                                    {$option.label}
                                </option>
                            {/foreach}
                        </select>
                    
                    {case "checkbox"}
                        <input type="checkbox" 
                               name="{$field.name}" 
                               id="{$field.name}"
                               value="1"
                               {if $record.{$field.name}}checked{/if}>
                {/switch}
                
                {if $field.help_text}
                    <small class="help-block">{$field.help_text}</small>
                {/if}
            </div>
        {/foreach}
    </div>
    
    <div class="form-actions">
        <button type="submit" class="btn btn-primary">
            <i class="fa fa-save"></i> {$submit_label|default:'Save'}
        </button>
        <a href="{$cancel_url}" class="btn btn-default">
            Cancel
        </a>
    </div>
</form>
```

## Best Practices

1. **CSRF Protection**: Always use tokens in forms
2. **Validation**: Validate both client and server side
3. **Sanitization**: Sanitize all input data
4. **Permissions**: Check admin permissions
5. **Logging**: Log all CRUD operations
6. **Soft Delete**: Prefer soft delete for data recovery
7. **Transactions**: Use transactions for complex operations
8. **Feedback**: Provide clear success/error messages
