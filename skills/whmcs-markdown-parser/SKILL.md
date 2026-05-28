# WHMCS Markdown Parser Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing Markdown rendering in WHMCS.

## When to Use

- Content modules
- Knowledge base systems
- Email template with markdown

## Markdown Patterns

```php
<?php
class MarkdownParser {
    private Parsedown $parser;

    public function __construct() {
        $this->parser = new Parsedown();
        $this->parser->setBreaksEnabled(true);
        $this->parser->setUrlsLinked(true);
    }

    public function parse(string $text): string {
        // Sanitize input
        $text = htmlspecialchars($text, ENT_QUOTES, 'UTF-8');

        // Convert markdown
        return $this->parser->text($text);
    }

    public function parseToPdf(string $text, string $outputPath): bool {
        $html = $this->parse($text);

        // Generate PDF using dompdf or similar
        $dompdf = new \Dompdf\Dompdf();
        $dompdf->loadHtml($html);
        $dompdf->render();
        file_put_contents($outputPath, $dompdf->output());

        return file_exists($outputPath);
    }
}
```

### Smarty Plugin
```php
// function.markdown.php
function smarty_modifier_markdown(string $text): string {
    $parser = new MarkdownParser();
    return $parser->parse($text);
}

// Usage in template: {$text|markdown}
```

---

**Related Skills:**
- whmcs-template-styling
- whmcs-addon-builder
- whmcs-clientarea-builder
