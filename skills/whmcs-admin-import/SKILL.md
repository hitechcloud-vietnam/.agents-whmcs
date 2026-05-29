# WHMCS Admin Import

## Overview
Guide for implementing CSV import functionality in WHMCS admin area. Covers file upload, parsing, validation, and batch import.

## CSV Import System

### Import Controller

```php
<?php
// /includes/hooks/admin_import.php

add_hook("AdminImportHandler", 1, function(array $params) {
    $type = $params["type"];
    $file = $_FILES["import_file"];
    
    // Validate file
    if ($file["error"] !== UPLOAD_ERR_OK) {
        return ["error" => "File upload failed"];
    }
    
    if ($file["type"] !== "text/csv" && !preg_match("/\.csv$/", $file["name"])) {
        return ["error" => "Please upload a CSV file"];
    }
    
    // Parse CSV
    $rows = parseCSV($file["tmp_name"]);
    if (empty($rows)) {
        return ["error" => "CSV file is empty"];
    }
    
    // Validate headers
    $headers = array_shift($rows);
    $requiredHeaders = getRequiredHeaders($type);
    
    $missingHeaders = array_diff($requiredHeaders, $headers);
    if (!empty($missingHeaders)) {
        return [
            "error" => "Missing required columns: " . implode(", ", $missingHeaders)
        ];
    }
    
    // Validate rows
    $errors = [];
    $validRows = [];
    
    foreach ($rows as $index => $row) {
        $rowNumber = $index + 2; // +2 for header and 0-index
        $rowData = array_combine($headers, $row);
        
        $rowErrors = validateImportRow($type, $rowData);
        if (!empty($rowErrors)) {
            $errors[] = "Row {$rowNumber}: " . implode("; ", $rowErrors);
        } else {
            $validRows[] = $rowData;
        }
    }
    
    return [
        "success" => true,
        "headers" => $headers,
        "total_rows" => count($rows),
        "valid_rows" => count($validRows),
        "errors" => $errors,
        "preview" => array_slice($validRows, 0, 10)
    ];
});

function parseCSV(string $filePath): array
{
    $rows = [];
    $handle = fopen($filePath, "r");
    
    if ($handle === false) {
        return [];
    }
    
    // Detect delimiter
    $firstLine = fgets($handle);
    rewind($handle);
    
    $delimiter = ",";
    if (substr_count($firstLine, ";") > substr_count($firstLine, ",")) {
        $delimiter = ";";
    }
    
    while (($row = fgetcsv($handle, 0, $delimiter)) !== false) {
        // Clean up values
        $row = array_map("trim", $row);
        if (!empty(array_filter($row))) {
            $rows[] = $row;
        }
    }
    
    fclose($handle);
    return $rows;
}
```

### Import Validation

```php
function validateImportRow(string $type, array $row): array
{
    $errors = [];
    
    switch ($type) {
        case "clients":
            $errors = validateClientRow($row);
            break;
        case "products":
            $errors = validateProductRow($row);
            break;
        case "services":
            $errors = validateServiceRow($row);
            break;
    }
    
    return $errors;
}

function validateClientRow(array $row): array
{
    $errors = [];
    
    // Required fields
    if (empty($row["email"])) {
        $errors[] = "Email is required";
    } elseif (!filter_var($row["email"], FILTER_VALIDATE_EMAIL)) {
        $errors[] = "Invalid email format";
    } elseif (emailExists($row["email"])) {
        $errors[] = "Email already exists";
    }
    
    if (empty($row["firstname"])) {
        $errors[] = "First name is required";
    }
    
    if (empty($row["lastname"])) {
        $errors[] = "Last name is required";
    }
    
    // Optional field validation
    if (!empty($row["phonenumber"]) && !preg_match("/^[\d\s\-\+\(\)]+$/", $row["phonenumber"])) {
        $errors[] = "Invalid phone number";
    }
    
    return $errors;
}
```

### Batch Import

```php
add_hook("AdminProcessImport", 1, function(array $params) {
    $type = $params["type"];
    $rows = $params["rows"];
    $updateExisting = $params["update_existing"] ?? false;
    
    $imported = 0;
    $updated = 0;
    $failed = 0;
    $errors = [];
    
    foreach ($rows as $index => $row) {
        $result = processImportRow($type, $row, $updateExisting);
        
        if ($result["success"]) {
            if ($result["action"] === "created") {
                $imported++;
            } else {
                $updated++;
            }
        } else {
            $failed++;
            $errors[] = "Row " . ($index + 1) . ": " . $result["error"];
        }
    }
    
    // Log import
    logAdminActivity("Imported {$imported} records, updated {$updated}, failed {$failed}", 0);
    
    return [
        "imported" => $imported,
        "updated" => $updated,
        "failed" => $failed,
        "errors" => $errors
    ];
});

function processImportRow(string $type, array $row, bool $updateExisting): array
{
    switch ($type) {
        case "clients":
            return importClient($row, $updateExisting);
        case "products":
            return importProduct($row, $updateExisting);
        case "services":
            return importService($row, $updateExisting);
    }
    
    return ["success" => false, "error" => "Unknown import type"];
}

function importClient(array $row, bool $updateExisting): array
{
    // Check if client exists
    $existing = Capsule::table("tblclients")
        ->where("email", $row["email"])
        ->first();
    
    if ($existing && !$updateExisting) {
        return ["success" => false, "error" => "Client already exists"];
    }
    
    $clientData = [
        "firstname" => $row["firstname"],
        "lastname" => $row["lastname"],
        "email" => $row["email"],
        "companyname" => $row["companyname"] ?? "",
        "phonenumber" => $row["phonenumber"] ?? "",
        "address1" => $row["address1"] ?? "",
        "city" => $row["city"] ?? "",
        "state" => $row["state"] ?? "",
        "postcode" => $row["postcode"] ?? "",
        "country" => $row["country"] ?? "",
        "datecreated" => date("Y-m-d")
    ];
    
    if ($existing && $updateExisting) {
        Capsule::table("tblclients")
            ->where("id", $existing->id)
            ->update($clientData);
        
        return ["success" => true, "action" => "updated", "id" => $existing->id];
    }
    
    // Generate password
    $password = bin2hex(random_bytes(8));
    $clientData["password"] = createPasswordHash($password);
    
    $id = Capsule::table("tblclients")->insertGetId($clientData);
    
    return ["success" => true, "action" => "created", "id" => $id];
}
```

## Import Template

```smarty
<!-- /admin/templates/import_csv.tpl -->
<div class="import-container">
    <h2>Import {$import_type|ucfirst}s</h2>
    
    <div class="import-steps">
        <div class="step {if $step eq 1}active{elseif $step gt 1}completed{/if}">
            <span class="step-number">1</span>
            <span class="step-label">Upload File</span>
        </div>
        <div class="step {if $step eq 2}active{elseif $step gt 2}completed{/if}">
            <span class="step-number">2</span>
            <span class="step-label">Validate Data</span>
        </div>
        <div class="step {if $step eq 3}active{elseif $step gt 3}completed{/if}">
            <span class="step-number">3</span>
            <span class="step-label">Review</span>
        </div>
        <div class="step {if $step eq 4}active{/if}">
            <span class="step-number">4</span>
            <span class="step-label">Complete</span>
        </div>
    </div>
    
    <form method="post" enctype="multipart/form-data" action="import.php">
        <input type="hidden" name="token" value="{$token}">
        <input type="hidden" name="type" value="{$import_type}">
        <input type="hidden" name="step" value="{$step}">
        
        <div class="import-form">
            <div class="form-group">
                <label for="import_file">CSV File</label>
                <input type="file" name="import_file" id="import_file" 
                       accept=".csv" required class="form-control">
                <small class="help-text">
                    Maximum file size: 10MB
                </small>
            </div>
            
            <div class="form-group">
                <label>
                    <input type="checkbox" name="update_existing" value="1">
                    Update existing records
                </label>
            </div>
            
            <div class="form-group">
                <label>
                    <input type="checkbox" name="has_headers" value="1" checked>
                    First row contains headers
                </label>
            </div>
            
            <button type="submit" class="btn btn-primary">
                Continue
            </button>
        </div>
    </form>
    
    {if $step >= 2}
        <div class="import-results">
            <h3>Validation Results</h3>
            <div class="stats">
                <div class="stat">
                    <strong>{$total_rows}</strong> Total Rows
                </div>
                <div class="stat success">
                    <strong>{$valid_rows}</strong> Valid
                </div>
                <div class="stat danger">
                    <strong>{$error_count}</strong> Errors
                </div>
            </div>
            
            {if $errors}
                <div class="errors">
                    <h4>Errors</h4>
                    <ul>
                        {foreach $errors as $error}
                            <li>{$error}</li>
                        {/foreach}
                    </ul>
                </div>
            {/if}
        </div>
    {/if}
</div>
```

## Best Practices

1. **File Size**: Limit maximum file size
2. **Encoding**: Handle different encodings (UTF-8, UTF-16)
3. **Delimiter**: Auto-detect CSV delimiter
4. **Preview**: Show preview before import
5. **Validation**: Validate all rows before import
6. **Transactions**: Use transactions for batch imports
7. **Progress**: Show import progress
8. **Rollback**: Support rollback on failure
