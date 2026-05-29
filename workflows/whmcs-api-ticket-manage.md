# WHMCS API Ticket Manage Workflow

## Purpose
Guide developers through managing support tickets via WHMCS API.

## Prerequisites
- WHMCS installation
- Support API access
- Ticket departments configured
- PHP skills

## Steps

### Phase 1: Ticket Operations

1. Create ticket
   ```php
   function createTicket($clientId, $subject, $message, $deptId = 1): array {
       return localAPI('OpenTicket', [
           'clientid' => $clientId,
           'deptid' => $deptId,
           'subject' => $subject,
           'message' => $message,
           'priority' => 'Medium',
       ]);
   }
   ```

2. Reply to ticket
   ```php
   localAPI('AddTicketReply', [
       'ticketid' => 123,
       'message' => 'This is a reply to the ticket',
       'clientid' => 456,
   ]);
   ```

### Phase 2: Ticket Management

1. Get ticket list
   ```php
   $tickets = localAPI('GetTickets', [
       'clientid' => 123,
       'status' => 'Open',
   ]);
   ```

2. Get ticket details
   ```php
   $ticket = localAPI('GetTicket', [
       'ticketid' => 123,
   ]);
   ```

3. Update ticket status
   ```php
   localAPI('UpdateTicket', [
       'ticketid' => 123,
       'status' => 'Answered',
       'priority' => 'High',
   ]);
   ```

## Related Workflows
- whmcs-api-client-creation
- whmcs-api-integration
- whmcs-api-reporting
