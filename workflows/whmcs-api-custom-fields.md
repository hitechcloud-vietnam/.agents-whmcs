# WHMCS API Custom Fields Workflow

## Purpose
Guide developers through managing custom fields via WHMCS API.

## Prerequisites
- WHMCS installation
- Custom field API access
- PHP skills

## Steps

### Phase 1: Custom Field Operations

1. Get custom fields
   ```php
   // Get client custom fields
   $client = localAPI('GetClientsDetails', [
       'clientid' => 123,
       'stats' => false,
   ]);
   
   $customFields = $client['customfields']['customfield'];
   ```

2. Update custom fields
   ```php
   localAPI('UpdateClient', [
       'clientid' => 123,
       'customfields' => [
           'field_1' => 'value',
           'field_2' => 'another value',
       ],
   ]);
   ```

3. Create custom fields via API
   ```php
   Capsule::table('tblcustomfields')->insert([
       'type' => 'client',
       'relid' => 0,
       'fieldname' => 'Custom Field',
       'fieldtype' => 'text',
       'description' => 'Description',
       'required' => 'on',
   ]);
   ```

## Related Workflows
- whmcs-api-client-creation
- whmcs-api-client-update
- whmcs-api-integration
