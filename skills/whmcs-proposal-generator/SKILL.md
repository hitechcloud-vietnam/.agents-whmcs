# WHMCS Proposal Generator

## Concept
Professional proposal creation.

## Code
```php
<?php
class ProposalGenerator {
    public static function createProposal($clientId, $title, $content, $products) {
        $proposalId = insert_query("tbl_proposals", [
            "client_id" => $clientId, "title" => $title,
            "content" => $content, "created_at" => now()
        ]);
        
        // Add products with pricing
        foreach ($products as $product) {
            self::addProposalLine($proposalId, $product);
        }
        
        return $proposalId;
    }
    
    public static function sendProposal($proposalId) {
        update_query("tbl_proposals", ["status" => "Sent", "sent_at" => now()], ["id" => $proposalId]);
        send_email("proposal", getProposal($proposalId)["client_id"]);
    }
}
```
