# WHMCS Service Activation Workflow

## Overview
This workflow automates service activation after payment.

## Prerequisites
- WHMCS installation
- Provisioning system

## Step-by-Step Guide

### Step 1: Create Activation Hook
```php
add_hook('ServiceProvision', 1, function($vars) {
    $serviceId = $vars['serviceId'];
    $params = $vars['params'];
    
    // Activate in external system
    $api = new YourModuleAPI();
    $result = $api->activate($params);
    
    // Update WHMCS service
    \WHMCS\Database\Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->update([
            'subscription_id' => $result['subscription_id'],
            'domain' => $result['domain'],
        ]);
    
    // Send welcome email
    send_service_welcome_email($serviceId);
    
    return ['success' => true];
});
```

### Step 2: Configure Welcome Email
```php
function send_service_welcome_email(int $serviceId): void
{
    $service = \WHMCS\Database\Capsule::table('tblhosting')
        ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
        ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
        ->where('tblhosting.id', $serviceId)
        ->first();
    
    $emailTemplate = \WHMCS\Mail\Template::where('name', 'Service Welcome')->first();
    
    $parser = new TemplateParser($emailTemplate->message);
    $parser->setVariables([
        'client_name' => $service->firstname . ' ' . $service->lastname,
        'product_name' => $service->productname,
        'domain' => $service->domain,
        'username' => $service->username,
        // Don't send password in plain text!
    ]);
    
    send_email($service->email, $parser->parse(), $emailTemplate->subject);
}
```

## Service Activation Checklist

### Activation
- [ ] External system notified
- [ ] Credentials generated
- [ ] WHMCS updated

### Notification
- [ ] Welcome email sent
- [ ] Documentation linked
- [ ] Support info included
