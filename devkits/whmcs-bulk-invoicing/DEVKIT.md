# WHMCS Bulk Invoice Generation Devkit

## Overview
A bulk invoice generation system for WHMCS that supports batch processing, selective invoicing, invoice templates, and export capabilities for mass billing operations.

## Features
- Bulk invoice generation
- Selective service invoicing
- Invoice template management
- CSV/Excel import for billing
- Preview before generation
- Batch status tracking
- Invoice export (PDF, CSV, XML)
- Grouped billing options
- Prorate calculations for bulk
- Custom line items
- Tax handling per invoice
- Aging analysis reports

## WHMCS Integration Points
- Module: addon/BulkInvoicing
- Hook: OrderPaid
- Admin area integration
- Queue processing

## Database Schema
```sql
CREATE TABLE mod_bulk_invoice_jobs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    job_name VARCHAR(255),
    created_by INT NOT NULL,
    status ENUM('pending', 'processing', 'completed', 'failed', 'cancelled') DEFAULT 'pending',
    total_invoices INT DEFAULT 0,
    processed_count INT DEFAULT 0,
    generated_invoice_ids JSON,
    errors JSON,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_bulk_invoice_criteria (
    id INT AUTO_INCREMENT PRIMARY KEY,
    job_id INT NOT NULL,
    criteria_type ENUM('product', 'billing_cycle', 'date_range', 'amount_range', 'client_group') NOT NULL,
    criteria_value JSON NOT NULL,
    action ENUM('include', 'exclude') DEFAULT 'include'
);

CREATE TABLE mod_bulk_invoice_templates (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    template_data JSON NOT NULL,
    is_default TINYINT(1) DEFAULT 0,
    created_by INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## API Endpoints
- POST /api/bulk-invoice/jobs - Create job
- GET /api/bulk-invoice/jobs/:id - Get job status
- POST /api/bulk-invoice/jobs/:id/process - Process job
- GET /api/bulk-invoice/jobs/:id/invoices - Get generated invoices
- DELETE /api/bulk-invoice/jobs/:id - Cancel job

## Testing Checklist
- [ ] Bulk generation speed
- [ ] Selective filtering
- [ ] Template application
- [ ] Export formats
- [ ] Error handling
- [ ] Progress tracking

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
