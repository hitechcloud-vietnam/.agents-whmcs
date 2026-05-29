# WHMCS AML Screening Skill

## Purpose
Provides patterns for implementing AML (Anti-Money Laundering) screening in WHMCS, screening customers against sanctions lists, PEP databases, and adverse media sources.

## Implementation Patterns

### AML Screening Engine
```php
<?php
class AMLScreeningEngine {
    private $db;
    
    public function screen($party) {
        $results = [
            'party_name' => $party['name'],
            'timestamp' => date('Y-m-d H:i:s'),
            'sanctions_matches' => [],
            'pep_matches' => [],
            'risk_score' => 0,
            'recommendation' => 'approve'
        ];
        
        // Sanctions check
        $sanctionsMatches = $this->checkSanctions($party['name']);
        $results['sanctions_matches'] = $sanctionsMatches;
        if (!empty($sanctionsMatches)) {
            $results['risk_score'] += 50;
        }
        
        // PEP check
        $pepMatches = $this->checkPEP($party['name']);
        $results['pep_matches'] = $pepMatches;
        if (!empty($pepMatches)) {
            $results['risk_score'] += 30;
        }
        
        // Calculate recommendation
        if ($results['risk_score'] >= 70) {
            $results['recommendation'] = 'block';
        } elseif ($results['risk_score'] >= 40) {
            $results['recommendation'] = 'review';
        }
        
        $this->storeResults($results);
        return $results;
    }
    
    private function checkSanctions($name) {
        return $this->db->select(
            "SELECT * FROM mod_sanctions_entries 
             WHERE UPPER(name) LIKE ? LIMIT 5",
            ['%' . strtoupper($name) . '%']
        );
    }
    
    private function checkPEP($name) {
        return $this->db->select(
            "SELECT * FROM mod_pep_database 
             WHERE UPPER(name) LIKE ? LIMIT 10",
            ['%' . strtoupper($name) . '%']
        );
    }
    
    private function storeResults($results) {
        $this->db->insert('mod_aml_screenings', [
            'party_name' => $results['party_name'],
            'risk_score' => $results['risk_score'],
            'recommendation' => $results['recommendation'],
            'matches' => json_encode([
                'sanctions' => $results['sanctions_matches'],
                'pep' => $results['pep_matches']
            ]),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    public function screenTransaction($transaction) {
        $score = 0;
        
        // Screen all parties
        foreach ($transaction['parties'] ?? [] as $party) {
            $result = $this->screen($party);
            $score += $result['risk_score'] * 0.2;
        }
        
        // Check for structuring
        if ($transaction['amount'] >= 8000 && $transaction['amount'] < 10000) {
            $score += 20;
        }
        
        return [
            'risk_score' => min(100, $score),
            'approved' => $score < 80
        ];
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_aml_screenings (
    id INT AUTO_INCREMENT PRIMARY KEY,
    party_name VARCHAR(255),
    risk_score DECIMAL(5,2),
    recommendation VARCHAR(50),
    matches JSON,
    created_at DATETIME
);

CREATE TABLE mod_pep_database (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    position VARCHAR(255),
    organization VARCHAR(255),
    pep_category VARCHAR(50),
    country VARCHAR(10)
);
```

## Usage Examples

### Screen Customer
```php
$aml = new AMLScreeningEngine();
$result = $aml->screen(['name' => 'John Smith']);

if ($result['recommendation'] === 'block') {
    echo "Customer blocked due to sanctions match";
}
```

### Screen Transaction
```php
$result = $aml->screenTransaction([
    'amount' => 9500,
    'parties' => [['name' => $clientName]]
]);
```
