# WHMCS Support Metrics Widget Module

## Overview
Support ticket statistics widget displaying ticket metrics, response times, and support team performance.

## Module File: widget.php

```php
<?php
/**
 * WHMCS Support Metrics Widget
 * 
 * @package    WHMCS\Module\Widgets
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class SupportMetricsWidget extends WHMCS\Module\Contracts\WidgetModuleInterface
{
    protected $title = 'Support Metrics';
    protected $description = 'Display ticket statistics and support performance metrics';
    protected $priority = 90;
    protected $icon = 'fa-headset';

    public function getData(): array
    {
        return [
            'open_tickets' => $this->getOpenTickets(),
            'pending_tickets' => $this->getPendingTickets(),
            'average_response_time' => $this->getAverageResponseTime(),
            'today_tickets' => $this->getTodayTickets(),
            'sla_compliance' => $this->getSLACompliance(),
            'tickets_by_status' => $this->getTicketsByStatus(),
            'top_departments' => $this->getTopDepartments(),
            'recent_tickets' => $this->getRecentTickets(),
        ];
    }

    public function generateOutput(array $data): string
    {
        $statusColors = [
            'Open' => 'info',
            'Answered' => 'warning',
            'Customer-Reply' => 'primary',
            'Closed' => 'success',
        ];

        return <<<HTML
<div class="widget-content-padded">
    <div class="row">
        <div class="col-sm-4 text-center">
            <div class="metric-box">
                <span class="metric-value">{$data['open_tickets']}</span>
                <span class="metric-label">Open Tickets</span>
            </div>
        </div>
        <div class="col-sm-4 text-center">
            <div class="metric-box">
                <span class="metric-value">{$data['pending_tickets']}</span>
                <span class="metric-label">Awaiting Reply</span>
            </div>
        </div>
        <div class="col-sm-4 text-center">
            <div class="metric-box">
                <span class="metric-value">{$data['today_tickets']}</span>
                <span class="metric-label">Today</span>
            </div>
        </div>
    </div>

    <hr>

    <div class="row">
        <div class="col-sm-6">
            <div class="metric-box">
                <span class="metric-value">{$data['average_response_time']}</span>
                <span class="metric-label">Avg Response</span>
            </div>
        </div>
        <div class="col-sm-6">
            <div class="metric-box">
                <span class="metric-value {$this->getSLAClass($data['sla_compliance'])}">{$data['sla_compliance']}%</span>
                <span class="metric-label">SLA Compliance</span>
            </div>
        </div>
    </div>

    <hr>

    <div class="tickets-by-status">
        <h5>Tickets by Status</h5>
        {$this->renderStatusChart($data['tickets_by_status'])}
    </div>

    <hr>

    <div class="recent-tickets">
        <h5>Recent Tickets</h5>
        <table class="table table-striped table-sm">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Subject</th>
                    <th>Status</th>
                </tr>
            </thead>
            <tbody>
                {$this->renderRecentTickets($data['recent_tickets'])}
            </tbody>
        </table>
    </div>
</div>

<style>
.metric-box {
    padding: 10px;
}
.metric-value {
    display: block;
    font-size: 24px;
    font-weight: bold;
    color: #2a2a2a;
}
.metric-label {
    display: block;
    font-size: 11px;
    color: #777;
    text-transform: uppercase;
}
.metric-value.warning { color: #e67e22; }
.metric-value.danger { color: #e74c3c; }
.metric-value.success { color: #27ae60; }
</style>
HTML;
    }

    protected function getOpenTickets(): int
    {
        return Capsule::table('tbltickets')
            ->whereIn('status', ['Open', 'Answered', 'Customer-Reply'])
            ->count();
    }

    protected function getPendingTickets(): int
    {
        return Capsule::table('tbltickets')
            ->where('status', 'Customer-Reply')
            ->count();
    }

    protected function getTodayTickets(): int
    {
        return Capsule::table('tbltickets')
            ->where('date', '>=', date('Y-m-d 00:00:00'))
            ->count();
    }

    protected function getAverageResponseTime(): string
    {
        $hours = Capsule::table('tblticketreplies')
            ->where('date', '>=', date('Y-m-d 00:00:00', strtotime('-7 days')))
            ->selectRaw('AVG(TIMESTAMPDIFF(HOUR, 
                (SELECT date FROM tbltickets WHERE id = tblticketreplies.tid ORDER BY date ASC LIMIT 1),
                tblticketreplies.date
            )) as avg_hours')
            ->value('avg_hours');

        if ($hours === null) {
            return 'N/A';
        }

        if ($hours < 1) {
            return '< 1h';
        } elseif ($hours < 24) {
            return round($hours) . 'h';
        } else {
            return round($hours / 24) . 'd';
        }
    }

    protected function getSLACompliance(): int
    {
        $total = Capsule::table('mod_ticket_sla')
            ->where('status', 'closed')
            ->where('created_at', '>=', date('Y-m-d 00:00:00', strtotime('-30 days')))
            ->count();

        if ($total === 0) {
            return 100;
        }

        $met = Capsule::table('mod_ticket_sla')
            ->where('status', 'closed')
            ->where('resolution_status', 'met')
            ->where('created_at', '>=', date('Y-m-d 00:00:00', strtotime('-30 days')))
            ->count();

        return round(($met / $total) * 100);
    }

    protected function getTicketsByStatus(): array
    {
        return Capsule::table('tbltickets')
            ->groupBy('status')
            ->selectRaw('status, COUNT(*) as count')
            ->get();
    }

    protected function getTopDepartments(): array
    {
        return Capsule::table('tbltickets')
            ->join('tbldepartments', 'tbltickets.did', '=', 'tbldepartments.id')
            ->whereIn('tbltickets.status', ['Open', 'Answered', 'Customer-Reply'])
            ->groupBy('tbldepartments.id')
            ->selectRaw('tbldepartments.name, COUNT(*) as count')
            ->orderBy('count', 'desc')
            ->limit(5)
            ->get();
    }

    protected function getRecentTickets(): array
    {
        return Capsule::table('tbltickets')
            ->orderBy('lastreply', 'desc')
            ->limit(5)
            ->get(['id', 'tid', 'subject', 'status']);
    }

    protected function getSLAClass(int $compliance): string
    {
        if ($compliance >= 90) {
            return 'success';
        } elseif ($compliance >= 70) {
            return 'warning';
        }
        return 'danger';
    }

    protected function renderStatusChart(array $statuses): string
    {
        $html = '<div class="progress" style="height: 25px;">';
        $total = array_sum(array_column($statuses, 'count'));
        
        if ($total === 0) {
            return '<p class="text-muted">No tickets</p>';
        }

        $colors = [
            'Open' => 'bg-info',
            'Answered' => 'bg-warning',
            'Customer-Reply' => 'bg-primary',
            'Closed' => 'bg-success',
        ];

        foreach ($statuses as $status) {
            $width = ($status->count / $total) * 100;
            $color = $colors[$status->status] ?? 'bg-secondary';
            $html .= "<div class=\"progress-bar {$color}\" style=\"width: {$width}%\" 
                data-toggle=\"tooltip\" title=\"{$status->status}: {$status->count}\">
                {$status->count}
            </div>";
        }

        $html .= '</div>';
        return $html;
    }

    protected function renderRecentTickets(array $tickets): string
    {
        $html = '';
        
        foreach ($tickets as $ticket) {
            $statusClass = $this->getStatusClass($ticket->status);
            $subject = htmlspecialchars(substr($ticket->subject, 0, 40));
            if (strlen($ticket->subject) > 40) {
                $subject .= '...';
            }
            
            $html .= "<tr>
                <td>#{$ticket->id}</td>
                <td>{$subject}</td>
                <td><span class=\"label label-{$statusClass}\">{$ticket->status}</span></td>
            </tr>";
        }
        
        return $html ?: '<tr><td colspan="3" class="text-center text-muted">No recent tickets</td></tr>';
    }

    protected function getStatusClass(string $status): string
    {
        $classes = [
            'Open' => 'info',
            'Answered' => 'warning',
            'Customer-Reply' => 'primary',
            'Closed' => 'success',
        ];
        return $classes[$status] ?? 'default';
    }

    public function getId(): string
    {
        return 'support_metrics_widget';
    }

    public function getName(): string
    {
        return $this->title;
    }
}
```

## Activation & Deactivation

```php
<?php
function whmcs_support_metrics_widget_activate(): array
{
    return ['status' => 'success', 'description' => 'Support Metrics Widget activated'];
}

function whmcs_support_metrics_widget_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Support Metrics Widget deactivated'];
}
```

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
