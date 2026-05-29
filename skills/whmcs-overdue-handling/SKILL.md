# WHMCS Overdue Handling

## Overview
Master skill for handling overdue invoices and accounts in WHMCS. Covers late fees, suspension workflows, and collection processes.

## Overdue Hooks

```php
<?php
// /includes/hooks/overdue_hooks.php

add_hook("InvoiceOverdue", 1, function(array $params) {
    $invoiceId = $params["invoiceid"];
    $daysOverdue = $params["daysOverdue"];
    $clientId = $params["userid"];
    
    switch ($daysOverdue) {
        case 1:
            send_email("InvoiceReminder1", $clientId, [
                "invoice_id" => $invoiceId,
                "days_overdue" => $daysOverdue,
            ]);
            break;
            
        case 3:
            send_email("InvoiceReminder3", $clientId, [
                "invoice_id" => $invoiceId,
                "days_overdue" => $daysOverdue,
            ]);
            break;
            
        case 7:
            addLateFee($invoiceId, 5.00);
            send_email("InvoiceReminder7", $clientId, [
                "invoice_id" => $invoiceId,
                "days_overdue" => $daysOverdue,
            ]);
            break;
            
        case 14:
            suspendServices($clientId);
            send_email("ServicesSuspended", $clientId, [
                "reason" => "Overdue invoice",
            ]);
            break;
    }
    
    return true;
});

function suspendServices(int $clientId): void
{
    $services = \Illuminate\Database\Capsule\Manager::table("tblhosting")
        ->where("userid", $clientId)
        ->where("domainstatus", "Active")
        ->get();
    
    foreach ($services as $service) {
        $module = new \WHMCS\Module\Server();
        $module->load($service->module);
        $module->suspendAccount([
            "serviceid" => $service->id,
            "suspendreason" => "Overdue Invoice",
        ]);
    }
}
```

## Overdue Manager

```php
<?php
// /includes/managers/OverdueManager.php

namespace WHMCS\Billing;

class OverdueManager
{
    public function getOverdueInvoices(int $daysThreshold = 1): array
    {
        $cutoffDate = date("Y-m-d", strtotime("-{$daysThreshold} days"));
        
        return \Illuminate\Database\Capsule\Manager::table("tblinvoices")
            ->join("tblclients", "tblinvoices.userid", "=", "tblclients.id")
            ->where("tblinvoices.status", "Unpaid")
            ->where("tblinvoices.duedate", "<", $cutoffDate)
            ->select("tblinvoices.*", "tblclients.email", "tblclients.firstname", "tblclients.lastname")
            ->get();
    }
    
    public function addLateFee(int $invoiceId, float $fee): bool
    {
        $invoice = \WHMCS\Billing\Invoice::find($invoiceId);
        
        if (!$invoice || $invoice->status !== "Unpaid") {
            return false;
        }
        
        addInvoiceLineItem($invoiceId, [
            "description" => "Late Payment Fee",
            "amount" => $fee,
            "taxed" => false,
        ]);
        
        return true;
    }
    
    public function calculateDaysOverdue(int $invoiceId): int
    {
        $invoice = \WHMCS\Billing\Invoice::find($invoiceId);
        
        if (!$invoice) {
            return 0;
        }
        
        $dueDate = new \DateTime($invoice->duedate);
        $today = new \DateTime();
        
        if ($today <= $dueDate) {
            return 0;
        }
        
        return $today->diff($dueDate)->days;
    }
    
    public function shouldSuspend(int $invoiceId, int $thresholdDays = 14): bool
    {
        return $this->calculateDaysOverdue($invoiceId) >= $thresholdDays;
    }
    
    public function getCollectionStatus(int $clientId): array
    {
        $overdueInvoices = $this->getOverdueInvoices(1)
            ->where("userid", $clientId);
        
        $totalOverdue = $overdueInvoices->sum("total");
        $oldestDays = $overdueInvoices->max(function($inv) {
            return calculateDaysOverdue($inv->id);
        });
        
        return [
            "has_overdue" => $overdueInvoices->count() > 0,
            "total_overdue" => $totalOverdue,
            "oldest_days" => $oldestDays,
            "invoice_count" => $overdueInvoices->count(),
        ];
    }
}
```

## Best Practices

1. **Escalation Timeline**: Define clear overdue stages
2. **Multiple Reminders**: Send reminders at regular intervals
3. **Late Fees**: Apply progressive late fees
4. **Service Suspension**: Suspend services after threshold
5. **Collection Process**: Implement collection workflows
6. **Payment Plans**: Offer payment arrangements
7. **Documentation**: Log all collection activities
8. **Customer Communication**: Be professional but firm
