# WHMCS Currency Converter Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build multi-currency conversion modules for international billing.

## Currency Converter

```php
<?php
class CurrencyConverter {
    private array $rates = [];
    private string $baseCurrency = 'USD';

    public function __construct() {
        $this->loadRates();
    }

    public function convert(float $amount, string $from, string $to): float {
        if ($from === $to) {
            return $amount;
        }

        $fromRate = $this->getRate($from);
        $toRate = $this->getRate($to);

        $usdAmount = $amount / $fromRate;
        return $usdAmount * $toRate;
    }

    public function convertWithFees(float $amount, string $from, string $to, float $margin = 0): float {
        $converted = $this->convert($amount, $from, $to);
        return $converted * (1 + $margin / 100);
    }

    public function getRate(string $currency): float {
        return $this->rates[$currency] ?? 1.0;
    }

    private function loadRates(): void {
        $cached = Capsule::table('mod_currency_rates')
            ->where('base', $this->baseCurrency)
            ->where('updated_at', '>', date('Y-m-d H:i:s', strtotime('-1 hour')))
            ->first();

        if ($cached) {
            $this->rates = json_decode($cached->rates, true);
            return;
        }

        $this->fetchRates();
    }

    private function fetchRates(): void {
        $apiUrl = 'https://api.exchangerate-api.com/v4/latest/' . $this->baseCurrency;

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $apiUrl,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 10,
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        $data = json_decode($response, true);

        if ($data && isset($data['rates'])) {
            $this->rates = $data['rates'];
            $this->saveRates();
        }
    }

    private function saveRates(): void {
        Capsule::table('mod_currency_rates')->updateOrInsert(
            ['base' => $this->baseCurrency],
            [
                'rates' => json_encode($this->rates),
                'updated_at' => date('Y-m-d H:i:s'),
            ]
        );
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-internationalization