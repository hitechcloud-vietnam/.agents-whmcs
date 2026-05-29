# WHMCS Client Notes

## Overview
Master skill for managing client notes in WHMCS. Covers note creation, searching, and internal communications.

## Client Notes Helper

```php
<?php
// /includes/helpers/notes_helper.php

class ClientNotes
{
    public function addNote(int $clientId, int $adminId, string $content, string $type = "general"): int
    {
        return \Illuminate\Database\Capsule\Manager::table("mod_client_notes")
            ->insertGetId([
                "client_id" => $clientId,
                "admin_id" => $adminId,
                "content" => $content,
                "type" => $type,
                "created_at" => date("Y-m-d H:i:s"),
            ]);
    }
    
    public function getNotes(int $clientId, string $type = null): array
    {
        $query = \Illuminate\Database\Capsule\Manager::table("mod_client_notes")
            ->where("client_id", $clientId)
            ->orderBy("created_at", "desc");
        
        if ($type) {
            $query->where("type", $type);
        }
        
        return $query->get()->toArray();
    }
    
    public function searchNotes(string $term): array
    {
        return \Illuminate\Database\Capsule\Manager::table("mod_client_notes")
            ->where("content", "like", "%{$term}%")
            ->orderBy("created_at", "desc")
            ->limit(50)
            ->get()
            ->toArray();
    }
    
    public function updateNote(int $noteId, string $content): bool
    {
        return \Illuminate\Database\Capsule\Manager::table("mod_client_notes")
            ->where("id", $noteId)
            ->update(["content" => $content, "updated_at" => date("Y-m-d H:i:s")]) > 0;
    }
    
    public function deleteNote(int $noteId): bool
    {
        return \Illuminate\Database\Capsule\Manager::table("mod_client_notes")
            ->where("id", $noteId)
            ->delete() > 0;
    }
}
```

## Best Practices

1. **Organization**: Use note types for categorization
2. **Privacy**: Notes are internal only
3. **Search**: Make notes searchable
4. **Timestamps**: Track when notes were added
5. **Permissions**: Control who can view notes
6. **Formatting**: Support rich text formatting
7. **Audit Trail**: Log note changes
8. **Attachments**: Support file attachments
