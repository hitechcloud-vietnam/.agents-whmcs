# WHMCS Customer Segmentation

## Concept
Customer segmentation for targeted marketing.

## Code
```php
<?php
class CustomerSegmentation {
    public static function createSegment($name, $criteria) {
        $segmentId = insert_query("tbl_customer_segments", ["name" => $name, "criteria" => json_encode($criteria)]);
        self::populateSegment($segmentId);
        return $segmentId;
    }
    
    public static function populateSegment($segmentId) {
        $segment = getSegment($segmentId);
        $clients = self::findMatchingClients($segment["criteria"]);
        foreach ($clients as $client) {
            insert_query("tbl_segment_members", ["segment_id" => $segmentId, "client_id" => $client["id"]]);
        }
    }
}
```
