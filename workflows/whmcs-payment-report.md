# WHMCS Payment Report Workflow

## Overview
This workflow generates payment method and transaction reports.

## Step-by-Step Process

```php
<?php
// /includes/reports/PaymentReportGenerator.php

class PaymentReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'summary' => $this->getPaymentSummary($startDate, $endDate),
            'by_method' => $this->getByPaymentMethod($startDate, $endDate),
            'refunds' => $this->getRefundStats($startDate, $endDate),
            'fees' => $this->getFeeBreakdown($startDate, $endDate)
        ];
    }

    private function getPaymentSummary(string $startDate, string $endDate): array
    {
        $data = Capsule::select("
            SELECT
                COUNT(*) as transactions,
                SUM(amountin) as total_in,
                SUM(amountout) as total_out,
                SUM(amountin - amountout) as net_amount,
                AVG(amountin) as avg_transaction
            FROM tblaccounts
            WHERE date BETWEEN ? AND ?
        ", [$startDate, $endDate]);

        return (array)$data[0];
    }

    private function getByPaymentMethod(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                paymentmethod,
                COUNT(*) as transactions,
                SUM(amountin) as total,
                AVG(amountin) as avg,
                MIN(amountin) as min,
                MAX(amountin) as max
            FROM tblaccounts
            JOIN tblinvoices ON tblaccounts.invoiceid = tblinvoices.id
            WHERE tblaccounts.date BETWEEN ? AND ?
            AND tblaccounts.amountin > 0
            GROUP BY paymentmethod
            ORDER BY total DESC
        ", [$startDate, $endDate]);
    }

    private function getRefundStats(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                COUNT(*) as refund_count,
                SUM(amountout) as refund_total,
                AVG(amountout) as avg_refund
            FROM tblaccounts
            WHERE date BETWEEN ? AND ?
            AND amountout > 0
        ", [$startDate, $endDate]);
    }

    private function getFeeBreakdown(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                paymentmethod,
                SUM(fees) as total_fees,
                AVG(fees) as avg_fee,
                SUM(amountin) as total_volume,
                (SUM(fees) / SUM(amountin)) * 100 as fee_percentage
            FROM tblaccounts
            WHERE date BETWEEN ? AND ?
            AND fees > 0
            GROUP BY paymentmethod
        ", [$startDate, $endDate]);
    }
}
```

## Related Workflows
- [WHMCS Invoice Report](./whmcs-invoice-report.md)
- [WHMCS Revenue Report](./whmcs-revenue-report.md)