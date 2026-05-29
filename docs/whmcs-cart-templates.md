# WHMCS Cart Templates

## Overview

Cart templates in WHMCS control the shopping cart and checkout experience. They handle product selection, domain configuration, and payment processing.

## Template Structure

### Directory Location

```
templates/orderforms/{theme_name}/
    viewcart.tpl
    configureproduct.tpl
    configuredomain.tpl
    checkout.tpl
    login.tpl
    register.tpl
    complete.tpl
    cartsidebar.tpl
    cartindex.tpl
```

### Theme Selection

WHMCS allows selecting different cart themes from Configuration > General Settings > Ordering.

## Cart Variables

### Cart Object

```smarty
{$cart}                   {* Full cart object *}
{$cartitems}              {* Array of cart items *}
{$carttotal}              {* Total amount *}
{$carttotalrecurring}     {* Recurring total *}
{$promotioncode}          {* Applied promo code *}
{$promodescription}        {* Promo description *}
{$promotiondiscount}       {* Discount amount *}
{$taxrate}                {* Tax rate *}
{$taxtotal}               {* Tax amount *}
{$selectedcurrency}       {* Selected currency *}
```

### Cart Item Object

```smarty
{foreach $cartitems as $item}
    {$item.id}             {* Cart item ID *}
    {$item.productinfo}    {* Product info object *}
    {$item.domain}         {* Domain name *}
    {$item.domainid}       {* Domain ID *}
    {$item.billingcycle}   {* Billing cycle *}
    {$item.amount}         {* Item amount *}
    {$item.recurring}      {* Recurring amount *}
    {$item.configoptions}  {* Configurable options *}
    {$item.customfields}   {* Custom fields *}
{/foreach}
```

## View Cart Template

### Full Cart Display

```smarty
{*
    viewcart.tpl
*}
<div class="view-cart" id="view-cart">
    <h2>{$LANG.carttitle}</h2>
    
    {if $cartitems}
        <form method="post" action="cart.php?a=update">
            
            <table class="cart-items table">
                <thead>
                    <tr>
                        <th>{$LANG.cartproduct}</th>
                        <th>{$LANG.cartconfiguration}</th>
                        <th>{$LANG.cartbillingcycle}</th>
                        <th>{$LANG.cartprice}</th>
                        <th></th>
                    </tr>
                </thead>
                <tbody>
                    {foreach $cartitems as $num => $item}
                        <tr>
                            <td>
                                <strong>{$item.productinfo.name}</strong>
                                {if $item.domain}
                                    <br>
                                    <span class="domain">{$item.domain}</span>
                                {/if}
                            </td>
                            <td>
                                {if $item.configoptions}
                                    {foreach $item.configoptions as $option}
                                        <div class="config-option">
                                            {$option.optionname}: {$option.optionvalue}
                                        </div>
                                    {/foreach}
                                {/if}
                            </td>
                            <td>
                                <select name="billingcycle[{$item.id}]">
                                    <option value="monthly"{if $item.billingcycle == "monthly"} selected{/if}>
                                        Monthly
                                    </option>
                                    <option value="quarterly"{if $item.billingcycle == "quarterly"} selected{/if}>
                                        Quarterly
                                    </option>
                                    <option value="semiannually"{if $item.billingcycle == "semiannually"} selected{/if}>
                                        Semi-Annual
                                    </option>
                                    <option value="annually"{if $item.billingcycle == "annually"} selected{/if}>
                                        Annual
                                    </option>
                                </select>
                            </td>
                            <td class="price">
                                {if $item.recurring}
                                    <span class="recurring">{$item.recurring}</span>
                                {/if}
                                {$item.amount}
                            </td>
                            <td>
                                <a href="cart.php?a=remove&id={$item.id}" 
                                   class="btn btn-xs btn-danger">
                                    Remove
                                </a>
                            </td>
                        </tr>
                    {/foreach}
                </tbody>
            </table>
            
            <div class="cart-actions">
                <button type="submit" class="btn btn-default">
                    {$LANG.updatecart}
                </button>
                <a href="cart.php?a=checkout" class="btn btn-primary">
                    {$LANG.checkout}
                </a>
            </div>
            
        </form>
        
        {include file="$template/promotion.tpl"}
        
    {else}
        <div class="empty-cart">
            <p>{$LANG.cartempty}</p>
            <a href="cart.php" class="btn btn-primary">
                Continue Shopping
            </a>
        </div>
    {/if}
</div>
```

## Checkout Template

### Complete Checkout

```smarty
{*
    checkout.tpl
*}
<div class="checkout-page">
    <h1>{$LANG.checkout}</h1>
    
    <div class="row">
        <div class="col-md-8">
            
            {if !$loggedin}
                {include file="$template/login.tpl"}
            {/if}
            
            <div class="checkout-section">
                <h3>{$LANG.paymentmethod}</h3>
                
                <div class="payment-methods">
                    {foreach $paymentmethods as $method}
                        <label class="payment-option">
                            <input type="radio" 
                                   name="paymentmethod" 
                                   value="{$method.sysname}"
                                   {if $method@first} checked{/if}>
                            <span class="payment-icon">
                                <img src="{$method.icon}" alt="{$method.name}">
                            </span>
                            <span class="payment-name">{$method.name}</span>
                        </label>
                    {/foreach}
                </div>
            </div>
            
            {if $accepttos}
                <div class="checkout-section">
                    <label class="checkbox">
                        <input type="checkbox" name="accepttos" required>
                        {$LANG.ordertosagreement} 
                        <a href="tos.php" target="_blank">{$LANG.termsofservice}</a>
                    </label>
                </div>
            {/if}
            
            <form method="post" action="cart.php?a=complete">
                <input type="hidden" name="token" value="{$token}">
                
                <div class="checkout-actions">
                    <a href="cart.php" class="btn btn-default">
                        {$LANG.returncart}
                    </a>
                    <button type="submit" class="btn btn-success btn-lg">
                        {$LANG.placeorder}
                    </button>
                </div>
            </form>
            
        </div>
        
        <div class="col-md-4">
            {include file="$template/cartsummary.tpl"}
        </div>
    </div>
</div>
```

## Promotion Code

### Promo Template

```smarty
{*
    promotion.tpl
*}
<div class="promotion-section">
    <form method="post" action="cart.php?a=view">
        <div class="input-group">
            <input type="text" 
                   name="promocode" 
                   class="form-control" 
                   placeholder="{$LANG.promotioncode}"
                   value="{$promotioncode}">
            <span class="input-group-btn">
                <button type="submit" class="btn btn-default">
                    {if $promotioncode}
                        {$LANG.update}
                    {else}
                        {$LANG.apply}
                    {/if}
                </button>
            </span>
        </div>
    </form>
    
    {if $promotioncode && $promodescription}
        <div class="promo-applied">
            <span class="promo-description">{$promodescription}</span>
            <span class="promo-discount">-{$promotiondiscount}</span>
        </div>
    {/if}
    
    {if $promotionerror}
        <div class="alert alert-danger">
            {$promotionerror}
        </div>
    {/if}
</div>
```

## Cart Summary

### Sidebar Summary

```smarty
{*
    cartsummary.tpl
*}
<div class="cart-summary panel panel-default">
    <div class="panel-heading">
        <h3 class="panel-title">{$LANG.ordersummary}</h3>
    </div>
    <div class="panel-body">
        <table class="summary-table">
            {foreach $cartitems as $item}
                <tr>
                    <td>
                        <strong>{$item.productinfo.name}</strong>
                        {if $item.domain}
                            <br><small>{$item.domain}</small>
                        {/if}
                    </td>
                    <td class="text-right">{$item.amount}</td>
                </tr>
            {/foreach}
            
            {if $promotiondiscount > 0}
                <tr class="discount">
                    <td>
                        {$promodescription}
                        <a href="cart.php?a=removepromo" class="remove-promo">&times;</a>
                    </td>
                    <td class="text-right text-success">
                        -{$promotiondiscount}
                    </td>
                </tr>
            {/if}
        </table>
        
        <hr>
        
        <table class="totals-table">
            <tr class="subtotal">
                <td>{$LANG.subtotal}:</td>
                <td class="text-right">{$subtotal}</td>
            </tr>
            {if $taxtotal > 0}
                <tr class="tax">
                    <td>{$taxname} ({$taxrate}%):</td>
                    <td class="text-right">{$taxtotal}</td>
                </tr>
            {/if}
            <tr class="total">
                <td><strong>{$LANG.carttotal}:</strong></td>
                <td class="text-right">
                    <strong>{$carttotal}</strong>
                </td>
            </tr>
        </table>
        
        {if $carttotalrecurring}
            <p class="recurring-notice">
                <small>
                    {$carttotalrecurring} {$LANG.recurring}
                </small>
            </p>
        {/if}
    </div>
</div>
```

## AJAX Cart Updates

### JavaScript Integration

```javascript
// Update cart via AJAX
function updateCart(itemId, billingCycle) {
    $.post('cart.php?a=update', {
        items[itemId]: billingCycle
    }, function(response) {
        // Update cart display
        updateCartDisplay(response);
    });
}

// Remove item via AJAX
function removeItem(itemId) {
    $.post('cart.php?a=remove', {
        id: itemId
    }, function(response) {
        if (response.success) {
            // Remove row from cart
            $('#cart-item-' + itemId).fadeOut(function() {
                $(this).remove();
                updateTotals(response.totals);
            });
        }
    });
}
```

## Customization Tips

### Adding Product Comparison

```smarty
{foreach $cartitems as $item}
    <tr class="cart-item" data-item-id="{$item.id}">
        <td>
            <div class="product-compare">
                <input type="checkbox" 
                       name="compare[]" 
                       value="{$item.productinfo.gid}">
                Compare
            </div>
            {$item.productinfo.name}
        </td>
        ...
    </tr>
{/foreach}
```

### Quick Edit Configuration

```smarty
<div class="quick-config" data-item-id="{$item.id}">
    <a href="cart.php?a=confproduct&pid={$item.productinfo.gid}&id={$item.id}">
        Configure
    </a>
</div>
```

## Best Practices

1. **Responsive design** for mobile checkout
2. **Clear pricing** with all fees visible
3. **Easy promo code** application
4. **Progress indicators** for multi-step checkout
5. **AJAX updates** for better UX

## See Also

- [Order Form Templates](../whmcs-orderform-templates.md)
- [Template Functions](../whmcs-template-functions.md)
- [JavaScript Hooks](../whmcs-javascript-hooks.md)