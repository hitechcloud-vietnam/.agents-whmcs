# WHMCS Client CSAT Survey Workflow

## Description
Send customer satisfaction surveys after interactions.

## Steps

### Step 1: Create Survey
```php
<?php
function sendCSATSurvey($ticketId)
{
    $ticket = Capsule::table('tbltickets')->find($ticketId);
    
    // Generate survey token
    $token = bin2hex(random_bytes(16));
    
    Capsule::table('mod_csat_surveys')->insert([
        'ticket_id' => $ticketId,
        'client_id' => $ticket->userid,
        'token' => $token,
        'sent_at' => date('Y-m-d H:i:s'),
        'expires_at' => date('Y-m-d H:i:s', strtotime('+7 days')),
    ]);
    
    $surveyUrl = $systemUrl . '/survey.php?token=' . $token;
    
    sendEmail('CSATSurvey', $ticket->userid, [
        'ticket_id' => $ticketId,
        'survey_url' => $surveyUrl,
    ]);
}
```

### Step 2: Survey Submission
```php
<?php
function submitSurvey($token, $rating, $feedback)
{
    $survey = Capsule::table('mod_csat_surveys')
        ->where('token', $token)
        ->where('status', 'pending')
        ->first();
    
    if (!$survey) {
        return ['error' => 'Invalid survey token'];
    }
    
    Capsule::table('mod_csat_surveys')
        ->where('id', $survey->id)
        ->update([
            'rating' => $rating,
            'feedback' => $feedback,
            'submitted_at' => date('Y-m-d H:i:s'),
            'status' => 'completed',
        ]);
    
    // Alert on low ratings
    if ($rating <= 2) {
        sendEmail('LowCSATAlert', $survey->ticket_id);
    }
    
    return ['success' => true];
}
```

## Tags
- survey
- csat
- feedback
- satisfaction