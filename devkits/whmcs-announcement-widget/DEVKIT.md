# WHMCS Announcement Widget Module

## Overview
Display latest announcements widget for admin dashboard.

## Module File: widget.php

```php
<?php
/**
 * WHMCS Announcement Widget
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class AnnouncementWidget extends WHMCS\Module\Contracts\WidgetModuleInterface
{
    protected $title = 'Announcements';
    protected $description = 'Latest system announcements';
    protected $priority = 60;
    protected $icon = 'fa-bullhorn';

    public function getData(): array
    {
        return [
            'announcements' => $this->getAnnouncements(),
            'draft_count' => $this->getDraftCount(),
        ];
    }

    public function generateOutput(array $data): string
    {
        return <<<HTML
<div class="widget-content-padded">
    <div class="row text-center">
        <div class="col-sm-6">
            <span class="text-muted">Latest announcement posted</span>
        </div>
    </div>

    <hr>

    <div class="announcement-list">
        {$this->renderAnnouncements($data['announcements'])}
    </div>

    <div class="text-center" style="margin-top: 10px;">
        <a href="announcements.php" class="btn btn-default btn-sm">View All</a>
    </div>
</div>
HTML;
    }

    protected function getAnnouncements(): array
    {
        return Capsule::table('tblannouncements')
            ->orderBy('date', 'desc')
            ->limit(5)
            ->get(['id', 'title', 'date', 'published']);
    }

    protected function getDraftCount(): int
    {
        return Capsule::table('tblannouncements')
            ->where('published', 0)
            ->count();
    }

    protected function renderAnnouncements(array $announcements): string
    {
        if (empty($announcements)) {
            return '<p class="text-muted text-center">No announcements</p>';
        }

        $html = '<ul class="list-unstyled">';
        foreach ($announcements as $ann) {
            $date = date('M j, Y', strtotime($ann->date));
            $title = htmlspecialchars($ann->title);
            $statusClass = $ann->published ? '' : 'text-muted';
            $icon = $ann->published ? 'fa-check-circle text-success' : 'fa-clock text-warning';
            
            $html .= '<li style="padding: 8px 0; border-bottom: 1px solid #eee;">
                <i class="fa ' . $icon . '"></i>
                <a href="announcements.php?action=edit&id=' . $ann->id . '" class="' . $statusClass . '">' . $title . '</a>
                <div class="small text-muted">' . $date . '</div>
            </li>';
        }
        $html .= '</ul>';
        return $html;
    }

    public function getId(): string { return 'announcement_widget'; }
    public function getName(): string { return $this->title; }
}

function whmcs_announcement_widget_activate(): array
{
    return ['status' => 'success', 'description' => 'Announcement Widget activated'];
}

function whmcs_announcement_widget_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Announcement Widget deactivated'];
}
