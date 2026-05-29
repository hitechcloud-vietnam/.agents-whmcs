# WHMCS WhatsApp Notification Provider - DEVKIT

## Module Information
- **Name**: WhatsApp Notifications
- **Version**: 1.0.0
- **Type**: Notification Provider
- **Description**: Send WHMCS notifications via WhatsApp Business API

## Installation
1. Copy to `/modules/notifications/whatsapp/`
2. Activate via WHMCS Admin > Configuration > Notification Channels

## whatsapp.php
```php
<?php
/**
 * WHMCS WhatsApp Notification Provider
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function whatsapp_config()
{
    return [
        'name' => 'WhatsApp',
        'description' => 'Send notifications via WhatsApp Business API',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'access_token' => [
                'FriendlyName' => 'Access Token',
                'Type' => 'password',
                'Size' => '80',
                'Description' => 'WhatsApp Business API Access Token'
            ],
            'phone_number_id' => [
                'FriendlyName' => 'Phone Number ID',
                'Type' => 'text',
                'Size' => '50',
                'Description' => 'WhatsApp Business Phone Number ID'
            ],
            'business_account_id' => [
                'FriendlyName' => 'Business Account ID',
                'Type' => 'text',
                'Size' => '50',
                'Description' => 'Facebook Business Account ID'
            ],
            'template_namespace' => [
                'FriendlyName' => 'Template Namespace',
                'Type' => 'text',
                'Size' => '100',
                'Description' => 'Message template namespace'
            ],
            'default_template' => [
                'FriendlyName' => 'Default Template',
                'Type' => 'text',
                'Size' => '50',
                'Default' => 'whmcs_notification',
                'Description' => 'Default message template name'
            ],
            'api_version' => [
                'FriendlyName' => 'API Version',
                'Type' => 'text',
                'Size' => '10',
                'Default' => 'v18.0'
            ],
            'webhook_verify_token' => [
                'FriendlyName' => 'Webhook Verify Token',
                'Type' => 'password',
                'Size' => '50',
                'Description' => 'Token for webhook verification'
            ]
        ]
    ];
}

function whatsapp_send($params)
{
    $accessToken = $params['access_token'];
    $phoneNumberId = $params['phone_number_id'];
    $apiVersion = $params['api_version'] ?? 'v18.0';
    
    $to = $params['to'] ?? '';
    $title = $params['title'] ?? 'WHMCS Notification';
    $message = $params['message'] ?? '';
    $severity = $params['severity'] ?? 'info';
    
    if (empty($to)) {
        return ['success' => false, 'error' => 'No recipient specified'];
    }
    
    // Format phone number (ensure country code)
    $to = preg_replace('/[^0-9]/', '', $to);
    if (substr($to, 0, 1) !== '1' && strlen($to) >= 10) {
        $to = '1' . $to; // Add US country code if missing
    }
    
    // Build message content
    $headerText = $title;
    $bodyText = $message;
    
    // Prepare template message
    $templateData = [
        'messaging_product' => 'whatsapp',
        'to' => $to,
        'type' => 'template',
        'template' => [
            'name' => $params['template'] ?? $params['default_template'] ?? 'whmcs_notification',
            'language' => [
                'code' => 'en_US'
            ],
            'components' => [
                [
                    'type' => 'header',
                    'parameters' => [
                        [
                            'type' => 'text',
                            'text' => $headerText
                        ]
                    ]
                ],
                [
                    'type' => 'body',
                    'parameters' => [
                        [
                            'type' => 'text',
                            'text' => $bodyText
                        ]
                    ]
                ]
            ]
        ]
    ];
    
    // Add severity indicator if provided
    if (!empty($params['fields'])) {
        $fieldParams = [];
        foreach ($params['fields'] as $field) {
            $fieldParams[] = [
                'type' => 'text',
                'text' => ($field['title'] ?? '') . ': ' . ($field['value'] ?? '')
            ];
        }
        
        if (!empty($fieldParams)) {
            $templateData['template']['components'][] = [
                'type' => 'body',
                'parameters' => $fieldParams
            ];
        }
    }
    
    // Send via WhatsApp API
    $url = 'https://graph.facebook.com/' . $apiVersion . '/' . $phoneNumberId . '/messages';
    
    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($templateData));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'Authorization: Bearer ' . $accessToken,
        'Content-Type: application/json'
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if (isset($result['messages'][0]['id'])) {
        // Log successful send
        full_query("
            INSERT INTO " . TABLE_PREFIX . "mod_whatsapp_log 
            (recipient, message_id, status, created_at)
            VALUES (
                '" . db_escape_string($to) . "',
                '" . db_escape_string($result['messages'][0]['id']) . "',
                'sent',
                NOW()
            )
        ");
        
        return [
            'success' => true,
            'message_id' => $result['messages'][0]['id']
        ];
    }
    
    return [
        'success' => false,
        'error' => $result['error']['message'] ?? 'Failed to send notification'
    ];
}

function whatsapp_preview($params)
{
    $testPhone = $params['test_phone'] ?? '';
    
    return whatsapp_send([
        'access_token' => $params['access_token'],
        'phone_number_id' => $params['phone_number_id'],
        'template' => $params['default_template'] ?? 'whmcs_notification',
        'to' => $testPhone,
        'title' => 'Test Notification',
        'message' => 'This is a test message from WHMCS',
        'severity' => 'info'
    ]);
}

function whatsapp_callback($params)
{
    $verifyToken = $params['webhook_verify_token'] ?? '';
    
    // Webhook verification
    if ($_GET['hub_mode'] == 'subscribe' && $_GET['hub_verify_token'] == $verifyToken) {
        echo $_GET['hub_challenge'];
        exit;
    }
    
    // Handle incoming webhook
    $payload = json_decode(file_get_contents('php://input'), true);
    
    if (empty($payload['entry'])) {
        return;
    }
    
    foreach ($payload['entry'] as $entry) {
        foreach ($entry['changes'] as $change) {
            $value = $change['value'];
            
            if (isset($value['messages'])) {
                foreach ($value['messages'] as $message) {
                    whatsapp_handleIncomingMessage($message);
                }
            }
        }
    }
}

function whatsapp_handleIncomingMessage($message)
{
    $from = $message['from'];
    $body = $message['text']['body'] ?? '';
    $messageId = $message['id'];
    
    // Log incoming message
    full_query("
        INSERT INTO " . TABLE_PREFIX . "mod_whatsapp_messages 
        (from_number, message_id, body, status, received_at)
        VALUES (
            '" . db_escape_string($from) . "',
            '" . db_escape_string($messageId) . "',
            '" . db_escape_string($body) . "',
            'received',
            NOW()
        )
    ");
    
    // Process command
    $command = strtolower(trim($body));
    
    if ($command == 'status') {
        whatsapp_sendStatusUpdate($from);
    } elseif ($command == 'help') {
        whatsapp_sendHelp($from);
    }
}

function whatsapp_sendStatusUpdate($to)
{
    // Get client status
    $result = full_query("
        SELECT c.*, COUNT(h.id) as services
        FROM " . TABLE_PREFIX . "tblclients c
        LEFT JOIN " . TABLE_PREFIX . "tblhosting h ON c.id = h.userid
        WHERE c.phone = '" . db_escape_string(preg_replace('/[^0-9]/', '', $to)) . "'
        GROUP BY c.id
    ");
    
    $client = mysql_fetch_array($result);
    
    if ($client) {
        $message = "Hello " . $client['firstname'] . "!\n\n";
        $message .= "Your account summary:\n";
        $message .= "- Active Services: " . ($client['services'] ?? 0) . "\n";
        $message .= "- Credit Balance: $" . number_format($client['credit'] ?? 0, 2) . "\n\n";
        $message .= "Reply HELP for assistance.";
        
        whatsapp_send([
            'to' => $to,
            'title' => 'Status Update',
            'message' => $message
        ]);
    }
}

function whatsapp_sendHelp($to)
{
    $message = "Available commands:\n";
    $message .= "STATUS - View your account status\n";
    $message .= "HELP - Show this help message\n";
    $message .= "SUPPORT - Open a support ticket\n\n";
    $message .= "You can also visit your client area for full support.";
    
    whatsapp_send([
        'to' => $to,
        'title' => 'Help',
        'message' => $message
    ]);
}
```

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('OrderPlaced', 1, function($vars) {
    $client = full_query("SELECT phone FROM " . TABLE_PREFIX . "tblclients WHERE id = " . (int)$vars['userid']);
    $clientData = mysql_fetch_array($client);
    
    if (!empty($clientData['phone'])) {
        send_whatsapp_notification($clientData['phone'], [
            'title' => 'Order Confirmation',
            'message' => 'Thank you for your order #' . $vars['orderid'] . '. Total: $' . $vars['amount']
        ]);
    }
});

add_hook('InvoicePaid', 1, function($vars) {
    $client = full_query("SELECT phone FROM " . TABLE_PREFIX . "tblclients WHERE id = " . (int)$vars['userid']);
    $clientData = mysql_fetch_array($client);
    
    if (!empty($clientData['phone'])) {
        send_whatsapp_notification($clientData['phone'], [
            'title' => 'Payment Confirmation',
            'message' => 'Payment received for Invoice #' . $vars['invoiceid'] . '. Amount: $' . $vars['amount']
        ]);
    }
});

add_hook('ServiceCreated', 1, function($vars) {
    $client = full_query("SELECT phone FROM " . TABLE_PREFIX . "tblclients WHERE id = " . (int)$vars['userid']);
    $clientData = mysql_fetch_array($client);
    
    if (!empty($clientData['phone'])) {
        send_whatsapp_notification($clientData['phone'], [
            'title' => 'Service Activated',
            'message' => 'Your service has been activated. Domain: ' . ($vars['domain'] ?? 'N/A')
        ]);
    }
});

add_hook('ServiceSuspended', 1, function($vars) {
    $client = full_query("SELECT phone FROM " . TABLE_PREFIX . "tblclients WHERE id = " . (int)$vars['userid']);
    $clientData = mysql_fetch_array($client);
    
    if (!empty($clientData['phone'])) {
        send_whatsapp_notification($clientData['phone'], [
            'title' => 'Service Suspended',
            'message' => 'Your service has been suspended. Please contact support.'
        ]);
    }
});

add_hook('TicketOpen', 1, function($vars) {
    // Send to admin
    send_whatsapp_notification($params['admin_phone'] ?? '', [
        'title' => 'New Support Ticket',
        'message' => 'Ticket #' . $vars['ticketid'] . ': ' . ($vars['subject'] ?? 'No subject')
    ]);
});

function send_whatsapp_notification($to, $data)
{
    $config = get_config('whatsapp');
    
    if (empty($config['access_token']) || empty($config['phone_number_id'])) {
        return false;
    }
    
    return whatsapp_send([
        'access_token' => $config['access_token'],
        'phone_number_id' => $config['phone_number_id'],
        'template' => $config['default_template'] ?? 'whmcs_notification',
        'to' => $to,
        'title' => $data['title'],
        'message' => $data['message'],
        'severity' => $data['severity'] ?? 'info',
        'fields' => $data['fields'] ?? []
    ]);
}
```