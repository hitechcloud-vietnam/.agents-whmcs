# WHMCS Order Form Templates

## Overview

Order form templates control the shopping cart and checkout experience in WHMCS. They include product selection, configure options, cart summary, and checkout forms.

## Template Structure

### Directory Layout

```
templates/orderforms/
    responsive/
        viewcart.tpl
        configureproduct.tpl
        configuredomain.tpl
        checkout.tpl
        login.tpl
        register.tpl
        complete.tpl
        /
    modern/
    classic/
    cards/
```

### Key Files

| File | Purpose |
|------|---------|
| `viewcart.tpl` | Shopping cart display |
| `configureproduct.tpl` | Product configuration |
| `configuredomain.tpl` | Domain configuration |
| `checkout.tpl` | Payment checkout |
| `login.tpl` | Account login/register |
| `complete.tpl` | Order confirmation |

## Cart Variables

### Cart Object

```smarty
{$cart}                   {* Full cart object *}
{$cartitems}              {* Cart items array *}
{$carttotal}              {* Total amount *}
{$carttotalrecurring}     {* Recurring total *}
{$promotioncode}          {* Applied promo code *}
{$promotiondescription}   {* Promo description *}
{$noconfiguration}        {* No product config needed *}
```

### Product in Cart

```smarty
{foreach $cartitems as $item}
    {$item.id}             {* Cart item ID *}
    {$item.productinfo}    {* Product details *}
    {$item.domain}         {* Domain name *}
    {$item.billingcycle}   {* Billing cycle *}
    {$item.amount}         {* Price *}
    {$item.recurring}      {* Recurring amount *}
    {$item.configoptions}  {* Config options *}
{/foreach}
```

## View Cart Template

### Basic Structure

```smarty
{*
    viewcart.tpl
*}
<div class="view-cart">
    <h2>Your Cart</h2>
    
    {if $cartitems}
        <table class="cart-items">
            <thead>
                <tr>
                    <th>Product</th>
                    <th>Configuration</th>
                    <th>Cycle</th>
                    <th>Price</th>
                    <th></th>
                </tr>
            </thead>
            <tbody>
                {foreach $cartitems as $item}
                    <tr>
                        <td>
                            <strong>{$item.productinfo.name}</strong>
                            {if $item.domain}
                                <br>{$item.domain}
                            {/if}
                        </td>
                        <td>
                            {foreach $item.configoptions as $option}
                                {$option.optionname}: {$option.optionvalue}
                            {/foreach}
                        </td>
                        <td>{$item.billingcycle}</td>
                        <td>
                            {if $item.recurring}
                                {$item.recurring}
                            {/if}
                            {$item.amount}
                        </td>
                        <td>
                            <a href="cart.php?a=remove&id={$item.id}">
                                Remove
                            </a>
                        </td>
                    </tr>
                {/foreach}
            </tbody>
            <tfoot>
                <tr>
                    <td colspan="3" class="text-right">
                        <strong>Total:</strong>
                    </td>
                    <td><strong>{$carttotal}</strong></td>
                    <td></td>
                </tr>
            </tfoot>
        </table>
    {else}
        <p>Your cart is empty</p>
    {/if}
</div>
```

## Product Configuration

### Configure Product Template

```smarty
{*
    configureproduct.tpl
*}
<div class="configure-product">
    <h2>Configure {$product.name}</h2>
    
    <form method="post" action="cart.php">
        <input type="hidden" name="a" value="confproduct">
        <input type="hidden" name="productid" value="{$product.gid}">
        
        <div class="product-info">
            <h3>{$product.name}</h3>
            <p>{$product.description}</p>
        </div>
        
        <div class="configuration-options">
            {if $product.configoptions}
                {foreach $product.configoptions as $configoption}
                    <div class="form-group">
                        <label>{$configoption.optionname}</label>
                        
                        {if $configoption.optiontype == 1}
                            {* Dropdown *}
                            <select name="configoption[{$configoption.id}]">
                                {foreach $configoption.options as $option}
                                    <option value="{$option.id}">
                                        {$option.name}
                                        {if $option.price > 0}
                                            (+{$option.price})
                                        {/if}
                                    </option>
                                {/foreach}
                            </select>
                            
                        {elseif $configoption.optiontype == 2}
                            {* Checkbox *}
                            <input type="checkbox" 
                                   name="configoption[{$configoption.id}]" 
                                   value="1">
                            
                        {elseif $configoption.optiontype == 3}
                            {* Quantity *}
                            <input type="number" 
                                   name="configoption[{$configoption.id}]" 
                                   value="1" min="0">
                        {/if}
                    </div>
                {/foreach}
            {/if}
        </div>
        
        <div class="pricing">
            {foreach $product.pricing as $cycle => $price}
                <label class="pricing-option">
                    <input type="radio" name="billingcycle" 
                           value="{$cycle}"{if $cycle == 'monthly'} checked{/if}>
                    {$cycle}: {$price}
                </label>
            {/foreach}
        </div>
        
        <div class="actions">
            <button type="submit" class="btn btn-primary">
                Add to Cart
            </button>
            <a href="cart.php" class="btn btn-default">
                View Cart
            </a>
        </div>
    </form>
</div>
```

## Domain Configuration

### Domain Options Template

```smarty
{*
    configuredomain.tpl
*}
<div class="domain-configure">
    <h2>Configure {$domain}</h2>
    
    <form method="post" action="cart.php">
        <input type="hidden" name="a" value="confdomains">
        <input type="hidden" name="domain" value="{$domain}">
        
        <div class="domain-registration">
            <h3>Registration Period</h3>
            <select name="regperiod">
                <option value="1">1 Year</option>
                <option value="2">2 Years (+10%)</option>
                <option value="3">3 Years (+15%)</option>
            </select>
        </div>
        
        <div class="addons">
            <h3>Add-ons</h3>
            
            <label class="addon-option">
                <input type="checkbox" name="dnsmanagement" value="1"
                       {if $domaincontact.dnsmanagement} checked{/if}>
                DNS Management (+{$dnsprice}/yr)
            </label>
            
            <label class="addon-option">
                <input type="checkbox" name="emailforwarding" value="1">
                Email Forwarding (+{$emailprice}/yr)
            </label>
            
            <label class="addon-option">
                <input type="checkbox" name="idprotection" value="1">
                WHOIS Privacy (+{$idprice}/yr)
            </label>
        </div>
        
        <button type="submit" class="btn btn-primary">
            Continue
        </button>
    </form>
</div>
```

## Checkout Template

### Basic Checkout

```smarty
{*
    checkout.tpl
*}
<div class="checkout">
    <h2>Checkout</h2>
    
    <div class="row">
        <div class="col-md-8">
            <div class="checkout-form">
                
                {if !$loggedin}
                    <div class="account-section">
                        <h3>Account</h3>
                        {include file="$template/login.tpl"}
                    </div>
                {/if}
                
                <div class="payment-section">
                    <h3>Payment Method</h3>
                    
                    {foreach $paymentmethods as $method}
                        <label class="payment-option">
                            <input type="radio" name="paymentmethod" 
                                   value="{$method.sysname}"
                                   {if $method@first} checked{/if}>
                            {$method.name}
                        </label>
                    {/foreach}
                </div>
                
                {if $accepttos}
                    <div class="tos-section">
                        <label>
                            <input type="checkbox" name="accepttos" required>
                            I agree to the <a href="tos.php">Terms of Service</a>
                        </label>
                    </div>
                {/if}
                
                <button type="submit" class="btn btn-success btn-lg">
                    Complete Order
                </button>
                
            </div>
        </div>
        
        <div class="col-md-4">
            <div class="order-summary">
                <h3>Order Summary</h3>
                {include file="$template/cartsummary.tpl"}
            </div>
        </div>
    </div>
</div>
```

## Cart Summary Component

### Summary Template

```smarty
{*
    cartsummary.tpl
*}
<div class="cart-summary">
    <table class="summary-table">
        <tbody>
            {foreach $cartitems as $item}
                <tr>
                    <td>
                        <strong>{$item.productinfo.name}</strong>
                        {if $item.domain}<br>{$item.domain}{/if}
                        <small>{$item.billingcycle}</small>
                    </td>
                    <td class="text-right">{$item.amount}</td>
                </tr>
            {/foreach}
            
            {if $promotioncode && $promotiondescription}
                <tr class="discount">
                    <td>{$promotiondescription}</td>
                    <td class="text-right">-{$promotiondiscount}</td>
                </tr>
            {/if}
        </tbody>
        <tfoot>
            <tr>
                <td><strong>Total:</strong></td>
                <td class="text-right">
                    <strong>{$carttotal}</strong>
                </td>
            </tr>
            {if $carttotalrecurring}
                <tr>
                    <td colspan="2" class="text-muted">
                        {$carttotalrecurring} {$lang.recurring}
                    </td>
                </tr>
            {/if}
        </tfoot>
    </table>
</div>
```

## Order Complete

### Success Template

```smarty
{*
    complete.tpl
*}
<div class="order-complete">
    <div class="success-icon">
        <i class="fa fa-check-circle"></i>
    </div>
    
    <h2>Thank You for Your Order!</h2>
    
    <p>Your order number is: <strong>{$orderid}</strong></p>
    
    {if $isInvoice}
        <p>
            An invoice has been generated and can be viewed 
            <a href="viewinvoice.php?id={$invoiceid}">here</a>.
        </p>
    {/if}
    
    <div class="next-steps">
        <h3>What's Next?</h3>
        <ul>
            <li>Check your email for order confirmation</li>
            <li>Complete payment if not already done</li>
            <li>Your services will be provisioned shortly</li>
        </ul>
    </div>
    
    <div class="actions">
        <a href="clientarea.php" class="btn btn-primary">
            Go to Client Area
        </a>
        <a href="cart.php" class="btn btn-default">
            Continue Shopping
        </a>
    </div>
</div>
```

## Best Practices

1. **Responsive design** - Mobile-friendly checkout
2. **Clear pricing** - Show all costs upfront
3. **Progress indicators** - Show checkout steps
4. **Form validation** - Validate before submit
5. **Promo codes** - Easy to apply and see discounts

## See Also

- [Client Area Templates](../whmcs-clientarea-templates.md)
- [Cart Templates](../whmcs-cart-templates.md)
- [Email Templates](../whmcs-email-templates.md)