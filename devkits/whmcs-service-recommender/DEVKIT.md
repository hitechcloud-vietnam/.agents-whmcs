# WHMCS Service Recommender Module - DEVKIT

## Module Information
- **Name**: Service Recommender
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: AI-powered product recommendations for customers

## Installation
1. Copy to `/modules/addons/service_recommender/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('ClientAreaPage', 1, function($vars) {
    $userId = $_SESSION['uid'] ?? 0;
    return ['recommendations' => ServiceRecommender::getRecommendations($userId)];
});

add_hook('OrderCompleted', 1, function($vars) {
    ServiceRecommender::recordPurchase($vars);
});
```

### includes/ServiceRecommender.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class ServiceRecommender
{
    private static $table = 'mod_service_recommender_log';
    
    public static function getRecommendations($userId, $limit = 5)
    {
        if (!$userId) {
            return self::getPopularProducts();
        }
        
        // Get user history
        $userProducts = self::getUserProducts($userId);
        $userCategories = self::getUserCategories($userId);
        
        // Get collaborative filtering recommendations
        $collaborative = self::getCollaborativeRecommendations($userId, $limit);
        
        // Get content-based recommendations
        $contentBased = self::getContentBasedRecommendations($userCategories, $userProducts, $limit);
        
        // Combine and rank
        return self::rankRecommendations($collaborative, $contentBased, $limit);
    }
    
    private static function getUserProducts($userId)
    {
        $result = full_query("
            SELECT DISTINCT packageid FROM " . TABLE_PREFIX . "tblorders
            WHERE userid = " . (int)$userId . " AND status IN ('Active', 'Suspended')
        ");
        
        $products = [];
        while ($row = mysql_fetch_array($result)) {
            $products[] = $row['packageid'];
        }
        
        return $products;
    }
    
    private static function getUserCategories($userId)
    {
        $result = full_query("
            SELECT DISTINCT p.gid
            FROM " . TABLE_PREFIX . "tblorders o
            JOIN " . TABLE_PREFIX . "tblproducts p ON o.packageid = p.id
            WHERE o.userid = " . (int)$userId . " AND o.status = 'Active'
        ");
        
        $categories = [];
        while ($row = mysql_fetch_array($result)) {
            $categories[] = $row['gid'];
        }
        
        return $categories;
    }
    
    private static function getCollaborativeRecommendations($userId, $limit)
    {
        // Find similar users based on purchase history
        $result = full_query("
            SELECT o2.packageid, COUNT(*) as score
            FROM " . TABLE_PREFIX . "tblorders o1
            JOIN " . TABLE_PREFIX . "tblorders o2 ON o1.packageid = o2.packageid
            WHERE o1.userid = " . (int)$userId . "
            AND o2.userid != " . (int)$userId . "
            AND o1.status IN ('Active', 'Suspended')
            AND o2.status IN ('Active', 'Suspended')
            AND o2.packageid NOT IN (SELECT packageid FROM " . TABLE_PREFIX . "tblorders WHERE userid = " . (int)$userId . ")
            GROUP BY o2.packageid
            ORDER BY score DESC
            LIMIT " . (int)$limit
        ");
        
        $recommendations = [];
        while ($row = mysql_fetch_array($result)) {
            $recommendations[$row['packageid']] = ['score' => $row['score'], 'type' => 'collaborative'];
        }
        
        return $recommendations;
    }
    
    private static function getContentBasedRecommendations($categories, $excludeProducts, $limit)
    {
        if (empty($categories)) {
            return [];
        }
        
        $excludeList = implode(',', array_map('intval', $excludeProducts));
        $categoryList = implode(',', array_map('intval', $categories));
        
        $result = full_query("
            SELECT p.id, p.name, p.description, p.monthly, COUNT(o.id) as popularity
            FROM " . TABLE_PREFIX . "tblproducts p
            LEFT JOIN " . TABLE_PREFIX . "tblorders o ON p.id = o.packageid
            WHERE p.gid IN (" . $categoryList . ")
            AND p.hidden = 0
            AND p.id NOT IN (" . ($excludeList ?: '0') . ")
            GROUP BY p.id
            ORDER BY popularity DESC, p.name ASC
            LIMIT " . (int)$limit
        );
        
        $recommendations = [];
        while ($row = mysql_fetch_array($result)) {
            $recommendations[$row['id']] = [
                'name' => $row['name'],
                'description' => substr($row['description'], 0, 200),
                'price' => $row['monthly'],
                'type' => 'content'
            ];
        }
        
        return $recommendations;
    }
    
    private static function rankRecommendations($collaborative, $contentBased, $limit)
    {
        $all = array_merge($collaborative, $contentBased);
        
        // Sort by score/type priority
        uasort($all, function($a, $b) {
            $scoreA = $a['score'] ?? 0;
            $scoreB = $b['score'] ?? 0;
            return $scoreB - $scoreA;
        });
        
        return array_slice($all, 0, $limit, true);
    }
    
    private static function getPopularProducts()
    {
        $result = full_query("
            SELECT p.id, p.name, p.description, p.monthly, COUNT(o.id) as order_count
            FROM " . TABLE_PREFIX . "tblproducts p
            LEFT JOIN " . TABLE_PREFIX . "tblorders o ON p.id = o.packageid
            WHERE p.hidden = 0
            GROUP BY p.id
            ORDER BY order_count DESC
            LIMIT 5
        ");
        
        $products = [];
        while ($row = mysql_fetch_array($result)) {
            $products[] = [
                'id' => $row['id'],
                'name' => $row['name'],
                'description' => substr($row['description'], 0, 200),
                'price' => $row['monthly']
            ];
        }
        
        return $products;
    }
    
    public static function recordPurchase($vars)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$table . "
            (user_id, product_id, order_id, recommended_from, created_at)
            VALUES (
                " . (int)($vars['userid'] ?? 0) . ",
                " . (int)($vars['pid'] ?? 0) . ",
                " . (int)($vars['orderid'] ?? 0) . ",
                '" . db_escape_string($vars['recommended_from'] ?? 'unknown') . "',
                NOW()
            )
        ");
    }
    
    public static function getRecommendationStats()
    {
        $result = full_query("
            SELECT 
                recommended_from,
                COUNT(*) as total,
                SUM(CASE WHEN order_id > 0 THEN 1 ELSE 0 END) as converted
            FROM " . TABLE_PREFIX . self::$table . "
            GROUP BY recommended_from
        ");
        
        $stats = [];
        while ($row = mysql_fetch_array($result)) {
            $stats[] = [
                'source' => $row['recommended_from'],
                'impressions' => $row['total'],
                'conversions' => $row['converted'],
                'rate' => $row['total'] > 0 ? round($row['converted'] / $row['total'] * 100, 2) : 0
            ];
        }
        
        return $stats;
    }
}

function service_recommender_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_service_recommender_log (
            id INT AUTO_INCREMENT PRIMARY KEY,
            user_id INT NOT NULL,
            product_id INT NOT NULL,
            order_id INT,
            recommended_from VARCHAR(50),
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("CREATE INDEX idx_user ON " . TABLE_PREFIX . "mod_service_recommender_log(user_id)");
    
    return ['status' => 'success', 'description' => 'Service Recommender activated'];
}

function service_recommender_deactivate()
{
    return ['status' => 'success', 'description' => 'Service Recommender deactivated'];
}

function service_recommender_config()
{
    return [
        'name' => 'Service Recommender',
        'description' => 'AI-powered product recommendations',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'max_recommendations' => [
                'Type' => 'text',
                'FriendlyName' => 'Max Recommendations',
                'Default' => '5'
            ],
            'collaborative_weight' => [
                'Type' => 'text',
                'FriendlyName' => 'Collaborative Weight',
                'Default' => '0.6'
            ]
        ]
    ];
}
```