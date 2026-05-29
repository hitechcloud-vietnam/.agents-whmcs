# WHMCS AWS Integration Workflow

## Overview
This workflow implements AWS services integration for WHMCS.

## Prerequisites
- WHMCS with AWS SDK
- AWS account credentials
- Admin access

## Step-by-Step Process

### Step 1: AWS Integration
```php
<?php
// /includes/integrations/AWSIntegration.php

class AWSIntegration {
    private $awsKey;
    private $awsSecret;
    private $region;

    public function __construct()
    {
        $this->awsKey = getConfig('aws_access_key');
        $this->awsSecret = getConfig('aws_secret_key');
        $this->region = getConfig('aws_region') ?? 'us-east-1';
    }

    /**
     * Upload to S3
     */
    public function uploadToS3(string $bucket, string $key, string $data, string $acl = 'private'): array
    {
        $ch = curl_init("https://{$bucket}.s3.{$this->region}.amazonaws.com/{$key}");

        curl_setopt_array($ch, [
            CURLOPT_PUT => true,
            CURLOPT_POSTFIELDS => $data,
            CURLOPT_HTTPHEADER => [
                'Authorization: AWS4-HMAC-SHA256 Credential=' . $this->awsKey,
                'Content-Type: application/octet-stream',
                'x-amz-acl' => $acl
            ],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return [
            'success' => $httpCode === 200,
            'http_code' => $httpCode
        ];
    }

    /**
     * Send SNS notification
     */
    public function sendSNS(string $topicArn, string $message, string $subject = ''): array
    {
        $ch = curl_init('https://sns.' . $this->region . '.amazonaws.com/');

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'Action' => 'Publish',
                'TopicArn' => $topicArn,
                'Message' => $message,
                'Subject' => $subject,
                'Version' => '2010-03-31'
            ]),
            CURLOPT_HTTPHEADER => [
                'Authorization: AWS4-HMAC-SHA256 Credential=' . $this->awsKey
            ],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }
}
```

## Related Workflows
- [WHMCS Google Cloud Integration](./whmcs-google-cloud-integration.md)
- [WHMCS Azure Integration](./whmcs-azure-integration.md)