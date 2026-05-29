# WHMCS Client Activity Widget Module

## Overview
Display recent client activity feed showing logins, purchases, and service changes.

## Module File: widget.php

```php
<?php
/**
 * WHMCS Client Activity Widget
 * 
 * @package    WHMCS\Module\Widgets
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class ClientActivityWidget extends WHMCS\Module\Contracts\WidgetModuleInterface
{
    protected $title = 'Client Activity';
    protected $description = 'Recent client activity feed';
    protected $priority = 75;
    protected $icon = 'fa-users';

    public function getData(): array
    {
        return [
            'activities' => $this->getRecentActivities(),
            'active_clients' => $this->getActiveClientsToday(),
            'new_registrations' => $this->getNewRegistrations(),
        ];
    }

    public function generateOutput(array $data): string
    {
        return <<<HTML
<div class="widget-content-padded">
    <div class="row text-center">
        <div class="col-sm-6">
            <div class="metric-box">
                <span class="metric-value">{$data['active_clients']}</span>
                <span class="metric-label">Active Today</span>
            </div>
        </div>
        <div class="col-sm-6">
            <div class="metric-box">
                <span class="metric-value text-success">{$data['new_registrations']}</span>
                <span class="metric-label">New Clients</span>
            </div>
        </div>
    </div>

    <hr>

    <div class="activity-feed">
        {$this->renderActivities($data['activities'])}
    </div>
</div>

<style>
.activity-item { padding: 10px 0; border-bottom: 1px solid #eee; }
.activity-icon { width: 30px; height: 30px; border-radius: 50%; display: inline-flex; align-items: center; justify-content: center; margin-right: 10px; }
.activity-icon.login { background: #e8f5e9; color: #4caf50; }
.activity-icon.order { background: #e3f2fd; color: #2196f3; }
.activity-icon.payment { background: #fff3e0; color: #ff9800; }
.activity-icon.ticket { background: #fce4ec; color: #e91e63; }
.activity-content { display: inline-block; vertical-align: top; }
.activity-text { font-size: 13px; }
.activity-time { font-size: 11px; color: #999; }
</style>
HTML;
    }

    protected function getRecentActivities(): array
    {
        $activities = [];

        // Recent logins
        $logins = Capsule::table('mod_security_login_attempts')
            ->join('tblclients', 'mod_security_login_attempts.user_id', '=', 'tblclients.id')
            ->where('mod_security_login_attempts.success', true)
            ->orderBy('mod_security_login_attempts.attempted_at', 'desc')
            ->limit(5)
            ->get([
                'mod_security_login_attempts.attempted_at as timestamp',
                Capsule::raw("CONCAT(tblclients.firstname, ' ', tblclients.lastname) as client_name"),
                Capsule::raw("'login' as type"),
            ]);

        foreach ($logins as $login) {
            $activities[] = [
                'type' => 'login',
                'icon' => 'fa-sign-in',
                'text' => $login->client_name . ' logged in',
                'time' => $login->timestamp,
            ];
        }

        // Recent orders
        $orders = Capsule::table('tblorders')
            ->join('tblclients', 'tblorders.userid', '=', 'tblclients.id')
            ->orderBy('tblorders.date', 'desc')
            ->limit(5)
            ->get([
                'tblorders.date as timestamp',
                'tblorders.total',
                Capsule::raw("CONCAT(tblclients.firstname, ' ', tblclients.lastname) as client_name"),
                Capsule::raw("'order' as type"),
            ]);

        foreach ($orders as $order) {
            $activities[] = [
                'type' => 'order',
                'icon' => 'fa-shopping-cart',
                'text' => $order->client_name . ' placed order (#' . $order->id . ')',
                'time' => $order->timestamp,
            ];
        }

        // Sort by time
        usort($activities, fn($a, $b) => strtotime($b['time']) - strtotime($a['time']));

        return array_slice($activities, 0, 8);
    }

    protected function getActiveClientsToday(): int
    {
        return Capsule::table('mod_security_login_attempts')
            ->where('success', true)
            ->where('attempted_at', '>=', date('Y-m-d 00:00:00'))
            ->distinct('user_id')
            ->count('user_id');
    }

    protected function getNewRegistrations(): int
    {
        return Capsule::table('tblclients')
            ->where('datecreated', '>=', date('Y-m-d 00:00:00'))
            ->count();
    }

    protected function renderActivities(array $activities): string
    {
        if (empty($activities)) {
            return '<p class="text-muted text-center">No recent activity</p>';
        }

        $html = '';
        foreach ($activities as $activity) {
            $time = $this->formatTime($activity['time']);
            $html .= <<<HTML
<div class="activity-item">
    <div class="activity-icon {$activity['type']}">
        <i class="fa {$activity['icon']}"></i>
    </div>
    <div class="activity-content">
        <div class="activity-text">{$activity['text']}</div>
        <div class="activity-time">{$time}</div>
    </div>
</div>
HTML;
        }
        return $html;
    }

    protected function formatTime(string $timestamp): string
    {
        $time = strtotime($timestamp);
        $diff = time() - $time;

        if ($diff < 60) {
            return 'Just now';
        } elseif ($diff < 3600) {
            return floor($diff / 60) . ' min ago';
        } elseif ($diff < 86400) {
            return floor($diff / 3600) . ' hours ago';
        }
        return date('M j', $time);
    }

    public function getId(): string
    {
        return 'client_activity_widget';
    }

    public function getName(): string
    {
        return $this->title;
    }
}

function whmcs_client_activity_widget_activate(): array
{
    return ['status' => 'success', 'description' => 'Client Activity Widget activated'];
}

function whmcs_client_activity_widget_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Client Activity Widget deactivated'];
}
