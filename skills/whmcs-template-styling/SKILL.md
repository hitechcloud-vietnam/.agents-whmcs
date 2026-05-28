# WHMCS Template Styling Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for styling WHMCS module templates with proper CSS, responsive design, and WHMCS compatibility.

## When to Use

- Creating admin templates for addon modules
- Building client area templates for services
- Styling forms and tables

## Template Structure

```
modules/addons/{module}/templates/
├── admin/
│   ├── index.tpl
│   ├── dashboard.tpl
│   ├── settings.tpl
│   └── logs.tpl
└── clientarea/
    ├── overview.tpl
    ├── manage.tpl
    └── dashboard.tpl
```

## Smarty Template Basics

```smarty
{* Comment *}

{* Variable output with escape *}
{$variable|escape:'html'}
{$variable|default:'fallback'}
{$variable|upper}
{$variable|lower}
{$variable|date_format:'%Y-%m-%d'}

{* Loop *}
{foreach $items as $item}
    <div class="item">{$item.name}</div>
{/foreach}

{* Conditional *}
{if $status eq 'active'}
    <span class="badge badge-success">Active</span>
{elseif $status eq 'suspended'}
    <span class="badge badge-warning">Suspended</span>
{else}
    <span class="badge badge-secondary">{$status}</span>
{/if}

{* Include partial *}
{include file="$templatepath/partial.tpl"}
```

## Basic Template Example

```smarty
<div class="module-container">
    <div class="module-header">
        <h2>{$module_title}</h2>
        <span class="badge badge-{$status|lower}">{$status}</span>
    </div>

    <div class="module-content">
        <div class="info-grid">
            {foreach $details as $key => $value}
            <div class="info-row">
                <span class="label">{$key|escape:'html'}:</span>
                <span class="value">{$value|escape:'html'}</span>
            </div>
            {/foreach}
        </div>

        <div class="actions">
            <a href="?action=edit" class="btn btn-primary">
                <i class="fa fa-edit"></i> Edit
            </a>
            <a href="?action=delete" class="btn btn-danger"
               onclick="return confirm('Are you sure?')">
                <i class="fa fa-trash"></i> Delete
            </a>
        </div>
    </div>
</div>
```

## CSS Styling

```css
/* Container */
.module-container {
    padding: 20px;
    background: #fff;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

/* Header */
.module-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding-bottom: 15px;
    border-bottom: 1px solid #eee;
    margin-bottom: 20px;
}

.module-header h2 {
    margin: 0;
    font-size: 24px;
    color: #333;
}

/* Info Grid */
.info-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 15px;
    margin: 20px 0;
}

.info-row {
    display: flex;
    justify-content: space-between;
    padding: 12px 15px;
    background: #f8f9fa;
    border-radius: 4px;
}

.info-row .label {
    font-weight: 600;
    color: #666;
}

.info-row .value {
    color: #333;
}

/* Buttons */
.btn {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 10px 20px;
    border-radius: 4px;
    font-size: 14px;
    font-weight: 500;
    text-decoration: none;
    cursor: pointer;
    border: none;
    transition: all 0.2s;
}

.btn-primary {
    background: #007bff;
    color: #fff;
}

.btn-primary:hover {
    background: #0056b3;
}

.btn-secondary {
    background: #6c757d;
    color: #fff;
}

.btn-danger {
    background: #dc3545;
    color: #fff;
}

/* Badges */
.badge {
    padding: 4px 12px;
    border-radius: 20px;
    font-size: 12px;
    font-weight: 600;
}

.badge-success { background: #28a745; color: #fff; }
.badge-warning { background: #ffc107; color: #000; }
.badge-danger { background: #dc3545; color: #fff; }
.badge-info { background: #17a2b8; color: #fff; }
.badge-secondary { background: #6c757d; color: #fff; }

/* Tables */
.table-container {
    overflow-x: auto;
}

.data-table {
    width: 100%;
    border-collapse: collapse;
}

.data-table th,
.data-table td {
    padding: 12px 15px;
    text-align: left;
    border-bottom: 1px solid #eee;
}

.data-table th {
    background: #f8f9fa;
    font-weight: 600;
    color: #333;
}

.data-table tr:hover {
    background: #f8f9fa;
}

/* Forms */
.form-group {
    margin-bottom: 20px;
}

.form-group label {
    display: block;
    margin-bottom: 8px;
    font-weight: 500;
    color: #333;
}

.form-control {
    width: 100%;
    padding: 10px 15px;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-size: 14px;
}

.form-control:focus {
    outline: none;
    border-color: #007bff;
    box-shadow: 0 0 0 3px rgba(0,123,255,0.1);
}

/* Tabs */
.whmcs-tabs {
    display: flex;
    gap: 10px;
    border-bottom: 2px solid #eee;
    margin-bottom: 20px;
}

.whmcs-tabs a {
    padding: 10px 20px;
    color: #666;
    text-decoration: none;
    border-bottom: 2px solid transparent;
    transition: all 0.2s;
}

.whmcs-tabs a:hover {
    color: #007bff;
}

.whmcs-tabs a.active {
    color: #007bff;
    border-bottom-color: #007bff;
}
```

## Responsive Design

```css
/* Mobile */
@media (max-width: 768px) {
    .module-header {
        flex-direction: column;
        align-items: flex-start;
        gap: 10px;
    }

    .info-grid {
        grid-template-columns: 1fr;
    }

    .data-table {
        font-size: 12px;
    }

    .data-table th,
    .data-table td {
        padding: 8px;
    }

    .btn {
        padding: 8px 15px;
        font-size: 12px;
    }
}

/* Tablet */
@media (min-width: 769px) and (max-width: 1024px) {
    .info-grid {
        grid-template-columns: repeat(2, 1fr);
    }
}
```

## Checklist

- [ ] All output escaped with |escape:'html'
- [ ] CSS in separate file or style block
- [ ] Responsive design for mobile
- [ ] Consistent spacing and typography
- [ ] Loading states for async operations
- [ ] Error states displayed correctly

---

**Related Skills:**
- whmcs-clientarea-builder
- whmcs-ajax-patterns
- whmcs-admin-ui