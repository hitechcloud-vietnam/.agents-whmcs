# WHMCS Sanctions Check Skill

## Purpose
Provides patterns for implementing sanctions list checking in WHMCS, screening individuals and entities against OFAC, EU, UN, and other sanctions lists.

## Implementation Patterns

### Sanctions Checker
```php
<?php
class SanctionsChecker {
    private $db;
    
    public function checkName($name) {
        $normalizedName = strtoupper(trim($name));
        
        // Exact match
        $exactMatches = $this->db->select(
            "SELECT * FROM mod_sanctions_entries WHERE UPPER(name) = ?",
            [$normalizedName]
        );
        
        // Fuzzy match
        $fuzzyMatches = $this->db->select(
            "SELECT * FROM mod_sanctions_entries WHERE name LIKE ? LIMIT 10",
            ['%' . $normalizedName . '%']
        );
        
        $blocked = !empty($exactMatches);
        $requiresReview = !empty($fuzzyMatches);
        
        return [
            'query_name' => $name,
            'exact_matches' => $exactMatches,
            'fuzzy_matches' => $fuzzyMatches,
            'blocked' => $blocked,
            'requires_review' => $requiresReview
        ];
    }
    
    public function checkWithIdentifiers($data) {
        $results = ['blocked' => false];
        
        if (!empty($data['name'])) {
            $result = $this->checkName($data['name']);
            if ($result['blocked']) $results['blocked'] = true;
            $results['name_check'] = $result;
        }
        
        if (!empty($data['date_of_birth'])) {
            $dobMatch = $this->db->select(
                "SELECT * FROM mod_sanctions_entries WHERE date_of_birth = ?",
                [$data['date_of_birth']]
            );
            if (!empty($dobMatch)) $results['blocked'] = true;
            $results['dob_check'] = ['matches' => $dobMatch];
        }
        
        return $results;
    }
    
    public function batchCheck($names) {
        $results = [];
        foreach ($names as $name) {
            $results[$name] = $this->checkName($name);
        }
        return $results;
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_sanctions_entries (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    aliases JSON,
    date_of_birth DATE,
    nationality VARCHAR(10),
    addresses JSON,
    is_active TINYINT(1) DEFAULT 1
);

CREATE TABLE mod_sanctions_lists (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    entry_count INT,
    last_updated DATETIME
);
```

## Usage Examples

### Check a Name
```php
$checker = new SanctionsChecker();
$result = $checker->checkName('John Smith');

if ($result['blocked']) {
    echo "BLOCKED - Sanctions match found";
} elseif ($result['requires_review']) {
    echo "Review required - possible match";
} else {
    echo "Clear";
}
```
