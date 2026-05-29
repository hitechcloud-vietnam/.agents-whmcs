# WHMCS API Error Handling Workflow

## Purpose
Guide developers through implementing robust error handling for WHMCS API calls.

## Prerequisites
- WHMCS installation
- API integration
- PHP exception handling
- Logging knowledge

## Steps

### Phase 1: Error Types

1. Common API errors
   ```
   API Error Categories:
   - Authentication errors (401)
   - Permission errors (403)
   - Not found errors (404)
   - Validation errors (422)
   - Server errors (500)
   - Rate limit errors (429)
   ```

2. Error handling class
   ```php
   class ApiErrorHandler {
       public function handle($response, $context = []): void {
           if (isset($response['result']) && $response['result'] === 'error') {
               throw new WHMCSApiException(
                   $response['message'],
                   $this->getErrorCode($response['message']),
                   $context
               );
           }
           
           if (isset($response['status']) && $response['status'] >= 400) {
               throw new ApiHttpException(
                   $response['status'],
                   $response['message'] ?? 'Unknown error',
                   $context
               );
           }
       }
       
       private function getErrorCode($message): string {
           $codes = [
               'Access Denied' => 'AUTH_FAILED',
               'Invalid API Key' => 'INVALID_KEY',
               'IP Not Allowed' => 'IP_BLOCKED',
           ];
           
           return $codes[$message] ?? 'UNKNOWN';
       }
   }
   ```

### Phase 2: Retry Logic

1. Retry handler
   ```php
   class RetryHandler {
       private $maxRetries = 3;
       private $delays = [1, 5, 30];
       
       public function executeWithRetry($callback, $context = '') {
           $attempt = 0;
           
           while ($attempt < $this->maxRetries) {
               try {
                   return call_user_func($callback);
               } catch (RetryableException $e) {
                   $attempt++;
                   
                   if ($attempt >= $this->maxRetries) {
                       $this->logFailure($context, $e, $attempt);
                       throw $e;
                   }
                   
                   sleep($this->delays[$attempt - 1] ?? 30);
               }
           }
       }
   }
   ```

### Phase 3: Logging

1. API error logging
   ```php
   function logApiError($operation, $error, $context = []) {
       Capsule::table('mod_api_errors')->insert([
           'operation' => $operation,
           'error_message' => $error,
           'context' => json_encode($context),
           'created_at' => date('Y-m-d H:i:s'),
       ]);
       
       logActivity("API Error [$operation]: $error");
   }
   ```

## Related Workflows
- whmcs-api-authentication
- whmcs-api-rate-limiting
- whmcs-api-automation
