# WHMCS Widget Styling Workflow

## Purpose
Guide developers through customizing widget components in WHMCS.

## Prerequisites
- WHMCS installation
- CSS/SCSS knowledge
- HTML structure understanding

## Steps

### Phase 1: Widget Structure

1. Widget locations
   ```
   WHMCS Widgets:
   - Dashboard widgets
   - Sidebar widgets
   - Client area widgets
   - Home page widgets
   ```

2. Common widget types
   ```
   Widget Types:
   ├── Info widgets
   ├── Status widgets
   ├── Quick action widgets
   ├── Chart widgets
   └── List widgets
   ```

### Phase 2: Basic Widget Styling

1. Widget container styles
   ```css
   .widget {
       background: #fff;
       border-radius: 12px;
       padding: 20px;
       box-shadow: 0 2px 15px rgba(0,0,0,0.05);
       margin-bottom: 20px;
   }
   
   .widget-header {
       display: flex;
       justify-content: space-between;
       align-items: center;
       margin-bottom: 20px;
   }
   
   .widget-title {
       font-size: 16px;
       font-weight: 600;
       color: #212529;
       margin: 0;
   }
   
   .widget-actions {
       display: flex;
       gap: 10px;
   }
   
   .widget-body {
       /* Widget content area */
   }
   ```

### Phase 3: Info Widget

1. Info widget design
   ```css
   .widget-info {
       text-align: center;
       padding: 30px 20px;
   }
   
   .widget-info .widget-icon {
       width: 60px;
       height: 60px;
       border-radius: 50%;
       background: linear-gradient(135deg, #667eea, #764ba2);
       color: #fff;
       display: flex;
       align-items: center;
       justify-content: center;
       font-size: 28px;
       margin: 0 auto 20px;
   }
   
   .widget-info .widget-value {
       font-size: 36px;
       font-weight: 700;
       color: #212529;
       margin-bottom: 8px;
   }
   
   .widget-info .widget-label {
       font-size: 14px;
       color: #6c757d;
       margin-bottom: 15px;
   }
   
   .widget-info .widget-trend {
       font-size: 13px;
       font-weight: 500;
   }
   
   .widget-info .widget-trend.up {
       color: #28a745;
   }
   
   .widget-info .widget-trend.down {
       color: #dc3545;
   }
   ```

### Phase 4: Status Widget

1. Status widget design
   ```css
   .widget-status {
       display: flex;
       align-items: center;
       padding: 15px;
       background: #f8f9fa;
       border-radius: 8px;
       margin-bottom: 10px;
   }
   
   .widget-status .status-indicator {
       width: 10px;
       height: 10px;
       border-radius: 50%;
       margin-right: 12px;
   }
   
   .widget-status .status-indicator.online {
       background: #28a745;
   }
   
   .widget-status .status-indicator.offline {
       background: #dc3545;
   }
   
   .widget-status .status-indicator.pending {
       background: #ffc107;
   }
   
   .widget-status .status-info {
       flex: 1;
   }
   
   .widget-status .status-name {
       font-weight: 500;
       color: #212529;
       font-size: 14px;
   }
   
   .widget-status .status-detail {
       font-size: 12px;
       color: #6c757d;
   }
   
   .widget-status .status-badge {
       padding: 4px 10px;
       border-radius: 15px;
       font-size: 11px;
       font-weight: 600;
       text-transform: uppercase;
   }
   ```

### Phase 5: Quick Action Widget

1. Quick action widget
   ```css
   .widget-quick-actions {
       display: grid;
       grid-template-columns: repeat(2, 1fr);
       gap: 10px;
   }
   
   .quick-action-btn {
       display: flex;
       flex-direction: column;
       align-items: center;
       padding: 20px 15px;
       background: #f8f9fa;
       border-radius: 10px;
       color: #495057;
       text-decoration: none;
       transition: all 0.2s;
   }
   
   .quick-action-btn:hover {
       background: linear-gradient(135deg, #667eea, #764ba2);
       color: #fff;
       transform: translateY(-2px);
   }
   
   .quick-action-btn i {
       font-size: 24px;
       margin-bottom: 10px;
   }
   
   .quick-action-btn span {
       font-size: 13px;
       font-weight: 500;
   }
   ```

### Phase 6: List Widget

1. List widget design
   ```css
   .widget-list {
       padding: 0;
       margin: 0;
       list-style: none;
   }
   
   .widget-list-item {
       display: flex;
       align-items: center;
       padding: 15px 0;
       border-bottom: 1px solid #f0f0f0;
   }
   
   .widget-list-item:last-child {
       border-bottom: none;
   }
   
   .widget-list-item .item-icon {
       width: 40px;
       height: 40px;
       border-radius: 8px;
       background: #f0f0f0;
       display: flex;
       align-items: center;
       justify-content: center;
       margin-right: 15px;
       color: #495057;
   }
   
   .widget-list-item .item-content {
       flex: 1;
   }
   
   .widget-list-item .item-title {
       font-weight: 500;
       color: #212529;
       font-size: 14px;
       margin-bottom: 3px;
   }
   
   .widget-list-item .item-subtitle {
       font-size: 12px;
       color: #6c757d;
   }
   
   .widget-list-item .item-action {
       font-size: 13px;
       color: #667eea;
       text-decoration: none;
   }
   ```

### Phase 7: Chart Widget

1. Chart widget styles
   ```css
   .widget-chart {
       position: relative;
   }
   
   .widget-chart canvas {
       max-width: 100%;
   }
   
   .widget-chart .chart-legend {
       display: flex;
       justify-content: center;
       gap: 20px;
       margin-top: 15px;
   }
   
   .widget-chart .legend-item {
       display: flex;
       align-items: center;
       font-size: 12px;
       color: #6c757d;
   }
   
   .widget-chart .legend-color {
       width: 12px;
       height: 12px;
       border-radius: 3px;
       margin-right: 8px;
   }
   ```

### Phase 8: Widget Variants

1. Compact widget
   ```css
   .widget-compact {
       padding: 15px;
   }
   
   .widget-compact .widget-header {
       margin-bottom: 15px;
   }
   
   .widget-compact .widget-title {
       font-size: 13px;
   }
   ```

2. Bordered widget
   ```css
   .widget-bordered {
       border: 1px solid #dee2e6;
       box-shadow: none;
   }
   
   .widget-bordered .widget-header {
       padding-bottom: 15px;
       border-bottom: 1px solid #dee2e6;
   }
   ```

3. Flat widget
   ```css
   .widget-flat {
       background: #f8f9fa;
       border-radius: 8px;
       box-shadow: none;
   }
   ```

## Related Workflows
- whmcs-card-design
- whmcs-sidebar-modification
- whmcs-css-customization
- whmcs-widget-styling