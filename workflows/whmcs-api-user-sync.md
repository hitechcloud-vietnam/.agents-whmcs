# WHMCS API User Sync Workflow

## Purpose
Guide developers through synchronizing user data between WHMCS and external systems.

## Prerequisites
- WHMCS installation
- External user database
- API access credentials
- PHP development skills

## Steps

### Phase 1: User Sync Architecture

1. Sync strategy
   ```
   User Sync Methods:
   ├── Real-time sync (Webhooks)
   ├── Scheduled sync (Cron)
   ├── On-demand sync (API)
   └── Hybrid approach
   ```

2. Sync components
   ```
   User Data Fields:
   - Basic info (name, email, phone)
   - Address information
   - Custom fields
   - Client groups
   - Status/permissions
   ```

### Phase 2: User Sync Implementation

1. Sync service class
   ```php
   <?php
   class UserSyncService {
       private $whmcsApi;
       private $externalApi;
       
       public function __construct($whmcsConfig, $externalConfig) {
           $this->whmcsApi = new WHMCSApiClient($whmcsConfig);
           $this->externalApi = new ExternalApiClient($externalConfig);
       }
       
       public function syncUser($userId): array {
           // Fetch from external system
           $externalUser = $this->externalApi->getUser($userId);
           
           // Sync to WHMCS
           $result = $this->whmcsApi->call('UpdateClient', [
               'clientid' => $externalUser['whmcs_id'],
               'firstname' => $externalUser['first_name'],
               'lastname' => $externalUser['last_name'],
               'email' => $externalUser['email'],
               'phonenumber' => $externalUser['phone'],
           ]);
           
           return $result;
       }
   }
   ```

2. Full sync command
   ```php
   public function fullSync(): array {
       $externalUsers = $this->externalApi->getAllUsers();
       $results = ['created' => 0, 'updated' => 0, 'errors' => []];
       
       foreach ($externalUsers as $user) {
           try {
               $result = $this->syncUser($user['id']);
               if ($result['result'] === 'success') {
                   $results['updated']++;
               }
           } catch (Exception $e) {
               $results['errors'][] = [
                   'user_id' => $user['id'],
                   'error' => $e->getMessage(),
               ];
           }
       }
       
       return $results;
   }
   ```

### Phase 3: Sync Hooks

1. WHMCS hooks for sync
   ```php
   add_hook('ClientAdd', 1, function($vars) {
       // Sync new client to external system
       $externalService->createUser($vars);
   });
   
   add_hook('ClientEdit', 1, function($vars) {
       // Sync updates to external system
       $externalService->updateUser($vars);
   });
   
   add_hook('ClientDelete', 1, function($vars) {
       // Handle client deletion
       $externalService->deactivateUser($vars['user_id']);
   });
   ```

### Phase 4: Conflict Resolution

1. Conflict handling
   ```php
   function resolveConflict($whmcsData, $externalData, $strategy = 'whmcs_wins') {
       switch ($strategy) {
           case 'whmcs_wins':
               return $whmcsData;
           case 'external_wins':
               return $externalData;
           case 'newest_wins':
               return $whmcsData['updated_at'] > $externalData['updated_at'] 
                   ? $whmcsData : $externalData;
           case 'merge':
               return array_merge($externalData, $whmcsData);
       }
   }
   ```

## Related Workflows
- whmcs-api-integration
- whmcs-api-client-creation
- whmcs-api-sync
