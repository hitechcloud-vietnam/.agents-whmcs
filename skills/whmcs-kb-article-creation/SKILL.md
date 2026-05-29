# WHMCS KB Article Creation

## Concept
Systematic approach to creating effective knowledge base articles.

## Code
```php
<?php
class KBArticleCreator {
    public static function createArticle($data) {
        return insert_query("tblknowledgebase", [
            "title" => $data["title"], "article" => $data["content"],
            "categoryid" => $data["category_id"]
        ]);
    }
}
```
