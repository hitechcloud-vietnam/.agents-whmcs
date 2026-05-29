# WHMCS Newsletter Signup

## Concept
Newsletter subscription automation.

## Code
```php
<?php
add_hook("ClientSignup", 1, function($vars) {
    $client = getClientsDetails($vars["client_id"]);
    if ($client["newsletter"]) {
        EmailMarketing::addToCampaign($client["email"], DEFAULT_NEWSLETTER_LIST);
    }
});
```
