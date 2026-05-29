# WHMCS Custom Invoice Templates Devkit

## Overview
A comprehensive invoice template system for WHMCS with custom branding, multiple templates, template preview, and WYSIWYG editing for professional invoice presentation.

## Features
- Custom invoice templates
- WYSIWYG template editor
- Template preview
- Multiple template support
- Brand customization
- Logo management
- Color scheme configuration
- Font customization
- RTL language support
- Template variables
- Conditional content
- PDF generation options

## WHMCS Integration Points
- Module: addon/InvoiceTemplates
- Hook: InvoiceCreation
- Hook: InvoiceView
- Admin configuration

## Database Schema
```sql
CREATE TABLE mod_invoice_templates (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    template_html TEXT NOT NULL,
    template_css TEXT,
    logo_url VARCHAR(500),
    color_scheme VARCHAR(50),
    is_default TINYINT(1) DEFAULT 0,
    is_active TINYINT(1) DEFAULT 1,
    created_by INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE mod_invoice_template_variables (
    id INT AUTO_INCREMENT PRIMARY KEY,
    template_id INT NOT NULL,
    variable_name VARCHAR(100) NOT NULL,
    variable_value TEXT,
    is_custom TINYINT(1) DEFAULT 1
);

CREATE TABLE mod_invoice_template_assignments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    template_id INT NOT NULL,
    assign_type ENUM('client', 'product', 'billing_cycle') NOT NULL,
    assign_value VARCHAR(255) NOT NULL,
    priority INT DEFAULT 0
);
```

## API Endpoints
- POST /api/invoice-templates - Create template
- GET /api/invoice-templates - List templates
- GET /api/invoice-templates/:id - Get template
- PUT /api/invoice-templates/:id - Update template
- POST /api/invoice-templates/:id/preview - Preview template
- DELETE /api/invoice-templates/:id - Delete template

## Testing Checklist
- [ ] Template rendering
- [ ] WYSIWYG editor functionality
- [ ] PDF generation
- [ ] Logo display
- [ ] Variable substitution
- [ ] Conditional content

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
