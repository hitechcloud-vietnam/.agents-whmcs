# WHMCS Custom Fields Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Advanced custom field management with dynamic fields, conditional logic, and validation.

## Database Schema

```php
<?php
// modules/addons/custom_fields/custom_fields.php

use WHMCS\Database\Capsule;

function custom_fields_config(): array {
    return [
        'name' => 'Custom Fields',
        'description' => 'Advanced custom field management',
        'version' => '1.0',
    ];
}

function custom_fields_activate(): array {
    Capsule::schema()->create('mod_custom_field_groups', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('description')->nullable();
        $t->string('entity_type', 50);
        $t->integer('sort_order')->default(0);
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_custom_field_defs', function($t) {
        $t->increments('id');
        $t->integer('group_id')->unsigned();
        $t->string('field_name', 100);
        $t->string('field_key', 50)->unique();
        $t->string('field_type', 30);
        $t->text('description')->nullable();
        $t->string('validation_type', 50)->nullable();
        $t->text('validation_rules')->nullable();
        $t->string('default_value')->nullable();
        $t->text('options')->nullable();
        $t->boolean('is_required')->default(false);
        $t->boolean('is_unique')->default(false);
        $t->boolean('show_on_invoice')->default(false);
        $t->boolean('show_on_clientarea')->default(true);
        $t->integer('sort_order')->default(0);
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_field_conditions', function($t) {
        $t->increments('id');
        $t->integer('field_id')->unsigned();
        $t->string('condition_type', 30);
        $t->string('trigger_field', 50);
        $t->string('operator', 30);
        $t->string('value', 255);
        $t->string('action', 30)->default('show');
        $t->boolean('is_active')->default(true);
    });

    Capsule::schema()->create('mod_field_dependencies', function($t) {
        $t->increments('id');
        $t->integer('field_id')->unsigned();
        $t->integer('depends_on_field_id')->unsigned();
        $t->string('depends_on_value', 255)->nullable();
        $t->boolean('hide_when_hidden')->default(true);
    });

    Capsule::schema()->create('mod_field_values', function($t) {
        $t->increments('id');
        $t->integer('field_id')->unsigned();
        $t->integer('rel_id')->unsigned();
        $t->text('field_value')->nullable();
        $t->timestamp('created_at')->useCurrent();
        $t->timestamp('updated_at')->useCurrent();

        $t->unique(['field_id', 'rel_id']);
    });

    Capsule::schema()->create('mod_field_templates', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('description')->nullable();
        $t->text('fields')->nullable();
        $t->string('entity_type', 50);
        $t->boolean('is_active')->default(true);
        $t->timestamps();
    });

    return ['status' => 'success'];
}

function custom_fields_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_field_templates');
    Capsule::schema()->dropIfExists('mod_field_values');
    Capsule::schema()->dropIfExists('mod_field_dependencies');
    Capsule::schema()->dropIfExists('mod_field_conditions');
    Capsule::schema()->dropIfExists('mod_custom_field_defs');
    Capsule::schema()->dropIfExists('mod_custom_field_groups');
    return ['status' => 'success'];
}
```

## Custom Field Manager

```php
<?php
class CustomFieldManager {
    public function createField(array $fieldData): int {
        $fieldKey = $fieldData['field_key'] ?? $this->generateFieldKey($fieldData['field_name']);

        return Capsule::table('mod_custom_field_defs')->insertGetId([
            'group_id' => $fieldData['group_id'],
            'field_name' => $fieldData['field_name'],
            'field_key' => $fieldKey,
            'field_type' => $fieldData['type'],
            'description' => $fieldData['description'] ?? null,
            'validation_type' => $fieldData['validation_type'] ?? null,
            'validation_rules' => isset($fieldData['validation_rules']) ? json_encode($fieldData['validation_rules']) : null,
            'default_value' => $fieldData['default_value'] ?? null,
            'options' => isset($fieldData['options']) ? json_encode($fieldData['options']) : null,
            'is_required' => $fieldData['is_required'] ?? false,
            'is_unique' => $fieldData['is_unique'] ?? false,
            'show_on_invoice' => $fieldData['show_on_invoice'] ?? false,
            'show_on_clientarea' => $fieldData['show_on_clientarea'] ?? true,
            'sort_order' => $fieldData['sort_order'] ?? 0,
        ]);
    }

    private function generateFieldKey(string $name): string {
        $key = preg_replace('/[^a-zA-Z0-9]/', '', $name);
        $key = strtolower(substr($key, 0, 30));

        $exists = Capsule::table('mod_custom_field_defs')
            ->where('field_key', $key)
            ->exists();

        if ($exists) {
            $key = $key . '_' . time();
        }

        return $key;
    }

    public function getField(int $fieldId): ?object {
        return Capsule::table('mod_custom_field_defs')->find($fieldId);
    }

    public function getFieldsByGroup(int $groupId): array {
        return Capsule::table('mod_custom_field_defs')
            ->where('group_id', $groupId)
            ->where('is_active', 1)
            ->orderBy('sort_order')
            ->get();
    }

    public function getFieldsForEntity(string $entityType, int $relId): array {
        $group = Capsule::table('mod_custom_field_groups')
            ->where('entity_type', $entityType)
            ->where('is_active', 1)
            ->first();

        if (!$group) {
            return [];
        }

        $fields = $this->getFieldsByGroup($group->id);

        foreach ($fields as $field) {
            $field->value = $this->getFieldValue($field->id, $relId);
            $field->options = json_decode($field->options, true);
        }

        return $fields;
    }

    public function saveFieldValue(int $fieldId, int $relId, $value): array {
        $field = $this->getField($fieldId);

        if (!$field) {
            return ['success' => false, 'error' => 'Field not found'];
        }

        // Validate value
        $validation = $this->validateValue($field, $value);
        if (!$validation['valid']) {
            return $validation;
        }

        // Check uniqueness
        if ($field->is_unique && !$this->checkUniqueness($fieldId, $relId, $value)) {
            return ['success' => false, 'error' => 'This value is already in use'];
        }

        $existing = Capsule::table('mod_field_values')
            ->where('field_id', $fieldId)
            ->where('rel_id', $relId)
            ->first();

        if ($existing) {
            Capsule::table('mod_field_values')
                ->where('id', $existing->id)
                ->update([
                    'field_value' => is_array($value) ? json_encode($value) : $value,
                    'updated_at' => date('Y-m-d H:i:s'),
                ]);
        } else {
            Capsule::table('mod_field_values')->insert([
                'field_id' => $fieldId,
                'rel_id' => $relId,
                'field_value' => is_array($value) ? json_encode($value) : $value,
            ]);
        }

        return ['success' => true];
    }

    public function getFieldValue(int $fieldId, int $relId) {
        $record = Capsule::table('mod_field_values')
            ->where('field_id', $fieldId)
            ->where('rel_id', $relId)
            ->first();

        if (!$record) {
            $field = $this->getField($fieldId);
            return $field->default_value ?? null;
        }

        $field = $this->getField($fieldId);

        if ($field->field_type === 'multiselect' || $field->field_type === 'checkbox') {
            return json_decode($record->field_value, true) ?? [];
        }

        return $record->field_value;
    }

    private function validateValue(object $field, $value): array {
        // Required check
        if ($field->is_required && empty($value)) {
            return ['valid' => false, 'error' => "{$field->field_name} is required"];
        }

        if (is_null($value) || $value === '') {
            return ['valid' => true];
        }

        // Type-specific validation
        switch ($field->field_type) {
            case 'email':
                if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
                    return ['valid' => false, 'error' => 'Invalid email address'];
                }
                break;

            case 'url':
                if (!filter_var($value, FILTER_VALIDATE_URL)) {
                    return ['valid' => false, 'error' => 'Invalid URL'];
                }
                break;

            case 'phone':
                if (!preg_match('/^[\d\s\-\+\(\)]+$/', $value)) {
                    return ['valid' => false, 'error' => 'Invalid phone number'];
                }
                break;

            case 'number':
                if (!is_numeric($value)) {
                    return ['valid' => false, 'error' => 'Must be a number'];
                }
                break;

            case 'date':
                if (!strtotime($value)) {
                    return ['valid' => false, 'error' => 'Invalid date'];
                }
                break;
        }

        // Custom validation rules
        $rules = json_decode($field->validation_rules, true);
        if ($rules) {
            foreach ($rules as $rule) {
                $result = $this->applyValidationRule($rule, $value);
                if (!$result['valid']) {
                    return $result;
                }
            }
        }

        return ['valid' => true];
    }

    private function applyValidationRule(array $rule, $value): array {
        switch ($rule['type']) {
            case 'min_length':
                if (strlen($value) < $rule['value']) {
                    return ['valid' => false, 'error' => "Minimum {$rule['value']} characters required"];
                }
                break;

            case 'max_length':
                if (strlen($value) > $rule['value']) {
                    return ['valid' => false, 'error' => "Maximum {$rule['value']} characters allowed"];
                }
                break;

            case 'min_value':
                if (is_numeric($value) && $value < $rule['value']) {
                    return ['valid' => false, 'error' => "Minimum value is {$rule['value']}"];
                }
                break;

            case 'max_value':
                if (is_numeric($value) && $value > $rule['value']) {
                    return ['valid' => false, 'error' => "Maximum value is {$rule['value']}"];
                }
                break;

            case 'pattern':
                if (!preg_match($rule['value'], $value)) {
                    return ['valid' => false, 'error' => $rule['message'] ?? 'Invalid format'];
                }
                break;

            case 'in_list':
                $options = $rule['values'] ?? [];
                if (!in_array($value, $options)) {
                    return ['valid' => false, 'error' => 'Invalid selection'];
                }
                break;
        }

        return ['valid' => true];
    }

    private function checkUniqueness(int $fieldId, int $relId, $value): bool {
        $existing = Capsule::table('mod_field_values')
            ->where('field_id', $fieldId)
            ->where('rel_id', '!=', $relId)
            ->where('field_value', $value)
            ->first();

        return !$existing;
    }

    public function addCondition(int $fieldId, array $condition): int {
        return Capsule::table('mod_field_conditions')->insertGetId([
            'field_id' => $fieldId,
            'condition_type' => $condition['type'] ?? 'show',
            'trigger_field' => $condition['trigger_field'],
            'operator' => $condition['operator'],
            'value' => $condition['value'],
            'action' => $condition['action'] ?? 'show',
        ]);
    }

    public function getConditionalVisibility(int $fieldId, array $allValues): array {
        $conditions = Capsule::table('mod_field_conditions')
            ->where('field_id', $fieldId)
            ->where('is_active', 1)
            ->get();

        if ($conditions->isEmpty()) {
            return ['visible' => true];
        }

        $allVisible = true;
        $anyVisible = false;

        foreach ($conditions as $condition) {
            $triggerValue = $allValues[$condition->trigger_field] ?? null;
            $conditionMet = $this->evaluateCondition($condition, $triggerValue);

            if ($condition->action === 'show') {
                if ($conditionMet) {
                    $anyVisible = true;
                } else {
                    $allVisible = false;
                }
            }
        }

        return [
            'visible' => $allVisible && $anyVisible,
            'reason' => !$allVisible ? 'Conditions not met' : ($anyVisible ? null : 'Waiting for trigger'),
        ];
    }

    private function evaluateCondition(object $condition, $value): bool {
        $expectedValue = $condition->value;

        switch ($condition->operator) {
            case 'equals':
                return $value == $expectedValue;
            case 'not_equals':
                return $value != $expectedValue;
            case 'contains':
                return strpos($value, $expectedValue) !== false;
            case 'starts_with':
                return strpos($value, $expectedValue) === 0;
            case 'ends_with':
                return substr($value, -strlen($expectedValue)) === $expectedValue;
            case 'greater_than':
                return $value > $expectedValue;
            case 'less_than':
                return $value < $expectedValue;
            case 'is_empty':
                return empty($value);
            case 'is_not_empty':
                return !empty($value);
            case 'in':
                $options = explode(',', $expectedValue);
                return in_array($value, $options);
            default:
                return false;
        }
    }

    public function renderFieldHtml(object $field, $value = null): string {
        $options = $field->options ?? [];
        $required = $field->is_required ? ' required' : '';
        $name = "custom_field[{$field->id}]";
        $id = "field_{$field->id}";

        switch ($field->field_type) {
            case 'text':
                return '<input type="text" name="' . $name . '" id="' . $id . '" class="form-control"' . $required .
                       ' value="' . htmlspecialchars($value ?? '') . '">';

            case 'textarea':
                return '<textarea name="' . $name . '" id="' . $id . '" class="form-control"' . $required .
                       ' rows="4">' . htmlspecialchars($value ?? '') . '</textarea>';

            case 'select':
                $html = '<select name="' . $name . '" id="' . $id . '" class="form-control"' . $required . '>';
                $html .= '<option value="">Select...</option>';
                foreach ($options as $opt) {
                    $selected = ($value === $opt['value']) ? ' selected' : '';
                    $html .= '<option value="' . htmlspecialchars($opt['value']) . '"' . $selected . '>' .
                             htmlspecialchars($opt['label']) . '</option>';
                }
                $html .= '</select>';
                return $html;

            case 'multiselect':
                $currentValues = is_array($value) ? $value : [];
                $html = '<select name="' . $name . '[]" id="' . $id . '" class="form-control" multiple' . $required . '>';
                foreach ($options as $opt) {
                    $selected = in_array($opt['value'], $currentValues) ? ' selected' : '';
                    $html .= '<option value="' . htmlspecialchars($opt['value']) . '"' . $selected . '>' .
                             htmlspecialchars($opt['label']) . '</option>';
                }
                $html .= '</select>';
                return $html;

            case 'radio':
                $html = '';
                foreach ($options as $opt) {
                    $checked = ($value === $opt['value']) ? ' checked' : '';
                    $html .= '<div class="radio"><label>';
                    $html .= '<input type="radio" name="' . $name . '" value="' . htmlspecialchars($opt['value']) . '"' .
                             $checked . '> ' . htmlspecialchars($opt['label']);
                    $html .= '</label></div>';
                }
                return $html;

            case 'checkbox':
                $currentValues = is_array($value) ? $value : [];
                $html = '';
                foreach ($options as $opt) {
                    $checked = in_array($opt['value'], $currentValues) ? ' checked' : '';
                    $html .= '<div class="checkbox"><label>';
                    $html .= '<input type="checkbox" name="' . $name . '[]" value="' . htmlspecialchars($opt['value']) . '"' .
                             $checked . '> ' . htmlspecialchars($opt['label']);
                    $html .= '</label></div>';
                }
                return $html;

            case 'date':
                return '<input type="date" name="' . $name . '" id="' . $id . '" class="form-control datepicker"' . $required .
                       ' value="' . htmlspecialchars($value ?? '') . '">';

            case 'datetime':
                return '<input type="datetime-local" name="' . $name . '" id="' . $id . '" class="form-control"' . $required .
                       ' value="' . htmlspecialchars($value ?? '') . '">';

            case 'number':
                return '<input type="number" name="' . $name . '" id="' . $id . '" class="form-control"' . $required .
                       ' value="' . htmlspecialchars($value ?? '') . '">';

            case 'email':
                return '<input type="email" name="' . $name . '" id="' . $id . '" class="form-control"' . $required .
                       ' value="' . htmlspecialchars($value ?? '') . '">';

            case 'phone':
                return '<input type="tel" name="' . $name . '" id="' . $id . '" class="form-control"' . $required .
                       ' value="' . htmlspecialchars($value ?? '') . '">';

            case 'url':
                return '<input type="url" name="' . $name . '" id="' . $id . '" class="form-control"' . $required .
                       ' value="' . htmlspecialchars($value ?? '') . '">';

            case 'file':
                return '<input type="file" name="' . $name . '" id="' . $id . '" class="form-control"' . $required .
                       ' accept="' . ($field->accept ?? '*') . '">';

            default:
                return '<input type="text" name="' . $name . '" id="' . $id . '" class="form-control"' . $required .
                       ' value="' . htmlspecialchars($value ?? '') . '">';
        }
    }
}
```

## Form Integration

```php
<?php
add_hook('ClientAreaPageProductDetails', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $fieldManager = new CustomFieldManager();

    $fields = $fieldManager->getFieldsForEntity('service', $serviceId);

    // Apply conditional visibility
    foreach ($fields as $field) {
        $allValues = [];
        foreach ($fields as $f) {
            $allValues[$f->field_key] = $f->value;
        }

        $visibility = $fieldManager->getConditionalVisibility($field->id, $allValues);
        $field->is_visible = $visibility['visible'];
    }

    return ['custom_fields' => $fields];
});
```

---

**Related Skills:**
- whmcs-product-configurator
- whmcs-client-management
- whmcs-data-export