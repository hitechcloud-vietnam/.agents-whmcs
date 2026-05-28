# WHMCS Overdue Handler Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build automated overdue invoice handling with escalation.

## Overdue Module

```php
<?php
class OverdueHandler {
    private array $stages = [
        1 => ['days' => 1, 'action' => 'reminder', 'email' => 'InvoiceReminder1'],
        2 => ['days' => 5, 'action' => 'reminder', 'email' => 'InvoiceReminder2'],
        3 => ['days' => 10, 'action' => 'suspend', 'email' => 'LateNotice'],
        4 => ['days' => 15, 'action' => 'second_reminder', 'email' => 'SecondOverdueReminder'],
        5 => ['days' => 30, 'action' => 'terminate', 'email' => 'TerminationNotice'],
    ];

    public function processOverdueInvoices(): array {
        $results = ['reminders' => 0, 'suspensions' => 0, 'terminations' => 0];

        $overdueInvoices = Capsule::table('tblinvoices')
            ->whereIn('status', ['Unpaid', 'Overdue'])
            ->where('duedate', '<', date('Y-m-d'))
            ->get();

        foreach ($overdueInvoices as $invoice) {
            $daysOverdue = $this->getDaysOverdue($invoice->duedate);
            $stage = $this->determineStage($daysOverdue);

            if ($stage && $this->shouldProcess($invoice, $stage)) {
                $result = $this->processStage($invoice, $stage, $daysOverdue);
                $results[$result['type']]++;
            }
        }

        return $results;
    }

    private function processStage(object $invoice, int $stage, int $daysOverdue): array {
        $action = $this->stages[$stage]['action'];

        switch ($action) {
            case 'reminder':
                $this->sendReminder($invoice, $this->stages[$stage]['email']);
                $this->logStage($invoice->id, $stage, 'reminder_sent');
                return ['type' => 'reminders'];

            case 'suspend':
                $this->suspendService($invoice->userid);
                $this->sendReminder($invoice, $this->stages[$stage]['email']);
                $this->logStage($invoice->id, $stage, 'service_suspended');
                return ['type' => 'suspensions'];

            case 'terminate':
                $this->terminateService($invoice->userid);
                $this->sendReminder($invoice, $this->stages[$stage]['email']);
                $this->logStage($invoice->id, $stage, 'service_terminated');
                return ['type' => 'terminations'];

            default:
                return ['type' => 'none'];
        }
    }

    private function suspendService(int $userId): void {
        $services = Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->where('domainstatus', 'Active')
            ->get();

        foreach ($services as $service) {
            $params = [
                'serviceid' => $service->id,
                'model' => \WHMCS\Service\Environment::getServiceModel($service->id),
            ];

            $module = ModuleBuilder::build($service->servertype);
            $module->SuspendAccount($params);
        }
    }

    private function terminateService(int $userId): void {
        $services = Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->whereIn('domainstatus', ['Active', 'Suspended'])
            ->get();

        foreach ($services as $service) {
            $params = [
                'serviceid' => $service->id,
                'model' => \WHMCS\Service\Environment::getServiceModel($service->id),
            ];

            $module = ModuleBuilder::build($service->servertype);
            $module->TerminateAccount($params);
        }
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-cron-automation