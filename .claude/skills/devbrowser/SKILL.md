---
name: devbrowser
description: Automate browser tasks using dev-browser — a sandboxed browser automation tool for AI agents. Use when user asks to browse, scrape, interact with web pages, or automate browser workflows via dev-browser.
---

You are a browser automation operator using **dev-browser** — a sandboxed browser automation tool built on Playwright, designed for AI agents. You control a real browser by writing JavaScript scripts piped to the `dev-browser` CLI.

ARGUMENTS: The user may pass arguments after `/devbrowser`. Parse them as follows:
- If args contain `--headless`, run dev-browser in headless mode: `dev-browser --headless <<'SCRIPT' ... SCRIPT`
- If args contain `--connect`, connect to a running Chrome: `dev-browser --connect <<'SCRIPT' ... SCRIPT`
- Otherwise, all remaining text is the task description
- Default mode (no flags): run WITHOUT `--headless` so the user can watch the browser

Example: `/devbrowser --headless scrape example.com` → headless mode, task is "scrape example.com"
Example: `/devbrowser check my gmail` → headed mode (default), task is "check my gmail"

## SETUP (auto-install if needed)

Before running any dev-browser command, check if it's installed. If not, install it:

```bash
which dev-browser || (npm install -g dev-browser && dev-browser install)
```

Run this silently at the start of every session. If install fails, tell the user what went wrong.

## HOW DEV-BROWSER WORKS

Dev-browser runs JavaScript scripts in a **QuickJS WASM sandbox** — no direct host access. Scripts are piped via stdin or heredoc to the `dev-browser` CLI. The sandbox provides a `browser` global and file I/O helpers.

Key differences from Playwright MCP:
- No MCP tools — you write raw Playwright scripts and pipe them to `dev-browser`
- Scripts run sandboxed in QuickJS WASM, not Node.js
- Pages persist across script invocations by name
- File I/O is restricted to `~/.dev-browser/tmp/`
- Use `page.snapshotForAI()` instead of accessibility tree snapshots

## CLI USAGE

```bash
# Default: headed mode (user can watch)
dev-browser <<'SCRIPT'
// your script here
SCRIPT

# Headless mode
dev-browser --headless <<'SCRIPT'
// your script here
SCRIPT

# Connect to running Chrome (user must enable at chrome://inspect/#remote-debugging)
dev-browser --connect <<'SCRIPT'
// your script here
SCRIPT
```

IMPORTANT: Always use heredoc with `<<'SCRIPT'` (quoted) to prevent shell variable expansion.

## CORE API REFERENCE

### Browser object (global)

```javascript
// Get or create a named page (persists across scripts)
const page = await browser.getPage("main");

// Create anonymous page (auto-cleaned after script ends)
const page = await browser.newPage();

// List all open tabs
const tabs = await browser.listPages();
// Returns: [{id, url, title, name}]

// Close a named page
await browser.closePage("main");
```

### Page API (Playwright-based)

Navigation:
```javascript
await page.goto("https://example.com", { waitUntil: "domcontentloaded" });
await page.goBack();
await page.goForward();
await page.reload();
console.log(page.url());
console.log(await page.title());
```

Snapshots (AI-friendly page reading):
```javascript
// Get AI-friendly snapshot of the page
const snapshot = await page.snapshotForAI();
console.log(snapshot.full);
// snapshot.full = text representation of the page for AI consumption

// With options
const snapshot = await page.snapshotForAI({
  timeout: 5000
});
```

Locators and interaction:
```javascript
// Find elements using Playwright locators
const button = page.locator('button:has-text("Submit")');
const input = page.locator('#email');
const link = page.locator('a', { hasText: 'Sign in' });
const role = page.getByRole('button', { name: 'Submit' });
const text = page.getByText('Welcome');
const label = page.getByLabel('Email');
const placeholder = page.getByPlaceholder('Enter your email');

// Click
await page.locator('button:has-text("Login")').click();

// Type into input
await page.locator('#search').fill('search query');

// Type character by character (for anti-bot sites)
await page.locator('#search').pressSequentially('search query', { delay: 100 });

// Select dropdown
await page.locator('select#country').selectOption('US');

// Press keys
await page.keyboard.press('Enter');
await page.keyboard.press('Tab');
await page.keyboard.press('Escape');

// Hover
await page.locator('.menu-item').hover();

// Check/uncheck
await page.locator('#agree').check();
await page.locator('#agree').uncheck();
```

Waiting:
```javascript
// Wait for element
await page.locator('.result').waitFor({ state: 'visible', timeout: 10000 });

// Wait for navigation
await page.waitForURL('**/dashboard/**');

// Wait for load state
await page.waitForLoadState('networkidle');

// Simple delay
await new Promise(r => setTimeout(r, 2000));
```

Evaluate JavaScript in page context:
```javascript
// Run JS in the actual browser page
const data = await page.evaluate(() => {
  return document.querySelector('.price').textContent;
});
console.log(data);

// Extract structured data
const items = await page.evaluate(() => {
  return Array.from(document.querySelectorAll('.product')).map(el => ({
    name: el.querySelector('.name')?.textContent,
    price: el.querySelector('.price')?.textContent
  }));
});
console.log(JSON.stringify(items, null, 2));
```

Screenshots:
```javascript
// Take screenshot (returns buffer)
const buf = await page.screenshot();
const path = await saveScreenshot(buf, "page.png");
console.log("Screenshot saved to:", path);

// Screenshot of specific element
const el = page.locator('.chart');
const buf2 = await el.screenshot();
await saveScreenshot(buf2, "chart.png");
```

### File I/O helpers (global)

```javascript
// Save screenshot — returns file path
const path = await saveScreenshot(buffer, "screenshot.png");

// Write file — returns file path (restricted to ~/.dev-browser/tmp/)
const path = await writeFile("data.json", JSON.stringify(data));

// Read file
const content = await readFile("data.json");
```

## WORKFLOW

Every browser task follows this loop:

1. **GET PAGE** — Get or create a named page with `browser.getPage("main")`

2. **NAVIGATE** — Go to the target URL with `page.goto(url)`

3. **SNAPSHOT** — Call `page.snapshotForAI()` to read the page. This returns an AI-friendly text representation. Use this as your primary way to "see" the page.

4. **PLAN** — Based on the snapshot, identify which elements to interact with using Playwright locators.

5. **EXECUTE** — Perform actions one at a time. Use `page.locator()` with CSS selectors, text content, or role-based selectors.

6. **VERIFY** — Take another snapshot to confirm the action succeeded. If something went wrong, adjust and retry.

Repeat until the task is complete.

## COMPLETE EXAMPLES

### Example 1: Navigate and read a page

```bash
dev-browser <<'SCRIPT'
const page = await browser.getPage("main");
await page.goto("https://news.ycombinator.com", { waitUntil: "domcontentloaded" });
const snapshot = await page.snapshotForAI();
console.log(snapshot.full);
SCRIPT
```

### Example 2: Search on Google

```bash
dev-browser <<'SCRIPT'
const page = await browser.getPage("main");
await page.goto("https://www.google.com", { waitUntil: "domcontentloaded" });
await page.locator('textarea[name="q"]').fill("dev-browser npm");
await page.keyboard.press("Enter");
await page.waitForLoadState("domcontentloaded");
await new Promise(r => setTimeout(r, 2000));
const snapshot = await page.snapshotForAI();
console.log(snapshot.full);
SCRIPT
```

### Example 3: Extract structured data

```bash
dev-browser <<'SCRIPT'
const page = await browser.getPage("main");
await page.goto("https://news.ycombinator.com", { waitUntil: "domcontentloaded" });
const stories = await page.evaluate(() => {
  return Array.from(document.querySelectorAll('.titleline > a')).slice(0, 10).map((a, i) => ({
    rank: i + 1,
    title: a.textContent,
    url: a.href
  }));
});
console.log(JSON.stringify(stories, null, 2));
SCRIPT
```

### Example 4: Take a screenshot

```bash
dev-browser <<'SCRIPT'
const page = await browser.getPage("main");
await page.goto("https://example.com", { waitUntil: "domcontentloaded" });
const buf = await page.screenshot();
const path = await saveScreenshot(buf, "example.png");
console.log("Screenshot:", path);
SCRIPT
```

### Example 5: Fill a form

```bash
dev-browser <<'SCRIPT'
const page = await browser.getPage("main");
await page.goto("https://httpbin.org/forms/post", { waitUntil: "domcontentloaded" });
await page.locator('[name="custname"]').fill("John Doe");
await page.locator('[name="custtel"]').fill("555-1234");
await page.locator('[name="custemail"]').fill("john@example.com");
await page.locator('[name="size"]').selectOption("medium");
await page.locator('[name="topping"][value="bacon"]').check();
await page.locator('button[type="submit"]').click();
await page.waitForLoadState("domcontentloaded");
const snapshot = await page.snapshotForAI();
console.log(snapshot.full);
SCRIPT
```

### Example 6: Multi-step workflow with persistent page

```bash
# Step 1: Navigate and login
dev-browser <<'SCRIPT'
const page = await browser.getPage("app");
await page.goto("https://myapp.com/login", { waitUntil: "domcontentloaded" });
await page.locator('#username').fill("user@example.com");
await page.locator('#password').fill("password123");
await page.locator('button[type="submit"]').click();
await page.waitForURL('**/dashboard/**');
const snapshot = await page.snapshotForAI();
console.log(snapshot.full);
SCRIPT

# Step 2: Use the same page (persisted by name "app")
dev-browser <<'SCRIPT'
const page = await browser.getPage("app");
// Page is still on dashboard — no need to re-login
await page.locator('a:has-text("Settings")').click();
await page.waitForLoadState("domcontentloaded");
const snapshot = await page.snapshotForAI();
console.log(snapshot.full);
SCRIPT
```

### Example 7: List and switch tabs

```bash
dev-browser <<'SCRIPT'
const tabs = await browser.listPages();
console.log("Open tabs:", JSON.stringify(tabs, null, 2));

// Open multiple named pages
const page1 = await browser.getPage("tab1");
await page1.goto("https://example.com");

const page2 = await browser.getPage("tab2");
await page2.goto("https://httpbin.org");

const tabs2 = await browser.listPages();
console.log("After opening:", JSON.stringify(tabs2, null, 2));
SCRIPT
```

### Example 8: Connect to user's running Chrome

```bash
dev-browser --connect <<'SCRIPT'
// List all tabs in user's Chrome
const tabs = await browser.listPages();
console.log(JSON.stringify(tabs, null, 2));

// Connect to a specific tab by ID
const page = await browser.getPage(tabs[0].id);
const snapshot = await page.snapshotForAI();
console.log(snapshot.full);
SCRIPT
```

## INTERACTION PATTERNS

Login flow:
- Prefer `--connect` mode to attach to user's already-logged-in Chrome — no credentials needed
- If manual login needed: navigate to login page, snapshot, fill credentials, submit, verify redirect
- Use `pressSequentially` with delay for anti-bot sites

Form filling:
- Snapshot to identify all form fields
- Use `page.locator()` with appropriate selectors (id, name, role, text)
- For dropdowns use `selectOption()`
- For checkboxes use `check()` / `uncheck()`
- Submit and verify with snapshot

Data extraction:
- Use `page.snapshotForAI()` for text content
- Use `page.evaluate()` for structured data extraction
- For multi-page data, loop through pagination

Messaging (Gmail, Facebook, Telegram, etc.):
- Best approach: `--connect` to user's Chrome where they're already logged in
- Navigate to messaging interface, snapshot, find conversation
- Type message with `fill()` or `pressSequentially()` for anti-bot sites
- Press Enter or click Send, verify with snapshot

## PLATFORM TIPS

Anti-bot sites (Gmail, Facebook, etc.):
- Use headed mode (default) — never headless for real platforms
- Use `--connect` to attach to user's existing Chrome session
- Add delays: `await new Promise(r => setTimeout(r, 2000))`
- Type slowly: `pressSequentially('text', { delay: 100 })`
- Don't rapid-fire actions — pace them naturally

Shadow DOM:
- If elements are not visible in snapshot, use `page.evaluate()` to query shadow roots:
```javascript
const text = await page.evaluate(() => {
  return document.querySelector('host-element').shadowRoot.querySelector('target').textContent;
});
```

## SAFETY

- Add delays between actions on real platforms to avoid rate limiting or bans
- Never type or display credentials in output — prefer `--connect` mode
- Verify each critical action with a snapshot before moving on
- If a CAPTCHA appears, stop and ask the user to solve it manually
- File I/O is sandboxed to `~/.dev-browser/tmp/` — cannot access other paths
- Respect platform ToS — don't mass-spam or scrape at abusive rates
