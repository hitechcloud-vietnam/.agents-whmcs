# WHMCS Client Segment Email Workflow

## Description
Send targeted emails to client segments.

## Steps

### Step 1: Create Segment Campaign
```php
<?php
function sendSegmentCampaign($segmentId, $emailTemplate, $subject)
{
    $segment = Capsule::table('mod_client_segments')->find($segmentId);
    $clients = getSegmentClientsByFilter($segment->criteria);
    
    foreach ($clients as $client) {
        sendEmail($emailTemplate, $client->id, [
            'subject' => $subject,
            'segment_name' => $segment->name,
        ]);
        
        // Track sent
        Capsule::table('mod_campaign_tracking')->insert([
            'segment_id' => $segmentId,
            'client_id' => $client->id,
            'sent_at' => date('Y-m-d H:i:s'),
            'template' => $emailTemplate,
        ]);
    }
    
    return count($clients);
}
```

### Step 2: Track Campaign Performance
```php
<?php
function getCampaignStats($campaignId)
{
    $sent = Capsule::table('mod_campaign_tracking')
        ->where('campaign_id', $campaignId)
        ->count();
    
    $opened = Capsule::table('mod_email_tracking')
        ->where('campaign_id', $campaignId)
        ->where('opened', 1)
        ->count();
    
    $clicked = Capsule::table('mod_email_tracking')
        ->where('campaign_id', $campaignId)
        ->where('clicked', 1)
        ->count();
    
    return [
        'sent' => $sent,
        'opened' => $opened,
        'clicked' => $clicked,
        'open_rate' => $sent > 0 ? ($opened / $sent) * 100 : 0,
        'click_rate' => $sent > 0 ? ($clicked / $sent) * 100 : 0,
    ];
}
```

## Tags
- email-marketing
- segment-email
- campaign
- automation