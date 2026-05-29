# WHMCS Helpdesk Sidebar Widget Devkit

## Overview
A helpdesk sidebar widget for WHMCS providing quick access to support tickets, knowledge base, live chat, and self-service tools from any page.

## Features
- Collapsible sidebar widget
- Quick ticket creation
- Ticket status overview
- Recent tickets list
- Knowledge base search
- Live chat launcher
- FAQ display
- Quick actions
- Unread notifications
- Customer satisfaction rating
- Multi-language support
- Customizable appearance

## WHMCS Integration Points
- Module: addon/HelpdeskWidget
- Hook: ClientAreaPage
- Hook: PageHead
- JavaScript widget injection

## Database Schema
```sql
CREATE TABLE mod_helpdesk_widget_settings (
    id INT AUTO_INCREMENT PRIMARY KEY,
    position ENUM('left', 'right') DEFAULT 'right',
    offset_vertical INT DEFAULT 100,
    offset_horizontal INT DEFAULT 20,
    is_expanded TINYINT(1) DEFAULT 0,
    default_tab VARCHAR(50) DEFAULT 'tickets',
    show_knowledge_base TINYINT(1) DEFAULT 1,
    show_live_chat TINYINT(1) DEFAULT 1,
    show_quick_actions TINYINT(1) DEFAULT 1,
    color_scheme VARCHAR(20) DEFAULT '#007bff',
    widget_title VARCHAR(100) DEFAULT 'Support'
);

CREATE TABLE mod_helpdesk_widget_tabs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    tab_name VARCHAR(50) NOT NULL,
    tab_icon VARCHAR(50),
    tab_order INT DEFAULT 0,
    is_enabled TINYINT(1) DEFAULT 1,
    content_type ENUM('tickets', 'kb', 'chat', 'faq', 'custom') DEFAULT 'tickets'
);
```

## Testing Checklist
- [ ] Widget display
- [ ] Ticket quick view
- [ ] Knowledge base integration
- [ ] Live chat integration
- [ ] Mobile responsiveness

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
