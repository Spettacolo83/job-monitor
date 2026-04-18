# Application Agent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an automatic job application system that uses Playwright browser sessions controlled by Claude to fill and submit job applications, with Telegram inline buttons for triggering and feedback.

**Architecture:** The scraper-proxy on VPS gets new interactive session endpoints that Claude controls step-by-step from a dedicated Routine. A Telegram webhook on the scraper-proxy handles inline button callbacks. The residential proxy on Mac is extended with the same session endpoints for sites requiring residential IP. Config files in the job-monitor repo store the application profile, knowledge base, and tracking data.

**Tech Stack:** Node.js, Express, Playwright, Telegram Bot API, Claude Code Routines (API trigger).

---

## File Map

### Scraper-proxy (VPS) — `/Users/stefanorussello/Documents/Projects/FollowTheFlow/scraper-proxy/`
| File | Action | Purpose |
|---|---|---|
| `src/sessions.js` | Create | Session manager: create/close sessions, execute commands |
| `src/session-routes.js` | Create | Express routes for /session/* endpoints |
| `src/telegram.js` | Create | Telegram webhook handler + inline button callbacks |
| `src/server.js` | Modify | Mount session routes + telegram webhook |
| `public/admin.html` | Modify | Add "Account Credentials" section |

### Residential proxy (Mac) — `/Users/stefanorussello/Documents/Projects/FollowTheFlow/residential-proxy/`
| File | Action | Purpose |
|---|---|---|
| `proxy.js` | Modify | Add session endpoints alongside existing /fetch |

### Job Monitor (repo) — `/Users/stefanorussello/Documents/Projects/FollowTheFlow/LinkedIn-Lavoro/`
| File | Action | Purpose |
|---|---|---|
| `config/application_profile.json` | Create | Personal data, work preferences, salary strategy |
| `config/knowledge_base.json` | Create | Learned answers + field mappings |
| `config/application_settings.json` | Create | Mode, threshold, limits |
| `data/application_queue.json` | Create | Pending applications queue |
| `data/applications.json` | Create | Application tracking history |
| `routine/application_prompt.md` | Create | Application Agent routine instructions |
| `routine/prompt.md` | Modify | Update notifications to use inline buttons |
| `assets/` | Create | Directory for CV PDF |

---

### Task 1: Session manager module (scraper-proxy)

**Files:**
- Create: `src/sessions.js`

- [ ] **Step 1: Create src/sessions.js**

```javascript
const { firefox } = require('playwright');
const config = require('./config');

const sessions = new Map();
const SESSION_TIMEOUT = 5 * 60 * 1000; // 5 minutes

async function getBrowser() {
  const cfg = config.get().browser || {};
  const browser = await firefox.launch({
    headless: true,
    args: []
  });
  return browser;
}

async function create(options = {}) {
  const id = 'sess_' + Date.now().toString(36) + Math.random().toString(36).substr(2, 5);
  const cfg = config.get().browser || {};

  const browser = await getBrowser();
  const context = await browser.newContext({
    userAgent: cfg.user_agent || 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)',
    viewport: { width: 1280, height: 720 },
    locale: 'en-US',
    timezoneId: 'Europe/Madrid'
  });
  const page = await context.newPage();

  const session = {
    id,
    browser,
    context,
    page,
    createdAt: Date.now(),
    lastActivity: Date.now(),
    useResidential: options.use_residential || false
  };

  sessions.set(id, session);
  scheduleTimeout(id);

  console.log(`Session created: ${id}`);
  return { session_id: id, created: true };
}

function scheduleTimeout(id) {
  const session = sessions.get(id);
  if (!session) return;
  if (session.timer) clearTimeout(session.timer);
  session.timer = setTimeout(() => close(id), SESSION_TIMEOUT);
}

function touch(id) {
  const session = sessions.get(id);
  if (session) {
    session.lastActivity = Date.now();
    scheduleTimeout(id);
  }
}

function getSession(id) {
  const session = sessions.get(id);
  if (!session) throw new Error(`Session ${id} not found`);
  touch(id);
  return session;
}

async function goto(id, url) {
  const { page } = getSession(id);
  const cfg = config.get().browser || {};

  const response = await page.goto(url, {
    waitUntil: 'domcontentloaded',
    timeout: cfg.timeout_ms || 45000
  });

  // Handle Cloudflare
  try {
    const title = await page.title();
    if (title.includes('Just a moment') || title.includes('Checking') || title.includes('Attention')) {
      console.log(`Session ${id}: Cloudflare detected, waiting...`);
      await page.waitForFunction(
        () => {
          const t = document.title.toLowerCase();
          return !t.includes('just a moment') && !t.includes('checking') && !t.includes('attention');
        },
        { timeout: 25000 }
      );
      await page.waitForTimeout(3000);
    } else {
      await page.waitForTimeout(cfg.wait_after_load_ms || 2000);
    }
  } catch {
    await page.waitForTimeout(2000);
  }

  return {
    success: true,
    title: await page.title(),
    url: page.url(),
    status_code: response ? response.status() : 0
  };
}

async function content(id, format = 'html') {
  const { page } = getSession(id);
  let result;
  if (format === 'text') {
    result = await page.innerText('body');
  } else {
    result = await page.content();
  }
  return { success: true, content: result, url: page.url() };
}

async function fill(id, { selector, label, value }) {
  const { page } = getSession(id);
  try {
    if (selector) {
      await page.fill(selector, value);
    } else if (label) {
      await page.getByLabel(label).fill(value);
    }
    return { success: true };
  } catch (e) {
    // Try alternative strategies
    try {
      if (label) {
        await page.locator(`[placeholder*="${label}" i]`).fill(value);
        return { success: true };
      }
    } catch {}
    return { success: false, error: e.message };
  }
}

async function click(id, { selector, text }) {
  const { page } = getSession(id);
  const urlBefore = page.url();
  try {
    if (selector) {
      await page.click(selector, { timeout: 10000 });
    } else if (text) {
      await page.getByRole('button', { name: text }).or(
        page.getByRole('link', { name: text })
      ).or(
        page.locator(`text="${text}"`)
      ).first().click({ timeout: 10000 });
    }
    await page.waitForTimeout(2000);
    return {
      success: true,
      navigation: page.url() !== urlBefore,
      new_url: page.url()
    };
  } catch (e) {
    return { success: false, error: e.message };
  }
}

async function selectOption(id, { selector, label, value }) {
  const { page } = getSession(id);
  try {
    if (selector) {
      await page.selectOption(selector, value);
    } else if (label) {
      await page.getByLabel(label).selectOption(value);
    }
    return { success: true };
  } catch (e) {
    return { success: false, error: e.message };
  }
}

async function upload(id, { selector, label, file_path }) {
  const { page } = getSession(id);
  try {
    let locator;
    if (selector) {
      locator = page.locator(selector);
    } else if (label) {
      locator = page.getByLabel(label);
    } else {
      locator = page.locator('input[type="file"]').first();
    }
    await locator.setInputFiles(file_path);
    return { success: true };
  } catch (e) {
    return { success: false, error: e.message };
  }
}

async function screenshot(id) {
  const { page } = getSession(id);
  const buffer = await page.screenshot({ fullPage: false });
  return { success: true, image_base64: buffer.toString('base64') };
}

async function close(id) {
  const session = sessions.get(id);
  if (!session) return { success: false, error: 'Session not found' };
  if (session.timer) clearTimeout(session.timer);
  try {
    await session.context.close();
    await session.browser.close();
  } catch {}
  sessions.delete(id);
  console.log(`Session closed: ${id}`);
  return { success: true };
}

function listSessions() {
  return Array.from(sessions.entries()).map(([id, s]) => ({
    id,
    created: new Date(s.createdAt).toISOString(),
    lastActivity: new Date(s.lastActivity).toISOString(),
    url: s.page.url(),
    useResidential: s.useResidential
  }));
}

module.exports = { create, goto, content, fill, click, selectOption, upload, screenshot, close, listSessions };
```

- [ ] **Step 2: Commit**

```bash
git add src/sessions.js
git commit -m "feat: add session manager for interactive browser control"
```

---

### Task 2: Session routes (scraper-proxy)

**Files:**
- Create: `src/session-routes.js`

- [ ] **Step 1: Create src/session-routes.js**

```javascript
const express = require('express');
const sessions = require('./sessions');
const { apiKeyAuth } = require('./auth');

const router = express.Router();

router.use(apiKeyAuth);

router.post('/create', async (req, res) => {
  try {
    const result = await sessions.create(req.body);
    res.json(result);
  } catch (e) {
    res.status(500).json({ success: false, error: e.message });
  }
});

router.get('/list', (req, res) => {
  res.json(sessions.listSessions());
});

router.post('/:id/goto', async (req, res) => {
  try {
    const result = await sessions.goto(req.params.id, req.body.url);
    res.json(result);
  } catch (e) {
    res.status(500).json({ success: false, error: e.message });
  }
});

router.get('/:id/content', async (req, res) => {
  try {
    const result = await sessions.content(req.params.id, req.query.format || 'html');
    res.json(result);
  } catch (e) {
    res.status(500).json({ success: false, error: e.message });
  }
});

router.post('/:id/fill', async (req, res) => {
  try {
    const result = await sessions.fill(req.params.id, req.body);
    res.json(result);
  } catch (e) {
    res.status(500).json({ success: false, error: e.message });
  }
});

router.post('/:id/click', async (req, res) => {
  try {
    const result = await sessions.click(req.params.id, req.body);
    res.json(result);
  } catch (e) {
    res.status(500).json({ success: false, error: e.message });
  }
});

router.post('/:id/select', async (req, res) => {
  try {
    const result = await sessions.selectOption(req.params.id, req.body);
    res.json(result);
  } catch (e) {
    res.status(500).json({ success: false, error: e.message });
  }
});

router.post('/:id/upload', async (req, res) => {
  try {
    const result = await sessions.upload(req.params.id, req.body);
    res.json(result);
  } catch (e) {
    res.status(500).json({ success: false, error: e.message });
  }
});

router.get('/:id/screenshot', async (req, res) => {
  try {
    const result = await sessions.screenshot(req.params.id);
    res.json(result);
  } catch (e) {
    res.status(500).json({ success: false, error: e.message });
  }
});

router.delete('/:id', async (req, res) => {
  try {
    const result = await sessions.close(req.params.id);
    res.json(result);
  } catch (e) {
    res.status(500).json({ success: false, error: e.message });
  }
});

module.exports = router;
```

- [ ] **Step 2: Commit**

```bash
git add src/session-routes.js
git commit -m "feat: add REST routes for interactive browser sessions"
```

---

### Task 3: Telegram webhook handler (scraper-proxy)

**Files:**
- Create: `src/telegram.js`

- [ ] **Step 1: Create src/telegram.js**

```javascript
const https = require('https');
const config = require('./config');

function sendTelegramRequest(method, body) {
  const cfg = config.get();
  const token = cfg.telegram_bot_token || process.env.TELEGRAM_BOT_TOKEN;
  if (!token) {
    console.log('Telegram bot token not configured');
    return Promise.resolve(null);
  }

  return new Promise((resolve, reject) => {
    const data = JSON.stringify(body);
    const req = https.request({
      hostname: 'api.telegram.org',
      path: `/bot${token}/` + method,
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Content-Length': data.length }
    }, (res) => {
      let body = '';
      res.on('data', chunk => body += chunk);
      res.on('end', () => {
        try { resolve(JSON.parse(body)); } catch { resolve(null); }
      });
    });
    req.on('error', reject);
    req.write(data);
    req.end();
  });
}

async function handleWebhook(req, res) {
  const update = req.body;

  // Handle callback query (inline button pressed)
  if (update.callback_query) {
    const callbackData = update.callback_query.data;
    const chatId = update.callback_query.message.chat.id;
    const messageId = update.callback_query.message.message_id;

    console.log(`Telegram callback: ${callbackData} from chat ${chatId}`);

    if (callbackData.startsWith('apply_')) {
      const jobId = callbackData.replace('apply_', '');

      // Answer the callback (removes loading state)
      await sendTelegramRequest('answerCallbackQuery', {
        callback_query_id: update.callback_query.id,
        text: 'Candidatura in coda!'
      });

      // Update the message to show it's queued
      await sendTelegramRequest('editMessageReplyMarkup', {
        chat_id: chatId,
        message_id: messageId,
        reply_markup: { inline_keyboard: [[{ text: '📝 In coda...', callback_data: 'noop' }]] }
      });

      // Queue the application (store job info for the routine to pick up)
      await queueApplication(jobId, chatId, update.callback_query.message.text);

      // TODO: Trigger Application Agent Routine via API
      // This will be configured once the Routine is created and we have the API endpoint

    } else if (callbackData.startsWith('ignore_')) {
      await sendTelegramRequest('answerCallbackQuery', {
        callback_query_id: update.callback_query.id,
        text: 'Ignorato'
      });
      await sendTelegramRequest('editMessageReplyMarkup', {
        chat_id: chatId,
        message_id: messageId,
        reply_markup: { inline_keyboard: [[{ text: '❌ Ignorato', callback_data: 'noop' }]] }
      });

    } else if (callbackData === 'noop') {
      await sendTelegramRequest('answerCallbackQuery', {
        callback_query_id: update.callback_query.id
      });
    }
  }

  // Handle text messages (responses to unusual questions)
  if (update.message && update.message.text && update.message.reply_to_message) {
    const chatId = update.message.chat.id;
    const responseText = update.message.text;
    const originalMessage = update.message.reply_to_message.text || '';

    console.log(`Telegram response: "${responseText}" to question in message`);

    // Store the answer for the knowledge base
    // The Application Agent routine will pick this up
    await storeUserResponse(chatId, responseText, originalMessage);

    await sendTelegramRequest('sendMessage', {
      chat_id: chatId,
      text: '✅ Risposta salvata! La candidatura riprenderà a breve.',
      reply_to_message_id: update.message.message_id
    });
  }

  res.json({ ok: true });
}

async function queueApplication(jobId, chatId, messageText) {
  // Extract job info from the notification message
  const cfg = config.get();
  const queue = cfg.application_queue || [];

  // Parse basic info from jobId (format: site_hash)
  queue.push({
    job_id: jobId,
    chat_id: chatId,
    requested_at: new Date().toISOString(),
    status: 'pending'
  });

  config.update({ application_queue: queue });
  console.log(`Application queued: ${jobId}`);
}

async function storeUserResponse(chatId, response, originalQuestion) {
  const cfg = config.get();
  const pendingResponses = cfg.pending_responses || [];

  pendingResponses.push({
    chat_id: chatId,
    response,
    original_question: originalQuestion,
    timestamp: new Date().toISOString()
  });

  config.update({ pending_responses: pendingResponses });
}

async function registerWebhook(webhookUrl) {
  const result = await sendTelegramRequest('setWebhook', {
    url: webhookUrl,
    allowed_updates: ['callback_query', 'message']
  });
  console.log('Webhook registration:', result);
  return result;
}

module.exports = { handleWebhook, registerWebhook, sendTelegramRequest };
```

- [ ] **Step 2: Commit**

```bash
git add src/telegram.js
git commit -m "feat: add Telegram webhook handler for inline buttons"
```

---

### Task 4: Mount new routes in server.js (scraper-proxy)

**Files:**
- Modify: `src/server.js`

- [ ] **Step 1: Add session routes and telegram webhook to server.js**

Add after the existing requires at the top:
```javascript
const sessionRoutes = require('./session-routes');
const telegram = require('./telegram');
```

Add after the admin routes (after line ~175):
```javascript
// Session management routes
app.use('/session', sessionRoutes);

// Telegram webhook
app.post('/telegram/webhook', express.json(), telegram.handleWebhook);

// Register Telegram webhook on startup (if configured)
app.get('/telegram/register', async (req, res) => {
  const webhookUrl = `https://scraper.followtheflowai.com/telegram/webhook`;
  const result = await telegram.registerWebhook(webhookUrl);
  res.json(result);
});
```

- [ ] **Step 2: Commit**

```bash
git add src/server.js
git commit -m "feat: mount session routes and telegram webhook"
```

---

### Task 5: Add account credentials to admin dashboard (scraper-proxy)

**Files:**
- Modify: `public/admin.html`

- [ ] **Step 1: Add accounts section to the dashboard**

Add a new section in the dashboard HTML after the Whitelist section and before the Admin Credentials section. The section manages site credentials (site name, username, password) stored in config.site_credentials array.

Features:
- List of saved credentials with site, username, masked password
- Add new credential form (site, username, password)
- Remove credential button
- All saved in config via PUT /admin/api/config

- [ ] **Step 2: Commit**

```bash
git add public/admin.html
git commit -m "feat: add account credentials section to admin dashboard"
```

---

### Task 6: Update residential proxy with session support (Mac)

**Files:**
- Modify: `proxy.js` in `/Users/stefanorussello/Documents/Projects/FollowTheFlow/residential-proxy/`

- [ ] **Step 1: Add session endpoints to Mac proxy**

Add the same session management functionality to the Mac proxy. The Mac proxy needs to handle:
- POST /session/create → creates local Playwright session
- POST /session/:id/goto → navigates
- GET /session/:id/content → returns page content
- POST /session/:id/fill → fills a field
- POST /session/:id/click → clicks an element
- POST /session/:id/select → selects dropdown option
- POST /session/:id/upload → uploads file
- GET /session/:id/screenshot → takes screenshot
- DELETE /session/:id → closes session

Use the same session manager pattern as the VPS but with Chromium (already working on Mac).

- [ ] **Step 2: Restart LaunchAgent**

```bash
launchctl unload ~/Library/LaunchAgents/com.followtheflow.residential-proxy.plist
launchctl load ~/Library/LaunchAgents/com.followtheflow.residential-proxy.plist
```

- [ ] **Step 3: Commit (no git repo for residential-proxy, just restart)**

---

### Task 7: Add session routing for residential domains (scraper-proxy)

**Files:**
- Modify: `src/sessions.js`

- [ ] **Step 1: Update session.create to route to Mac for residential domains**

When `use_residential: true` is passed and the config has residential_proxy enabled, forward all session commands to the Mac proxy via Tailscale SOCKS5 instead of creating a local session.

The VPS session manager acts as a proxy: it receives session commands from the Routine and forwards them to the Mac's session endpoints via SOCKS5.

- [ ] **Step 2: Commit**

```bash
git add src/sessions.js
git commit -m "feat: add residential routing for interactive sessions"
```

---

### Task 8: Create job-monitor config files

**Files (in job-monitor repo):**
- Create: `config/application_profile.json`
- Create: `config/knowledge_base.json`
- Create: `config/application_settings.json`
- Create: `data/application_queue.json`
- Create: `data/applications.json`
- Create: `assets/` directory

- [ ] **Step 1: Create all config files with content from the spec**

See the spec for exact JSON content of each file.

- [ ] **Step 2: Copy CV to assets/**

```bash
cp "/Users/stefanorussello/Downloads/CV Stefano Russello English 2025-11.pdf" assets/CV_Stefano_Russello_EN.pdf
```

- [ ] **Step 3: Commit**

```bash
git add config/application_profile.json config/knowledge_base.json config/application_settings.json data/application_queue.json data/applications.json assets/
git commit -m "feat: add application agent config files and CV"
```

---

### Task 9: Update Job Monitor notifications with inline buttons

**Files:**
- Modify: `routine/prompt.md` in job-monitor repo

- [ ] **Step 1: Update Step 6 notification format**

Change the Telegram sendMessage calls to include `reply_markup` with inline keyboard buttons:

```json
{
  "chat_id": "181648234",
  "parse_mode": "HTML",
  "text": "<message>",
  "reply_markup": {
    "inline_keyboard": [[
      {"text": "📝 Applica", "callback_data": "apply_{job_id}"},
      {"text": "❌ Ignora", "callback_data": "ignore_{job_id}"}
    ]]
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add routine/prompt.md
git commit -m "feat: add inline Apply/Ignore buttons to Telegram notifications"
```

---

### Task 10: Create Application Agent routine prompt

**Files:**
- Create: `routine/application_prompt.md` in job-monitor repo

- [ ] **Step 1: Create the application agent prompt**

This is the main prompt for the Application Agent Routine. It reads the queue, processes applications one by one using the session API on the scraper-proxy, and reports results.

The prompt must include:
- How to read config files (profile, knowledge_base, settings)
- How to process the application queue
- How to create and use browser sessions via the scraper-proxy API
- The adaptive form-filling loop
- How to handle standard fields, open questions, unusual questions
- How to generate cover letters
- Salary strategy
- Error handling and reporting
- How to update knowledge_base with learned answers
- Language adaptation (IT/ES/EN)

- [ ] **Step 2: Commit**

```bash
git add routine/application_prompt.md
git commit -m "feat: add Application Agent routine prompt"
```

---

### Task 11: Deploy scraper-proxy updates

- [ ] **Step 1: Push scraper-proxy to GitHub**

```bash
cd /Users/stefanorussello/Documents/Projects/FollowTheFlow/scraper-proxy
git push origin main
```

- [ ] **Step 2: Redeploy on EasyPanel**

Trigger rebuild on EasyPanel for the scraper app.

- [ ] **Step 3: Register Telegram webhook**

```bash
curl https://scraper.followtheflowai.com/telegram/register
```

- [ ] **Step 4: Verify session endpoints**

```bash
# Create session
curl -s -X POST http://scraper.followtheflowai.com/session/create \
  -H "X-API-Key: sk-81d2043b4412c7adf7f6279eeefb2385" \
  -H "Content-Type: application/json" \
  -d '{"use_residential": false}'

# Navigate
curl -s -X POST http://scraper.followtheflowai.com/session/{id}/goto \
  -H "X-API-Key: sk-81d2043b4412c7adf7f6279eeefb2385" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com"}'

# Close
curl -s -X DELETE http://scraper.followtheflowai.com/session/{id} \
  -H "X-API-Key: sk-81d2043b4412c7adf7f6279eeefb2385"
```

---

### Task 12: Push job-monitor updates

- [ ] **Step 1: Push all changes**

```bash
cd /Users/stefanorussello/Documents/Projects/FollowTheFlow/LinkedIn-Lavoro
git push origin main
```

---

### Task 13: Configure Application Agent Routine

This is done manually on claude.ai/code or the Mac app:

- [ ] **Step 1: Create new Routine "Application Agent"**

Configuration:
- Name: Application Agent
- Repository: Spettacolo83/job-monitor
- Trigger: API (not cron)
- Prompt: "Read and follow the instructions in routine/application_prompt.md. Execute all steps."

- [ ] **Step 2: Add environment variables**

- `TELEGRAM_BOT_TOKEN` = (same as Job Monitor)
- `SCRAPER_API_KEY` = sk-81d2043b4412c7adf7f6279eeefb2385

- [ ] **Step 3: Note the API endpoint and token**

Save the Routine's API endpoint URL and auth token — needed by the webhook to trigger it.

---

### Task 14: End-to-end test

- [ ] **Step 1: Trigger Job Monitor to send a notification with buttons**

Run Job Monitor manually. Verify the notification on Telegram shows [Applica] [Ignora] buttons.

- [ ] **Step 2: Press "Applica" and verify the queue**

Press the "Applica" button. Verify:
- Button changes to "In coda..."
- Application appears in application_queue (on scraper-proxy config)

- [ ] **Step 3: Trigger Application Agent manually**

Run the Application Agent routine manually. Verify it:
- Reads the queue
- Opens a browser session
- Navigates to the job page
- Attempts to fill the application
- Reports result on Telegram
