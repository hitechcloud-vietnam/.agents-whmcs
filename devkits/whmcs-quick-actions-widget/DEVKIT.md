# WHMCS Quick Actions Widget Module

## Overview
Admin shortcuts widget for quick access to common tasks.

## Module File: widget.php

```php
<?php
/**
 * WHMCS Quick Actions Widget
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class QuickActionsWidget extends WHMCS\Module\Contracts\WidgetModuleInterface
{
    protected $title = 'Quick Actions';
    protected $description = 'Admin shortcuts and quick links';
    protected $priority = 100;
    protected $icon = 'fa-bolt';

    public function getData(): array
    {
        return [
            'pending_orders' => $this->getPendingOrders(),
            'pending_tickets' => $this->getPendingTickets(),
            'overdue_invoices' => $this->getOverdueInvoices(),
        ];
    }

    public function generateOutput(array $data): string
    {
        return <<<HTML
<div class="widget-content-padded">
    <div class="quick-actions-grid">
        <a href="orders.php?status=Pending" class="quick-action-item">
            <i class="fa fa-shopping-cart"></i>
            <span class="quick-action-label">Pending Orders</span>
            <span class="quick-action-badge">{$data['pending_orders']}</span>
        </a>

        <a href="supporttickets.php?status=Open" class="quick-action-item">
            <i class="fa fa-life-ring"></i>
            <span class="quick-action-label">Open Tickets</span>
            <span class="quick-action-badge">{$data['pending_tickets']}</span>
        </a>

        <a href="invoices.php?status=Overdue" class="quick-action-item">
            <i class="fa fa-exclamation-triangle"></i>
            <span class="quick-action-label">Overdue Invoices</span>
            <span class="quick-action-badge">{$data['overdue_invoices']}</span>
        </a>

        <a href="clients.php?action=add" class="quick-action-item">
            <i class="fa fa-user-plus"></i>
            <span class="quick-action-label">Add Client</span>
        </a>

        <a href="orders.php?action=create" class="quick-action-item">
            <i class="fa fa-cart-plus"></i>
            <span class="quick-action-label">Create Order</span>
        </a>

        <a href="supporttickets.php?action=open" class="quick-action-item">
            <i class="fa fa-plus-circle"></i>
            <span class="quick-action-label">New Ticket</span>
        </a>
    </div>
</div>

<style>
.quick-actions-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 10px;
}
.quick-action-item {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 15px 5px;
    background: #f8f9fa;
    border-radius: 8px;
    text-decoration: none;
    color: #333;
    transition: all 0.2s;
}
.quick-action-item:hover {
    background: #e9ecef;
    transform: translateY(-2px);
}
.quick-action-item i {
    font-size: 24px;
    margin-bottom: 8px;
    color: #3498db;
}
.quick-action-label {
    font-size: 11px;
    text-align: center;
}
.quick-action-badge {
    background: #e74c3c;
    color: white;
    padding: 2px 8px;
    border-radius: 10px;
    font-size: 11px;
    margin-top: 5px;
}
</style>
HTML;
    }

    protected function getPendingOrders(): int
    {
        return Capsule::table('tblorders')
            ->whereIn('status', ['Pending', 'Pending_Manual'])
            ->count();
    }

    protected function getPendingTickets(): int
    {
        return Capsule::table('tbltickets')
            ->whereIn('status', ['Open', 'Answered'])
            ->count();
    }

    protected function getOverdueInvoices(): int
    {
        return Capsule::table('tblinvoices')
            ->where('status', 'Overdue')
            ->count();
    }

    public function getId(): string { return 'quick_actions_widget'; }
    public function getName(): string { return $this->title; }
}

function whmcs_quick_actions_widget_activate(): array
{
    return ['status' => 'success', 'description' => 'Quick Actions Widget activated'];
}

function whmcs_quick_actions_widget_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Quick Actions Widget deactivated'];
}
