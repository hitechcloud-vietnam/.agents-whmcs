# WHMCS Live Chat Integration Devkit

## Overview
A live chat integration system for WHMCS that connects with popular chat services, provides real-time support, chat routing, and visitor monitoring.

## Features
- Multi-provider chat integration
- Real-time visitor monitoring
- Chat routing by department
- Pre-chat form customization
- Chat file transfer
- Screen sharing integration
- Chat transcript storage
- Visitor information display
- Proactive chat triggers
- Offline message handling
- Chat rating system
- Integration analytics

## WHMCS Integration Points
- Module: addon/ChatSupport
- Hook: ClientAreaPage
- Hook: TicketCreate
- JavaScript widget integration

## Database Schema
```sql
CREATE TABLE mod_chat_sessions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    visitor_id VARCHAR(100),
    client_id INT,
    department_id INT,
    agent_id INT,
    status ENUM('waiting', 'active', 'ended', 'abandoned') DEFAULT 'waiting',
    started_at TIMESTAMP,
    ended_at TIMESTAMP,
    rating INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_chat_messages (
    id INT AUTO_INCREMENT PRIMARY KEY,
    session_id INT NOT NULL,
    sender_type ENUM('visitor', 'agent', 'system') NOT NULL,
    sender_id INT,
    message TEXT NOT NULL,
    is_read TINYINT(1) DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_session (session_id)
);

CREATE TABLE mod_chat_departments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    chat_provider_id VARCHAR(100),
    is_active TINYINT(1) DEFAULT 1
);

CREATE TABLE mod_chat_agents (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    department_id INT,
    status ENUM('online', 'away', 'offline') DEFAULT 'offline',
    max_concurrent INT DEFAULT 5,
    current_load INT DEFAULT 0
);
```

## API Endpoints
- POST /api/chat/sessions - Start session
- GET /api/chat/sessions/:id - Get session
- POST /api/chat/messages - Send message
- GET /api/chat/transcripts/:id - Get transcript

## Testing Checklist
- [ ] Chat widget display
- [ ] Message delivery
- [ ] Agent routing
- [ ] Transcript storage
- [ ] Rating system

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
