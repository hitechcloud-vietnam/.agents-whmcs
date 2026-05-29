# WHMCS Mail Provider Builder

## Concept

Mail providers allow WHMCS to send emails through custom email services like SendGrid, Mailgun, Amazon SES, etc. They replace the default PHP mail functionality.

## File Structure

```
/modules/mail/
├── yourmailprovider/
│   ├── yourmailprovider.php    # Mail provider
│   └── templates/
│       └── footer.tpl
```

## Core Mail Provider

```php
<?php
/**
 * Mail Provider: Your Mail Provider
 * Version: 1.0.0
 * Description: Send emails via Your Mail Provider
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Mail\MailProviderInterface;

if (interface_exists('WHMCS\Mail\MailProviderInterface')) {
    class YourMailProvider implements MailProviderInterface
    {
        private $apiKey;
        private $fromEmail;
        private $fromName;
        
        public function __construct(array $config)
        {
            $this->apiKey = $config['api_key'] ?? '';
            $this->fromEmail = $config['from_email'] ?? '';
            $this->fromName = $config['from_name'] ?? 'WHMCS';
        }
        
        public function setFromEmail($email)
        {
            $this->fromEmail = $email;
        }
        
        public function setFromName($name)
        {
            $this->fromName = $name;
        }
        
        public function setSubject($subject)
        {
            $this->subject = $subject;
        }
        
        public function setBody($body)
        {
            $this->body = $body;
        }
        
        public function addRecipient($email, $name = '')
        {
            $this->recipients[] = [
                'email' => $email,
                'name' => $name,
            ];
        }
        
        public function addCc($email, $name = '')
        {
            $this->cc[] = [
                'email' => $email,
                'name' => $name,
            ];
        }
        
        public function addBcc($email, $name = '')
        {
            $this->bcc[] = [
                'email' => $email,
                'name' => $name,
            ];
        }
        
        public function addAttachment($path, $filename = '', $mimeType = '')
        {
            $this->attachments[] = [
                'path' => $path,
                'filename' => $filename,
                'mimeType' => $mimeType,
            ];
        }
        
        public function send()
        {
            if (empty($this->apiKey)) {
                throw new \Exception('API key not configured');
            }
            
            $payload = $this->buildPayload();
            
            return $this->makeRequest($payload);
        }
        
        protected function buildPayload()
        {
            $payload = [
                'from' => [
                    'email' => $this->fromEmail,
                    'name' => $this->fromName,
                ],
                'subject' => $this->subject ?? '',
                'content' => [
                    [
                        'type' => 'text/html',
                        'value' => $this->body ?? '',
                    ],
                ],
                'personalizations' => [],
            ];
            
            // Add recipients
            foreach ($this->recipients as $recipient) {
                $personalization = [
                    'to' => [
                        [
                            'email' => $recipient['email'],
                            'name' => $recipient['name'],
                        ],
                    ],
                ];
                
                if (!empty($this->cc)) {
                    $personalization['cc'] = [];
                    foreach ($this->cc as $cc) {
                        $personalization['cc'][] = [
                            'email' => $cc['email'],
                            'name' => $cc['name'],
                        ];
                    }
                }
                
                if (!empty($this->bcc)) {
                    $personalization['bcc'] = [];
                    foreach ($this->bcc as $bcc) {
                        $personalization['bcc'][] = [
                            'email' => $bcc['email'],
                            'name' => $bcc['name'],
                        ];
                    }
                }
                
                $payload['personalizations'][] = $personalization;
            }
            
            // Add attachments
            if (!empty($this->attachments)) {
                $payload['attachments'] = [];
                foreach ($this->attachments as $attachment) {
                    $payload['attachments'][] = [
                        'content' => base64_encode(file_get_contents($attachment['path'])),
                        'filename' => $attachment['filename'] ?? basename($attachment['path']),
                        'type' => $attachment['mimeType'] ?? 'application/octet-stream',
                    ];
                }
            }
            
            return $payload;
        }
        
        protected function makeRequest(array $payload)
        {
            $ch = curl_init('https://api.yourmailprovider.com/v1/send');
            
            curl_setopt_array($ch, [
                CURLOPT_POST => true,
                CURLOPT_POSTFIELDS => json_encode($payload),
                CURLOPT_RETURNTRANSFER => true,
                CURLOPT_HTTPHEADER => [
                    'Authorization: Bearer ' . $this->apiKey,
                    'Content-Type: application/json',
                ],
                CURLOPT_TIMEOUT => 30,
            ]);
            
            $response = curl_exec($ch);
            $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            curl_close($ch);
            
            if ($httpCode >= 400) {
                $error = json_decode($response, true);
                throw new \Exception('Failed to send email: ' . ($error['message'] ?? 'Unknown error'));
            }
            
            return true;
        }
    }
}
```

## Configuration

```php
function yourmailprovider_config()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Mail Provider',
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'Your mail provider API key',
        ],
        'from_email' => [
            'FriendlyName' => 'Default From Email',
            'Type' => 'text',
            'Size' => '50',
            'Description' => 'Default sender email address',
        ],
        'from_name' => [
            'FriendlyName' => 'Default From Name',
            'Type' => 'text',
            'Size' => '50',
            'Default' => 'WHMCS',
        ],
    ];
}
```

## SendGrid Implementation

```php
<?php
class SendGridMailProvider implements MailProviderInterface
{
    private $apiKey;
    private $fromEmail = '';
    private $fromName = '';
    private $subject = '';
    private $body = '';
    private $recipients = [];
    private $cc = [];
    private $bcc = [];
    private $attachments = [];
    
    public function __construct(array $config)
    {
        $this->apiKey = $config['api_key'] ?? '';
        $this->fromEmail = $config['from_email'] ?? '';
        $this->fromName = $config['from_name'] ?? 'WHMCS';
    }
    
    public function send()
    {
        $payload = [
            'personalizations' => [],
            'from' => [
                'email' => $this->fromEmail,
                'name' => $this->fromName,
            ],
            'subject' => $this->subject,
            'content' => [
                [
                    'type' => 'text/html',
                    'value' => $this->body,
                ],
            ],
        ];
        
        foreach ($this->recipients as $recipient) {
            $personalization = [
                'to' => [['email' => $recipient['email'], 'name' => $recipient['name']]],
            ];
            
            if (!empty($this->cc)) {
                $personalization['cc'] = [];
                foreach ($this->cc as $cc) {
                    $personalization['cc'][] = ['email' => $cc['email'], 'name' => $cc['name']];
                }
            }
            
            $payload['personalizations'][] = $personalization;
        }
        
        // Add attachments
        if (!empty($this->attachments)) {
            $payload['attachments'] = [];
            foreach ($this->attachments as $att) {
                $payload['attachments'][] = [
                    'content' => base64_encode(file_get_contents($att['path'])),
                    'filename' => $att['filename'],
                    'type' => $att['mimeType'],
                ];
            }
        }
        
        $ch = curl_init('https://api.sendgrid.com/v3/mail/send');
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);
        
        curl_exec($ch);
        curl_close($ch);
        
        return true;
    }
    
    // ... other methods
}
```

## Step-by-Step Implementation

1. Create mail provider directory
2. Implement MailProviderInterface
3. Define configuration options
4. Implement send method
5. Build email payload
6. Add attachment support
7. Handle CC/BCC
8. Test email delivery

## Implementation Checklist

- [ ] Create provider directory in modules/mail
- [ ] Implement MailProviderInterface
- [ ] Define configuration options
- [ ] Implement setFrom methods
- [ ] Implement addRecipient
- [ ] Implement addCc/addBcc
- [ ] Implement addAttachment
- [ ] Implement buildPayload
- [ ] Implement makeRequest
- [ ] Test email delivery