# WHMCS Client Upgrade Workflow

## Description
Configure and manage client account upgrade workflows in WHMCS.

## Prerequisites
- WHMCS 7.0+
- Multiple product/service tiers

## Steps

### Step 1: Configure Product Tiers
```php
<?php
// Define upgrade paths in products

$upgradePaths = [
    'basic_hosting' => ['standard_hosting', 'premium_hosting'],
    'standard_hosting' => ['premium_hosting'],
    'starter_plan' => ['business_plan', 'enterprise_plan'],
];
```

### Step 2: Handle Upgrade Request
```php
<?php
add_hook('ClientAreaPageProductDetails', 1, function($vars) {
    if (isset($_POST['upgrade_package'])) {
        $result = processUpgradeRequest($vars['service']['id'], $_POST['new_package']);
        if ($result['success']) {
            return ['success' => true, 'message' => $result['message']];
        }
        return ['error' => $result['error']];
    }
});

function processUpgradeRequest($serviceId, $newPackageId)
{
    $service = Capsule::table('tblhosting')->find($serviceId);
    $currentProduct = Capsule::table('tblproducts')->find($service->packageid);
    $newProduct = Capsule::table('tblproducts')->find($newPackageId);
    
    // Validate upgrade path
    if (!isValidUpgrade($currentProduct->id, $newProduct->id)) {
        return ['error' => 'Invalid upgrade path'];
    }
    
    // Calculate price difference
    $currentPrice = getProductPrice($currentProduct, $service->billingcycle);
    $newPrice = getProductPrice($newProduct, $service->billingcycle);
    $priceDifference = $newPrice - $currentPrice;
    
    // Calculate remaining credit
    $daysUsed = daysUsedThisCycle($service);
    $daysRemaining = 30 - $daysUsed;
    $creditAmount = ($currentPrice / 30) * $daysRemaining;
    
    // Create upgrade invoice
    $invoiceId = createUpgradeInvoice($serviceId, $newPackageId, $creditAmount);
    
    return [
        'success' => true,
        'invoice_id' => $invoiceId,
        'price_difference' => $priceDifference,
        'credit_amount' => $creditAmount,
    ];
}
```

### Step 3: Automatic Provisioning
```php
<?php
add_hook('InvoicePaid', 1, function($vars) {
    $invoice = Capsule::table('tblinvoices')->find($vars['invoiceid']);
    
    if ($invoice->type === 'upgrade') {
        processUpgradeProvisioning($invoice->service_id, $invoice->new_package_id);
    }
});

function processUpgradeProvisioning($serviceId, $newPackageId)
{
    $service = Capsule::table('tblhosting')->find($serviceId);
    
    // Update package
    Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->update(['packageid' => $newPackageId]);
    
    // Notify client
    sendEmail('ServiceUpgradeComplete', $service->userid, [
        'service_id' => $serviceId,
        'new_package' => $newPackageId,
    ]);
    
    logActivity("Service upgraded: ID $serviceId to package $newPackageId");
}
```

### Step 4: Upgrade Templates
```smarty
<div class="upgrade-options">
    <h3>Available Upgrades</h3>
    
    {foreach $availableUpgrades as $product}
        <div class="upgrade-option">
            <h4>{$product.name}</h4>
            <p>{$product.description}</p>
            <ul>
                {foreach $product.features as $feature}
                    <li>{$feature}</li>
                {/foreach}
            </ul>
            <div class="price-info">
                +{formatCurrency($product.monthly_price - $currentPrice)}/month
            </div>
            <form method="post">
                <input type="hidden" name="new_package" value="{$product.id}">
                <button type="submit" class="btn btn-primary">Upgrade Now</button>
            </form>
        </div>
    {/foreach}
</div>
```

## Tags
- upgrade
- account-change
- billing
- product-change