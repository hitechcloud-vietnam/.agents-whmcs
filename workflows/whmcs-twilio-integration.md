# WHMCS Twilio Integration Workflow

## Overview
This workflow implements Twilio SMS integration for WHMCS.

## Prerequisites
- WHMCS with Twilio SDK
- Twilio account credentials
- Admin access

## Step-by-Step Process

### Step 1: Twilio Integration
```php
<?php
// /includes/sms/TwilioIntegration.php

class TwilioIntegration {
    private $accountSid;
    private $authToken;
    private $fromNumber;

    public function __construct()
    {
        $this->accountSid = getConfig('twilio_account_sid');
        $this->authToken = getConfig('twilio_auth_token');
        $this->fromNumber = getConfig('twilio_from_number');
    }

    /**
     * Send SMS
     */
    public function sendSMS(string $to, string $message): array
    {
        $ch = curl_init("https://api.twilio.com/2010-04-01/Accounts/{$this->accountSid}/Messages.json");

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'To' => $to,
                'From' => $this->fromNumber,
                'Body' => $message
            ]),
            CURLOPT_USERPWD => $this->accountSid . ':' . $this->authToken,
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }
}
```

### Step 2: SMS Hooks
```php
<?php
// /includes/hooks/sms_hooks.php

$twilio = new TwilioIntegration();

add_hook('InvoicePaid', 1, function($vars) use ($twilio) {
    $client = getClientsDetails($vars['userid']);

    if ($client['phonenumber']) {
        $twilio->sendSMS(
            $client['phonenumber'],
            "Payment received. Thank you!"
        );
    }
});
```

## Related Workflows
- [WHMCS SMS Automation](./whmcs-sms-automation.md)
- [WHMCS Notification Automation](./whmcs-notification-automation.md)