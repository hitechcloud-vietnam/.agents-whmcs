# WHMCS AI Ticket Prioritization Devkit

## Overview
An AI-powered ticket prioritization system for WHMCS that analyzes ticket content, customer history, and urgency indicators to automatically assign priority levels.

## Features
- AI-based priority scoring
- Content analysis
- Sentiment detection
- Urgency indicators
- Customer tier weighting
- SLA deadline calculation
- Auto-priority assignment
- Priority recommendations
- Training feedback loop
- Batch processing
- Priority analytics

## WHMCS Integration Points
- Module: addon/TicketPrioritizer
- Hook: TicketOpen
- Hook: TicketReply
- Hook: TicketSave

## Database Schema
```sql
CREATE TABLE mod_ticket_priorities (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    score_range_min INT,
    score_range_max INT,
    color_code VARCHAR(20),
    default_escalation_hours INT,
    sort_order INT DEFAULT 0
);

CREATE TABLE mod_priority_analysis (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ticket_id INT NOT NULL,
    priority_score DECIMAL(5,2) NOT NULL,
    assigned_priority VARCHAR(20),
    confidence_score DECIMAL(5,2),
    factors JSON,
    sentiment_score DECIMAL(5,2),
    urgency_keywords JSON,
    customer_tier INT,
    analyzed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_priority_training (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ticket_id INT NOT NULL,
    ai_priority VARCHAR(20),
    final_priority VARCHAR(20),
    adjustment_reason TEXT,
    is_correct TINYINT(1),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Testing Checklist
- [ ] Priority scoring accuracy
- [ ] Sentiment analysis
- [ ] SLA calculation
- [ ] Training feedback
- [ ] Analytics dashboard

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
- ML/NLP library
