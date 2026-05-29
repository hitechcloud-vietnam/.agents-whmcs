# WHMCS Report Debug Workflow

## Overview
This workflow guides you through debugging report generation issues.

## Prerequisites
- Report implementation
- WHMCS admin access

## Step-by-Step Guide

### Step 1: Enable Report Debugging
```php
// In your report class
class ClientReport extends \WHMCS\Report\Table
{
    protected function loadQueryBuilder()
    {
        try {
            $builder = \WHMCS\Database\Capsule::table('tblclients')
                ->select([
                    'id',
                    'firstname',
                    'lastname',
                    'email',
                    'datecreated',
                ]);

            $this->buildOrderByFromRequest($builder);

            return $builder;

        } catch (Exception $e) {
            logModuleCall(
                'yourmodule',
                'report_error',
                'ClientReport',
                $e->getMessage()
            );
            throw $e;
        }
    }
}
```

### Step 2: Test Report Directly
```bash
php -r "
require '/var/www/whmcs/init.php';
use WHMCS\Report\Factory;
\$report = Factory::report('ClientSummary');
\$report->run();
print_r(\$report->getData());
"
```

### Step 3: Check Report Permissions
```php
// Verify user has permission
if (!\WHMCS\Session::get('adminid')) {
    throw new Exception('Not authenticated');
}

$admin = \WHMCS\User\Admin::find(\WHMCS\Session::get('adminid'));
if (!$admin->hasPermission('reports')) {
    throw new Exception('Permission denied');
}
```

### Step 4: Common Report Issues
```php
// Issue: Report empty
// Fix: Check query, filters

// Issue: Report slow
// Fix: Optimize query, add indexes

// Issue: Export not working
// Fix: Check file permissions
```

## Report Debug Checklist

### Investigation
- [ ] Report data tested
- [ ] Query analyzed
- [ ] Permissions checked
- [ ] Export tested

### Resolution
- [ ] Query fixed
- [ ] Indexes added
- [ ] Permissions configured
- [ ] Export working
