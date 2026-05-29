# WHMCS Contract Builder

## Concept
Contract automation and e-signature.

## Code
```php
<?php
class ContractBuilder {
    public static function createContract($clientId, $templateId, $variables) {
        $template = getContractTemplate($templateId);
        $content = self::fillTemplate($template["content"], $variables);
        
        return insert_query("tbl_contracts", [
            "client_id" => $clientId, "content" => $content,
            "template_id" => $templateId, "status" => "Draft"
        ]);
    }
    
    public static function sendForSignature($contractId) {
        $contract = getContract($contractId);
        sendEsignatureRequest($contract["client_id"], $contract["content"]);
        update_query("tbl_contracts", ["status" => "Pending Signature"], ["id" => $contractId]);
    }
}
```
