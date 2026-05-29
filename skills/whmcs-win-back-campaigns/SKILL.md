# WHMCS Win-back Campaigns

## Concept
Campaign automation for lapsed customers.

## Code
```php
<?php
class WinBackCampaigns {
    public static function createWinBackCampaign($name, $conditions) {
        $segmentId = CustomerSegmentation::createSegment($name, $conditions);
        
        // Add email sequence
        addEmailSequence($segmentId, ["day_1" => "win_back_1", "day_7" => "win_back_2", "day_14" => "win_back_special"]);
        
        return $segmentId;
    }
}
```
