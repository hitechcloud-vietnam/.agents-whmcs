# WHMCS Table Styling Workflow

## Purpose
Guide developers through customizing data tables in WHMCS.

## Prerequisites
- WHMCS installation
- CSS/SCSS knowledge
- Bootstrap table components understanding

## Steps

### Phase 1: Table Structure

1. WHMCS table locations
   ```
   Tables in WHMCS:
   - Client area tables
   - Admin area tables
   - Order tables
   - Invoice tables
   - Service tables
   ```

2. Bootstrap table classes
   ```
   Table Classes:
   - .table
   - .table-striped
   - .table-bordered
   - .table-hover
   - .table-dark
   - .table-sm
   ```

### Phase 2: Basic Table Styling

1. Base table styles
   ```css
   .table {
       width: 100%;
       margin-bottom: 1rem;
       color: #212529;
       border-collapse: collapse;
   }
   
   .table th,
   .table td {
       padding: 12px 16px;
       border-top: 1px solid #dee2e6;
       vertical-align: middle;
   }
   
   .table thead th {
       font-weight: 600;
       text-transform: uppercase;
       font-size: 12px;
       letter-spacing: 0.5px;
       color: #495057;
       border-bottom: 2px solid #dee2e6;
       background: #f8f9fa;
   }
   
   .table tbody tr {
       transition: background-color 0.15s;
   }
   ```

### Phase 3: Table Header Styles

1. Modern header
   ```css
   .table thead th {
       background: linear-gradient(135deg, #667eea, #764ba2);
       color: #fff;
       border: none;
       padding: 14px 16px;
       font-weight: 600;
   }
   
   .table thead th:first-child {
       border-radius: 8px 0 0 8px;
   }
   
   .table thead th:last-child {
       border-radius: 0 8px 8px 0;
   }
   ```

2. Dark header
   ```css
   .table-dark-header thead th {
       background: #212529;
       color: #fff;
       border-color: #32383e;
   }
   ```

### Phase 4: Row Styles

1. Striped rows
   ```css
   .table-striped tbody tr:nth-of-type(odd) {
       background-color: #f8f9fa;
   }
   ```

2. Hover rows
   ```css
   .table-hover tbody tr:hover {
       background-color: #e9ecef;
   }
   ```

3. Bordered table
   ```css
   .table-bordered {
       border: 1px solid #dee2e6;
   }
   
   .table-bordered th,
   .table-bordered td {
       border: 1px solid #dee2e6;
   }
   
   .table-bordered thead th,
   .table-bordered thead td {
       border-bottom-width: 2px;
   }
   ```

### Phase 5: Cell Styles

1. Status badges in cells
   ```css
   .table .badge {
       padding: 5px 10px;
       border-radius: 20px;
       font-size: 11px;
       font-weight: 600;
       text-transform: uppercase;
   }
   
   .table .badge-active {
       background: #28a745;
       color: #fff;
   }
   
   .table .badge-pending {
       background: #ffc107;
       color: #212529;
   }
   
   .table .badge-suspended {
       background: #dc3545;
       color: #fff;
   }
   ```

2. Price cells
   ```css
   .table .price {
       font-weight: 700;
       font-size: 16px;
       color: #212529;
   }
   
   .table .price-original {
       text-decoration: line-through;
       color: #6c757d;
       font-weight: 400;
       font-size: 14px;
   }
   ```

3. Action buttons
   ```css
   .table .action-buttons {
       display: flex;
       gap: 8px;
   }
   
   .table .action-buttons .btn {
       padding: 5px 12px;
       font-size: 12px;
   }
   ```

### Phase 6: Responsive Tables

1. Horizontal scroll wrapper
   ```css
   .table-responsive {
       overflow-x: auto;
       -webkit-overflow-scrolling: touch;
   }
   
   @media (max-width: 767px) {
       .table-responsive {
           border: 1px solid #dee2e6;
           border-radius: 8px;
       }
   }
   ```

2. Card view on mobile
   ```css
   @media (max-width: 767px) {
       .table-cards {
           border: none;
       }
       
       .table-cards thead {
           display: none;
       }
       
       .table-cards tbody tr {
           display: block;
           margin-bottom: 16px;
           background: #fff;
           border: 1px solid #dee2e6;
           border-radius: 8px;
           box-shadow: 0 2px 8px rgba(0,0,0,0.05);
       }
       
       .table-cards tbody td {
           display: flex;
           justify-content: space-between;
           align-items: center;
           padding: 12px 16px;
           border: none;
           border-bottom: 1px solid #f0f0f0;
       }
       
       .table-cards tbody td::before {
           content: attr(data-label);
           font-weight: 600;
           color: #6c757d;
           font-size: 12px;
       }
       
       .table-cards tbody td:last-child {
           border-bottom: none;
       }
   }
   ```

### Phase 7: Custom Table Styles

1. Minimal table
   ```css
   .table-minimal {
       border: none;
   }
   
   .table-minimal th,
   .table-minimal td {
       border: none;
       padding: 14px 8px;
   }
   
   .table-minimal thead th {
       background: transparent;
       font-size: 11px;
       text-transform: uppercase;
       color: #adb5bd;
       letter-spacing: 1px;
   }
   
   .table-minimal tbody tr {
       border-bottom: 1px solid #f0f0f0;
   }
   ```

2. Modern data table
   ```css
   .data-table {
       background: #fff;
       border-radius: 12px;
       overflow: hidden;
       box-shadow: 0 2px 15px rgba(0,0,0,0.05);
   }
   
   .data-table thead th {
       background: #fff;
       border-bottom: 2px solid #e9ecef;
       font-weight: 600;
       color: #495057;
       padding: 16px;
   }
   
   .data-table tbody td {
       padding: 16px;
       border-bottom: 1px solid #f0f0f0;
   }
   
   .data-table tbody tr:last-child td {
       border-bottom: none;
   }
   
   .data-table tbody tr:hover {
       background: #fafbfc;
   }
   ```

### Phase 8: Sortable Tables

1. Sortable header styles
   ```css
   .table-sortable thead th {
       cursor: pointer;
       user-select: none;
       position: relative;
   }
   
   .table-sortable thead th::after {
       content: "";
       display: inline-block;
       width: 0;
       height: 0;
       margin-left: 8px;
       vertical-align: middle;
       border-left: 4px solid transparent;
       border-right: 4px solid transparent;
       border-top: 4px solid #adb5bd;
   }
   
   .table-sortable thead th.sort-asc::after {
       border-top: none;
       border-bottom: 4px solid #667eea;
   }
   
   .table-sortable thead th.sort-desc::after {
       border-bottom: none;
       border-top: 4px solid #667eea;
   }
   
   .table-sortable thead th:hover::after {
       border-top-color: #667eea;
   }
   ```

### Phase 9: WHMCS-Specific Tables

1. Invoice table styling
   ```css
   .invoice-items .table th {
       background: #f8f9fa;
       text-align: right;
   }
   
   .invoice-items .table th:first-child {
       text-align: left;
   }
   
   .invoice-items .table td {
       text-align: right;
   }
   
   .invoice-items .table td:first-child {
       text-align: left;
   }
   
   .invoice-summary td {
       font-weight: 600;
       background: #f8f9fa;
   }
   ```

2. Service table
   ```css
   .service-table .service-name {
       font-weight: 600;
   }
   
   .service-table .service-domain {
       color: #6c757d;
       font-size: 13px;
   }
   
   .service-table .service-status {
       font-size: 12px;
       font-weight: 600;
       text-transform: uppercase;
   }
   ```

## Related Workflows
- whmcs-css-customization
- whmcs-form-styling
- whmcs-card-design
- whmcs-responsive-tuning