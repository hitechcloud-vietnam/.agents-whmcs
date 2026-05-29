# WHMCS Template Variables

## Overview

WHMCS provides numerous pre-defined variables accessible in templates. These variables contain data about clients, products, invoices, and system configuration.

## Client Variables

### Basic Client Info

```smarty
{$client->id}
{$client->firstname}
{$client->lastname}
{$client->fullname}
{$client->companyname}
{$client->email}
{$client->phonecc}
{$client->phonenumber}
{$client->address1}
{$client->address2}
{$client->city}
{$client->state}
{$client->postcode}
{$client->country}
{$client->countrycode}
{$client->currency}
{$client->taxexempt}
{$client->notes}
{$client->status}
{$client->credit}
```

### Client Dates

```smarty
{$client->datecreated}
{$client->lastlogin}
{$client->lastcontact}
{$client->created_at}
```

### Client Properties Check

```smarty
{if $client->companyname}
    <p>Company: {$client->companyname}</p>
{/if}
```

## Product/Service Variables

### Service Object

```smarty
{$service->id}
{$service->ordernum}
{$service->orderid}
{$service->pid}
{$service->product_id}
{$service->package}
{$service->type}
{$service->domain}
{$service->username}
{$service->password}
{$service->status}
{$service->server}
{$service->subscriptionid}
{$service->registrationdate}
{$service->nextduedate}
{$service->termination_date}
{$service->firstpaymentamount}
{$service->recurringamount}
{$service->billingcycle}
{$service->paymentmethod}
{$service->notes}
```

### Product Pricing

```smarty
{$pricing->monthly}
{$pricing->quarterly}
{$pricing->semiannually}
{$pricing->annually}
{$pricing->biennially}
{$pricing->triennially}
{$pricing->msetupfee}
{$pricing->qsetupfee}
{$pricing->ssetupfee}
{$pricing->asetupfee}
{$pricing->bsetupfee}
{$pricing->tsetupfee}
{$pricing->product}
{$pricing->currency}
```

## Invoice Variables

### Invoice Object

```smarty
{$invoice->id}
{$invoice->invoicenum}
{$invoice->userid}
{$invoice->status}
{$invoice->date}
{$invoice->duedate}
{$invoice->datepaid}
{$invoice->subtotal}
{$invoice->tax}
{$invoice->tax2}
{$invoice->total}
{$invoice->credit}
{$invoice->amountpaid}
{$invoice->balance}
{$invoice->paymentmethod}
{$invoice->notes}
```

### Invoice Line Items

```smarty
{foreach $invoice->lines as $line}
    {$line->id}
    {$line->type}
    {$line->relid}
    {$line->description}
    {$line->amount}
    {$line->taxed}
{/foreach}
```

## Domain Variables

```smarty
{$domain->id}
{$domain->userid}
{$domain->type}
{$domain->domain}
{$domain->registrationdate}
{$domain->nextduedate}
{$domain->expirydate}
{$domain->status}
{$domain->dnsmanagement}
{$domain->emailforwarding}
{$domain->idprotection}
{$domain->registrar}
{$domain->registrationperiod}
{$domain->isexpired}
{$domain->daysuntilexpiry}
```

## Order Variables

```smarty
{$order->id}
{$order->ordernum}
{$order->userid}
{$order->contactid}
{$order->qtype}
{$order->date}
{$order->nameservers}
{$order->transfersecret}
{$order->orderdata}
{$order->amount}
{$order->invoiceid}
{$order->status}
```

## Ticket Variables

```smarty
{$ticket->id}
{$ticket->tid}
{$ticket->cannedcatid}
{$ticket->did}
{$ticket->userid}
{$ticket->contactid}
{$ticket->adminid}
{$ticket->title}
{$ticket->message}
{$ticket->status}
{$ticket->urgency}
{$ticket->replyingadmin}
{$ticket->replyingtime}
{$ticket->date}
{$ticket->lastreply}
```

## Custom Fields

```smarty
{foreach $customfields as $field}
    {$field->id}
    {$field->fieldname}
    {$field->fieldtype}
    {$field->value}
    {$field->raw_value}
{/foreach}

{* Direct access by name *}
{$customfields["Company Name"]}
{$customfields.CompanyName}
```

## Config Options

```smarty
{foreach $configoptions as $option}
    {$option->id}
    {$option->optionid}
    {$option->optionname}
    {$option->qty}
    {$option->rawqty}
    {$option->option}
{/foreach}
```

## System Variables

### Smarty Built-ins

```smarty
{$smarty.now}                    {* Current timestamp *}
{$smarty.template}               {* Current template name *}
{$smarty.current_dir}            {* Template directory *}
{$smarty.version}                {* Smarty version *}
{$smarty.const.SOME_CONSTANT}    {* PHP constants *}
{$smarty.session.var}            {* Session variables *}
{$smarty.cookies.var}            {* Cookie variables *}
{$smarty.get.var}                {* GET parameters *}
{$smarty.post.var}               {* POST parameters *}
{$smarty.server.SERVER_NAME}      {* Server variables *}
```

### System Config

```smarty
{$COMPANY_NAME}
{$COMPANY_EMAIL}
{$COMPANY_ADDRESS}
{$COMPANY_URL}
{$SYSTEM_URL}
{$SYSTEM_ROOT}
{$DATE_FORMAT}
{$TIME_FORMAT}
{$DEFAULT_COUNTRY}
```

## Cart Variables

```smarty
{$cart}{*$cart object*}
{$carttotal}
{$carttotalrecurring}
{$cartitems}
{$promotioncode}
{$noconfiguration}
{$selectedcurrency}
```

## Useful Conditionals

```smarty
{if $client}
    Logged in client
{/if}

{if $loggedin}
    User is logged in
{/if}

{if $inaddticket}
    Adding support ticket
{/if}

{if $inadmin}
    Admin area access
{/if}

{if $templatefile == "viewinvoice"}
    On invoice page
{/if}
```

## See Also

- [Smarty Syntax](../whmcs-smarty-syntax.md)
- [Template Functions](../whmcs-template-functions.md)