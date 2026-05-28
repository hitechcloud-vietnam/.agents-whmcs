# WHMCS Email Template Variables

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `smarty-template-reference`, `internationalization-guide`

## Overview

WHMCS uses Smarty template engine for email notifications. Variables are automatically populated based on context and can be combined with modifiers and functions.

## Global Variables

These variables are available in all email templates:

```smarty
{$company_name}           {* Company/Brand name *}
{$company_logo}            {* Company logo URL *}
{$company_address}         {* Full company address *}
{$company_url}             {* Company website URL *}
{$company_logo_url}        {* Logo image URL *}
{$date}                    {* Current date *}
{$time}                    {* Current time *}
{$timestamp}               {* Unix timestamp *}
{$email_verification_link} {* Email verification URL *}
{$whmcs_url}               {* WHMCS installation URL *}
{$currency_prefix}         {* Currency symbol (prefix) *}
{$currency_suffix}         {* Currency symbol (suffix) *}
```

## Client Variables

Available in client-related templates:

```smarty
{$client_first_name}           {* Client first name *}
{$client_last_name}            {* Client last name *}
{$client_full_name}            {* Full client name *}
{$client_email}                {* Client email address *}
{$client_company_name}         {* Company name *}
{$client_phone}                {* Phone number *}
{$client_password}             {* Account password (for new accounts) *}
{$client_password_random}      {* Random password *}
{$client_last_login}           {* Last login date/time *}
{$client_status}               {* Account status *}
{$client_id}                   {* Client ID *}
{$client_tax_id}               {* Tax ID / VAT number *}
{$client_credit}               {* Account credit amount *}
{$client_country}               {* Country code *}
{$client_state}                {* State/Province *}
{$client_city}                 {* City *}
{$client_postcode}             {* Postal code *}
{$client_address1}             {* Address line 1 *}
{$client_address2}             {* Address line 2 *}
{$client_language}             {* Preferred language *}
{$client_customfields}         {* Array of custom fields *}
{$client_customfield_1}        {* Specific custom field by ID *}
```

## Invoice Variables

```smarty
{$invoice_num}                 {* Invoice number *}
{$invoice_id}                 {* Invoice ID *}
{$invoice_date}               {* Invoice date *}
{$invoice_due_date}           {* Due date *}
{$invoice_date_created}       {* Creation date *}
{$invoice_amount}             {* Total amount *}
{$invoice_total}              {* Invoice total *}
{$invoice_subtotal}           {* Subtotal before tax *}
{$invoice_tax}                {* Tax amount *}
{$invoice_tax_rate}           {* Tax rate percentage *}
{$invoice_balance}           {* Outstanding balance *}
{$invoice_amount_paid}       {* Amount paid *}
{$invoice_status}            {* Paid, Unpaid, Cancelled, etc. *}
{$invoice_payment_url}       {* Online payment link *}
{$invoice_notes}             {* Invoice notes *}
{$invoice_items}             {* Array of line items *}

{* Line item structure *}
{foreach $invoice_items as $item}
    {$item.description}       {* Item description *}
    {$item.amount}           {* Item amount *}
    {$item.taxed}            {* Whether taxed *}
    {$item.type}             {* Item type (Hosting, Domain, etc.) *}
{/foreach}
```

### Invoice Related Items

```smarty
{$invoice_items_num}          {* Number of items *}
{$invoice_is_infinite}        {* Infinite billing flag *}
```

## Order Variables

```smarty
{$order_id}                   {* Order ID *}
{$order_num}                  {* Order number *}
{$order_date}                 {* Order date *}
{$order_status}              {* Order status *}
{$order_total}               {* Order total *}
{$order_payment_method}      {* Payment method *}
{$order_auto_setup}          {* Auto setup enabled *}
{$order_nameservers}         {* Name servers array *}
{$order_transient_id}        {* Transient order ID *}
```

## Product/Service Variables

```smarty
{$service_id}                 {* Service ID *}
{$service_domain}            {* Service domain *}
{$service_order_date}        {* Order date *}
{$service_first_payment}    {* First payment amount *}
{$service_recurring_amount} {* Recurring amount *}
{$service_billing_cycle}    {* Billing cycle *}
{$service_next_due_date}     {* Next due date *}
{$service_next_invoice_date} {* Next invoice date *}
{$service_termination_date} {* Termination date *}
{$service_suspension_date}   {* Suspension date *}
{$service_status}            {* Service status *}
{$service_username}          {* Service username *}
{$service_password}          {* Service password *}
{$service_prefix}            {* Service prefix *}
{$service_disk_usage}        {* Disk usage *}
{$service_disk_limit}        {* Disk limit *}
{$service_bw_usage}          {* Bandwidth usage *}
{$service_bw_limit}          {* Bandwidth limit *}
{$service_last_updated}      {* Last update date *}
{$service_subscription_id}   {* Subscription ID *}
{$service_custom_fields}     {* Service custom fields *}
{$service_config_options}    {* Configurable options *}
{$service_days_remaining}    {* Days until renewal *}
```

## Domain Variables

```smarty
{$domain_id}                 {* Domain ID *}
{$domain_name}               {* Domain name *}
{$domain_registration_date} {* Registration date *}
{$domain_next_due_date}      {* Next due date *}
{$domain_expiry_date}        {* Expiry date *}
{$domain_renewal_date}       {* Renewal date *}
{$domain_registrar}          {* Registrar name *}
{$domain_registration_period} {* Registration years *}
{$domain_status}            {* Domain status *}
{$domain_dns_management}     {* DNS management enabled *}
{$domain_email_forwarding}  {* Email forwarding enabled *}
{$domain_id_protection}     {* ID protection enabled *}
{$domain_epp_code}           {* EPP/Transfer code *}
{$domain_days_remaining}     {* Days until expiry *}
{$domain_transfer_secret}  {* Transfer authorization code *}
```

## Support Ticket Variables

```smarty
{$ticket_id}                 {* Ticket ID *}
{$ticket_number}             {* Ticket number *}
{$ticket_subject}           {* Ticket subject *}
{$ticket_status}            {* Ticket status *}
{$ticket_priority}          {* Priority level *}
{$ticket_message}           {* Initial message *}
{$ticket_reply}             {* Reply message *}
{$ticket_department}         {* Department name *}
{$ticket_created_date}       {* Creation date *}
{$ticket_updated_date}       {* Last update date *}
{$ticket_url}               {* Ticket URL *}
{$ticket_owner_name}        {* Assigned admin name *}
{$ticket_owner_email}       {* Assigned admin email *}
{$ticket_attachments}       {* Array of attachments *}
```

## Affiliate Variables

```smarty
{$affiliate_id}              {* Affiliate ID *}
{$affiliate_referral_url}   {* Referral URL *}
{$affiliate_pending_commission} {* Pending commission *}
{$affiliate_available_commission} {* Available commission *}
{$affiliate_withdrawn}       {* Withdrawn amount *}
{$affiliate_referrals}       {* Total referrals *}
{$affiliate_visitors}        {* Total visitors *}
{$affiliate_conversion_rate} {* Conversion percentage *}
```

## Admin Variables

```smarty
{$admin_user}               {* Admin username *}
{$admin_name}               {* Admin full name *}
{$admin_email}              {* Admin email *}
{$admin_edit_url}           {* Admin edit URL *}
{$admin_url}                {* Admin area URL *}
```

## Custom Fields

Access custom fields using dot notation:

```smarty
{* Client custom field *}
{$client_customfields.cf_field_name}

{* Service custom field *}
{$service_customfields.cf_specific_field}

{* Domain custom field *}
{$domain_customfields.cf_domain_field}

{* Specific custom field by ID *}
{$custom_field_1}
{$custom_field_2}
```

## Configurable Options

```smarty
{$configoptions}            {* Array of all config options *}

{foreach $configoptions as $option}
    {$option.name}           {* Option name *}
    {$option.value}          {* Selected value *}
    {$option.option}         {* Option type *}
    {$option.selectedvalue} {* Selected option value *}
    {$option.selectedqty}    {* Selected quantity *}
    {$option.recurring}      {* Recurring modifier *}
{/foreach}
```

## Payment Gateway Variables

```smarty
{$payment_method}           {* Payment method name *}
{$payment_method_nice}      {* Display name *}
{$transaction_id}           {* Transaction ID *}
{$transaction_amount}       {* Transaction amount *}
{$transaction_fee}          {* Transaction fee *}
{$transaction_currency}     {* Transaction currency *}
{$transaction_date}         {* Transaction date *}
{$transaction_status}       {* Transaction status *}
```

## Domain Registration Details

```smarty
{$domain_reg_first_name}   {* Registrant first name *}
{$domain_reg_last_name}    {* Registrant last name *}
{$domain_reg_company_name} {* Registrant company *}
{$domain_reg_email}        {* Registrant email *}
{$domain_reg_address1}     {* Registrant address *}
{$domain_reg_address2}     {* Registrant address 2 *}
{$domain_reg_city}         {* Registrant city *}
{$domain_reg_state}        {* Registrant state *}
{$domain_reg_postcode}     {* Registrant postcode *}
{$domain_reg_country}     {* Registrant country *}
{$domain_reg_phonenumber}  {* Registrant phone *}
```

## Conditional Logic

```smarty
{* If client has credit *}
{if $client_credit > 0}
    Your available credit: {$client_credit}
{/if}

{* Show custom field if set *}
{if $client_customfields.cf_vip_tier}
    VIP Tier: {$client_customfields.cf_vip_tier}
{/if}

{* Invoice status check *}
{if $invoice_status eq 'Unpaid'}
    <a href="{$invoice_payment_url}">Pay Now</a>
{/if}
```

## Modifiers and Functions

```smarty
{* Date formatting *}
{$invoice_due_date|date_format:"%d %B %Y"}

{* Currency formatting *}
{$invoice_total|string_format:"%.2f"}

{* Truncate text *}
{$ticket_subject|truncate:50:"..."}

{* Uppercase/lowercase *}
{$client_first_name|upper}
{$client_last_name|lower}

{* Escape for output *}
{$client_email|escape}
{$client_email|htmlspecialchars}

{* Default value *}
{$client_company_name|default:'Individual'}

{* Pluralize *}
    You have {$count} new ticket{if $count > 1}s{/if}

{* Markdown to HTML *}
{$ticket_message|markdown}
```

## Template Loop Examples

### Multiple Line Items

```smarty
{if $invoice_items}
<table>
    <tr>
        <th>Description</th>
        <th>Amount</th>
    </tr>
    {foreach $invoice_items as $item}
    <tr>
        <td>{$item.description}</td>
        <td>{$item.amount}</td>
    </tr>
    {/foreach}
</table>
{/if}
```

### Multiple Domains

```smarty
{foreach $domain_names as $domain}
- {$domain}
{/foreach}
```

### Related Services

```smarty
{for $i=1 to $num_services}
    Service: {$service_domain[$i]}
    Next Due: {$service_next_due_date[$i]}
{/for}
```

## Related Documentation

- [Smarty Template Reference](smarty-template-reference.md)
- [Internationalization Guide](internationalization-guide.md)
- [Hooks Reference](hooks-reference.md)
