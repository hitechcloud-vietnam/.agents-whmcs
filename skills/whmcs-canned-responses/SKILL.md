# WHMCS Canned Responses

## Concept
Pre-built reply templates for common support scenarios.

## Code
```php
<?php
class CannedResponses {
    public static function getResponses($category = null) {
        $query = "SELECT * FROM tblcannedresponses";
        if ($category) $query .= " WHERE category =  . e() . ";
        
        return full_query($query . " ORDER BY name");
    }
    
    public static function insertResponse($name, $content, $category = "General") {
        return insert_query("tblcannedresponses", [
            "name" => $name, "content" => $content, "category" => $category
        ]);
    }
}
```
