# WHMCS Service Selection Wizard Configuration Workflow

## Overview
Comprehensive workflow for creating step-by-step service selection wizards.

## Prerequisites
- WHMCS v8.0+
- Order form template

## Step-by-Step Guide

### Step 1: Create Wizard Steps
```php
<?php
class ServiceWizard
{
    protected array $steps = [
        ['id' => 'plan', 'title' => 'Choose Plan', 'required' => true],
        ['id' => 'domain', 'title' => 'Select Domain', 'required' => true],
        ['id' => 'options', 'title' => 'Configure Options', 'required' => false],
        ['id' => 'checkout', 'title' => 'Checkout', 'required' => true],
    ];

    public function getStep(int $currentStep): ?array
    {
        return $this->steps[$currentStep] ?? null;
    }

    public function validateStep(int $stepId, array $data): bool
    {
        $step = array_filter($this->steps, fn($s) => $s['id'] === $stepId);
        
        if (empty($step)) return false;
        
        $step = reset($step);
        
        if ($step['required'] && empty($data)) {
            return false;
        }
        
        return true;
    }
}
```

### Step 2: Wizard Controller
```php
<?php
add_hook('ClientAreaPage', 1, function($vars) {
    if ($vars['filename'] === 'cart' && $_GET['a'] === 'wizard') {
        $wizard = new ServiceWizard();
        $step = $_SESSION['wizard_step'] ?? 0;
        
        return [
            'wizard_step' => $wizard->getStep($step),
            'wizard_steps' => $wizard->steps,
        ];
    }
});
```

## Checklist
- Wizard steps defined
- Validation rules set
- Progress tracking enabled
- Mobile responsive tested
