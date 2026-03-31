# Rendering & Screenshots

Tips for generating visual output (screenshots, diagrams) from Claude Code tasks.

## Screenshots with Playwright

```bash
# Install once: npm install playwright && npx playwright install chromium
node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch({headless: true, args: ['--no-sandbox']});
  const page = await browser.newPage();
  await page.goto('file:///path/to/file.html', {waitUntil: 'networkidle'});
  await page.screenshot({path: '/tmp/output.png', fullPage: true});
  await browser.close();
})();
"
```

### SVG Content Cropping

Mermaid renders SVGs at their natural (small) size — often ~300px wide. To get a clean screenshot:

```bash
# 1. Get SVG bounding box
const bbox = await page.evaluate(() => {
  const svg = document.querySelector('.mermaid svg');
  return svg.getBoundingClientRect();
});

# 2. Crop to content area with padding
await page.screenshot({
  path: '/tmp/output.png',
  clip: { x: bbox.x - 30, y: bbox.y - 30, width: bbox.width + 60, height: bbox.height + 60 }
});
```

### High-DPI Output

```bash
const ctx = await browser.newContext({
  viewport: { width: 800, height: 800 },
  deviceScaleFactor: 3  // 3x for crisp output
});
```

### Telegram Compression

- Telegram compresses images sent as photos → blurry
- Send as document: `message action=send asDocument=true`

## Mermaid Pitfalls

### sequenceDiagram Color Support

| Feature | Works? | Alternative |
|---------|--------|-------------|
| `style` annotations | ❌ No | — |
| `rect rgb(...)` | ❌ No (NaN coordinates) | Use `rect #hexcolor` |
| `rect rgba(...)` | ❌ No | Use `rect #hexcolor` |
| `rect #dcfce7` (hex) | ✅ Yes | For colored backgrounds |
| `Note right of` | ✅ Yes | For annotations |

**Best practice for diff visualization**: Use `Note right of Participant: 🟢ADDED` instead of `rect`.

### CDN Timeout

Mermaid CDN can be slow/unreliable. For repeated rendering, install locally:

```bash
npm install mermaid
# Reference: /path/to/node_modules/mermaid/dist/mermaid.min.js
```

### SVG Auto-Sizing

Mermaid renders SVGs at natural size. Use `viewport` + `clip` or `fullPage: true` for complete screenshots.
