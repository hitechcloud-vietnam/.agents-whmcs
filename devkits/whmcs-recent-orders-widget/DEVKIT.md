# WHMCS Recent Orders Widget Module

## Overview
Display the latest orders and their status in the admin dashboard.

## Module File: widget.php

```php
<?php
/**
 * WHMCS Recent Orders Widget
 * 
 * @package    WHMCS\Module\Widgets
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class RecentOrdersWidget extends WHMCS\Module\Contracts\WidgetModuleInterface
{
    protected $title = 'Recent Orders';
    protected $description = 'Display latest orders and their status';
    protected $priority = 80;
    protected $icon = 'fa-shopping-cart';

    public function getData(): array
    {
        return [
            'orders' => $this->getRecentOrders(),
            'today_count' => $this->getTodayOrderCount(),
            'pending_count' => $this->getPendingCount(),
            'fraud_count' => $this->getFraudCount(),
        ];
    }

    public function generateOutput(array $data): string
    {
        return <<<HTML
<div class="widget-content-padded">
    <div class="row text-center">
        <div class="col-sm-4">
            <div class="metric-box">
                <span class="metric-value">{$data['today_count']}</span>
                <span class="metric-label">Today</span>
            </div>
        </div>
        <div class="col-sm-4">
            <div class="metric-box">
                <span class="metric-value text-warning">{$data['pending_count']}</span>
                <span class="metric-label">Pending</span>
            </div>
        </div>
        <div class="col-sm-4">
            <div class="metric-box">
                <span class="metric-value text-danger">{$data['fraud_count']}</span>
                <span class="metric-label">Fraud Check</span>
            </div>
        </div>
    </div>

    <hr>

    <div class="recent-orders">
        <table class="table table-striped table-sm">
            <thead>
                <tr>
                    <th>Order #</th>
                    <th>Client</th>
                    <th>Amount</th>
                    <th>Status</th>
                </tr>
            </thead>
            <tbody>
                {$this->renderOrders($data['orders'])}
            </tbody>
        </table>
    </div>
</div>

<style>
.metric-box { padding: 10px; }
.metric-value { display: block; font-size: 22px; font-weight: bold; }
.metric-label { display: block; font-size: 11px; color: #777; text-transform: uppercase; }
</style>
HTML;
    }

    protected function getRecentOrders(): array
    {
        return Capsule::table('tblorders')
            ->join('tblclients', 'tblorders.userid', '=', 'tblclients.id')
            ->orderBy('tblorders.date', 'desc')
            ->limit(8)
            ->get([
                'tblorders.id',
                'tblorders.date',
                'tblorders.total',
                'tblorders.status',
                Capsule::raw("CONCAT(tblclients.firstname, ' ', tblclients.lastname) as client_name"),
            ]);
    }

    protected function getTodayOrderCount(): int
    {
        return Capsule::table('tblorders')
            ->where('date', '>=', date('Y-m-d 00:00:00'))
            ->count();
    }

    protected function getPendingCount(): int
    {
        return Capsule::table('tblorders')
            ->whereIn('status', ['Pending', 'Pending_Manual'])
            ->count();
    }

    protected function getFraudCount(): int
    {
        return Capsule::table('tblorders')
            ->where('status', 'Fraud')
            ->count();
    }

    protected function renderOrders(array $orders): string
    {
        $html = '';
        foreach ($orders as $order) {
            $statusClass = $this->getStatusClass($order->status);
            $date = date('M j', strtotime($order->date));
            $amount = formatCurrency($order->total);
            $client = htmlspecialchars($order->client_name);
            
            $html .= "<tr>
                <td><a href=\"orders.php?action=view&id={$order->id}\">#{$order->id}</a></td>
                <td>{$client}</td>
                <td>{$amount}</td>
                <td><span class=\"label label-{$statusClass}\">{$order->status}</span></td>
            </tr>";
        }
        return $html ?: '<tr><td colspan="4" class="text-center text-muted">No recent orders</td></tr>';
    }

    protected function getStatusClass(string $status): string
    {
        $classes = [
            'Active' => 'success',
            'Pending' => 'warning',
            'Pending_Manual' => 'warning',
            'Fraud' => 'danger',
            'Cancelled' => 'default',
            'Completed' => 'success',
        ];
        return $classes[$status] ?? 'default';
    }

    public function getId(): string
    {
        return 'recent_orders_widget';
    }

    public function getName(): string
    {
        return $this->title;
    }
}

function whmcs_recent_orders_widget_activate(): array
{
    return ['status' => 'success', 'description' => 'Recent Orders Widget activated'];
}

function whmcs_recent_orders_widget_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Recent Orders Widget deactivated'];
}
