# WHMCS Mail Provider Module API

Complete reference for mail provider module development in WHMCS.

## Module Structure

```php
<?php
/**
 * WHMCS Mail Provider Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function yourmail_MetaData()
{
    return [
        'DisplayName' => 'Your Mail Service',
        'Description' => 'Send emails via your mail service',
    ];
}

function yourmail_ConfigArray()
{
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'Your Mail'],
        'ApiKey' => ['FriendlyName' => 'API Key', 'Type' => 'password'],
        'FromEmail' => ['FriendlyName' => 'From Email', 'Type' => 'text'],
    ];
}

function yourmail_send(array $params): array
{
    try {
        $api = new MailAPI($params['config']);
        
        $result = $api->send([
            'to' => $params['recipients'],
            'from' => $params['config']['fromemail'],
            'subject' => $params['subject'],
            'body' => $params['message'],
            'html' => $params['html'],
        ]);
        
        return ['success' => $result['success']];
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## Related Documentation

- [whmcs-functions-email.md](../functions/whmcs-functions-email.md)