# WHMCS Client Review Request Workflow

## Description
Request reviews from customers after service delivery.

## Steps

### Step 1: Send Review Request
```php
<?php
function sendReviewRequest($serviceId)
{
    $service = Capsule::table('tblhosting')->find($serviceId);
    
    $token = bin2hex(random_bytes(16));
    
    Capsule::table('mod_review_requests')->insert([
        'service_id' => $serviceId,
        'client_id' => $service->userid,
        'token' => $token,
        'sent_at' => date('Y-m-d H:i:s'),
    ]);
    
    $reviewUrl = $systemUrl . '/review.php?token=' . $token;
    
    sendEmail('ReviewRequest', $service->userid, [
        'service_id' => $serviceId,
        'service_name' => $service->domain,
        'review_url' => $reviewUrl,
    ]);
}
```

### Step 2: Display Reviews
```php
<?php
function getServiceReviews($serviceId)
{
    return Capsule::table('mod_service_reviews')
        ->where('service_id', $serviceId)
        ->where('approved', 1)
        ->orderBy('created_at', 'desc')
        ->get();
}
```

## Tags
- reviews
- testimonials
- feedback
- reputation