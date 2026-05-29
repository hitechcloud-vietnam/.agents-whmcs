# WHMCS API Reporting Workflow

## Purpose
Guide developers through generating reports via WHMCS API.

## Prerequisites
- WHMCS installation
- Report API access
- PHP skills

## Steps

### Phase 1: Report Generation

1. Get reports
   ```php
   // Monthly summary
   $report = localAPI('GetStats', []);
   
   // Client report
   $clients = localAPI('GetClients', [
       'limitstart' => 0,
       'limitnum' => 100,
   ]);
   
   // Order report
   $orders = localAPI('GetOrders', [
       'limitstart' => 0,
       'limitnum' => 100,
       'status' => 'Active',
   ]);
   ```

2. Financial reports
   ```php
   $invoices = localAPI('getInvoices', [
       'status' => 'Paid',
       'limitstart' => 0,
       'limitnum' => 100,
   ]);
   
   $totalRevenue = 0;
   foreach ($invoices['invoices']['invoice'] as $invoice) {
       $totalRevenue += $invoice['total'];
   }
   ```

### Phase 2: Custom Reports

1. Generate custom report
   ```php
   function generateClientReport($startDate, $endDate): array {
       $clients = Capsule::table('tblclients')
           ->whereBetween('created_at', [$startDate, $endDate])
           ->get();
       
       $report = [];
       foreach ($clients as $client) {
           $orders = Capsule::table('tblorders')
               ->where('userid', $client->id)
               ->whereBetween('datecreated', [$startDate, $endDate])
               ->get();
           
           $report[] = [
               'client' => $client,
               'total_orders' => count($orders),
               'total_revenue' => array_sum(array_column($orders, 'total')),
           ];
       }
       
       return $report;
   }
   ```

## Related Workflows
- whmcs-api-integration
- whmcs-api-invoice-sync
- whmcs-api-client-creation
