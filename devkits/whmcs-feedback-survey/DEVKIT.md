# WHMCS Post-Ticket Satisfaction Survey Devkit

## Overview
A post-ticket satisfaction survey system for WHMCS that sends surveys, collects feedback, analyzes responses, and generates reports.

## Features
- Post-ticket survey triggers
- Custom survey templates
- Multi-question surveys
- Rating scales (NPS, CSAT, CES)
- Comment collection
- Survey scheduling
- Response tracking
- Automated follow-ups
- Feedback analysis
- Report generation
- Survey analytics
- Integration with CRM

## WHMCS Integration Points
- Module: addon/FeedbackSurvey
- Hook: TicketClose
- Hook: TicketStatusChange

## Database Schema
```sql
CREATE TABLE mod_feedback_surveys (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    template_data JSON NOT NULL,
    trigger_type ENUM('ticket_close', 'manual', 'time_delay') DEFAULT 'ticket_close',
    delay_hours INT DEFAULT 1,
    reminder_enabled TINYINT(1) DEFAULT 1,
    reminder_hours INT DEFAULT 48,
    max_reminders INT DEFAULT 1,
    is_active TINYINT(1) DEFAULT 1
);

CREATE TABLE mod_survey_questions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    survey_id INT NOT NULL,
    question_type ENUM('rating', 'nps', 'multiple_choice', 'text', 'emoji') NOT NULL,
    question_text TEXT NOT NULL,
    options JSON,
    is_required TINYINT(1) DEFAULT 1,
    sort_order INT DEFAULT 0
);

CREATE TABLE mod_survey_responses (
    id INT AUTO_INCREMENT PRIMARY KEY,
    survey_id INT NOT NULL,
    ticket_id INT NOT NULL,
    client_id INT,
    responses JSON NOT NULL,
    nps_score INT,
    csat_score INT,
    ces_score INT,
    submitted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Testing Checklist
- [ ] Survey sending
- [ ] Response collection
- [ ] Rating calculation
- [ ] Report generation
- [ ] Follow-up triggers

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
