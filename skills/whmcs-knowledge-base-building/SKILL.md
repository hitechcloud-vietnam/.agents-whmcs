# WHMCS Knowledge Base Building

## Concept
Creating and maintaining self-service knowledge base articles.

## Code
```php
<?php
class KBManager {
    public static function createArticle($data) {
        $articleId = insert_query("tblknowledgebase", [
            "title" => $data["title"],
            "article" => $data["content"],
            "categoryid" => $data["category_id"],
            "author" => $data["author_id"],
            "created" => date("Y-m-d H:i:s"),
            "updated" => date("Y-m-d H:i:s"),
            "published" => $data["published"] ?? 0,
            "views" => 0
        ]);
        
        return $articleId;
    }
    
    public static function trackArticleView($articleId) {
        full_query("UPDATE tblknowledgebase SET views = views + 1 WHERE id = " . (int)$articleId);
    }
}
```
