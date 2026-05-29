# WHMCS Email Marketing

## Concept
Email campaign integration and automation.

## Code
```php
<?php
class EmailMarketing {
    public static function addToCampaign($email, $listId, $tags = []) {
        return addToEmailList($email, $listId, $tags);
    }
    
    public static function trackCampaignPerformance($campaignId) {
        return ["opens" => getEmailOpens($campaignId), "clicks" => getEmailClicks($campaignId)];
    }
}
```
