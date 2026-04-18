# Scraper Proxy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Playwright-based scraping proxy service with admin dashboard, deployable on EasyPanel via Docker.

**Architecture:** Node.js + Express API backed by Playwright Chromium for headless page rendering. Single browser instance with page pool for concurrency. Admin dashboard as static HTML page. Config persisted to JSON file on Docker volume.

**Tech Stack:** Node.js 20, Express, Playwright, Docker (mcr.microsoft.com/playwright base image), EasyPanel for deploy.

**Working directory:** Create a NEW directory at `/Users/stefanorussello/Documents/Projects/FollowTheFlow/scraper-proxy` — this is a separate project from the job-monitor.

---

### Task 1: Create repository and project structure

**Files:**
- Create: `package.json`
- Create: `.gitignore`
- Create: `.dockerignore`
- Create: `README.md`
- Create: `config.default.json`

- [ ] **Step 1: Create project directory and initialize git**

```bash
mkdir -p /Users/stefanorussello/Documents/Projects/FollowTheFlow/scraper-proxy
cd /Users/stefanorussello/Documents/Projects/FollowTheFlow/scraper-proxy
git init
```

- [ ] **Step 2: Create package.json**

```json
{
  "name": "scraper-proxy",
  "version": "1.0.0",
  "description": "Playwright-based scraping proxy with admin dashboard",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "node --watch src/server.js"
  },
  "dependencies": {
    "express": "^4.21.0",
    "playwright": "^1.52.0",
    "cookie-session": "^2.1.0",
    "crypto": "^1.0.1"
  },
  "engines": {
    "node": ">=20"
  }
}
```

- [ ] **Step 3: Create config.default.json**

```json
{
  "admin": {
    "username": "admin",
    "password": "changeme"
  },
  "api_keys": [
    {
      "key": "default-key-change-me",
      "name": "Default API Key",
      "created": "2026-04-18"
    }
  ],
  "rate_limit": {
    "max_requests_per_minute": 30,
    "max_concurrent": 3
  },
  "whitelist": [],
  "browser": {
    "timeout_ms": 30000,
    "wait_after_load_ms": 2000,
    "user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36"
  }
}
```

- [ ] **Step 4: Create .gitignore**

```
node_modules/
data/
.DS_Store
*.log
```

- [ ] **Step 5: Create .dockerignore**

```
node_modules/
data/
.git/
.DS_Store
*.md
```

- [ ] **Step 6: Create README.md**

```markdown
# Scraper Proxy

Playwright-based scraping proxy that renders pages with a headless Chromium browser, bypassing Cloudflare and JS-rendered content.

## API

### GET /fetch?url=<encoded_url>[&format=text]
Renders the URL and returns the HTML (or text if format=text).

Requires `X-API-Key` header.

### GET /health
Health check endpoint (no auth required).

### GET /admin
Admin dashboard for managing API keys, rate limits, and whitelist.

## Deploy

Built for EasyPanel deployment via Docker. See Dockerfile.

## Config

On first start, copies `config.default.json` to `data/config.json`. All config changes via the admin dashboard are persisted to this file.
```

- [ ] **Step 7: Commit**

```bash
git add package.json config.default.json .gitignore .dockerignore README.md
git commit -m "chore: initial project structure"
```

---

### Task 2: Implement config module

**Files:**
- Create: `src/config.js`

- [ ] **Step 1: Create src directory**

```bash
mkdir -p src
```

- [ ] **Step 2: Create src/config.js**

```javascript
const fs = require('fs');
const path = require('path');

const DATA_DIR = path.join(__dirname, '..', 'data');
const CONFIG_PATH = path.join(DATA_DIR, 'config.json');
const DEFAULT_CONFIG_PATH = path.join(__dirname, '..', 'config.default.json');

let config = null;

function ensureDataDir() {
  if (!fs.existsSync(DATA_DIR)) {
    fs.mkdirSync(DATA_DIR, { recursive: true });
  }
}

function load() {
  ensureDataDir();
  if (!fs.existsSync(CONFIG_PATH)) {
    const defaultConfig = fs.readFileSync(DEFAULT_CONFIG_PATH, 'utf-8');
    fs.writeFileSync(CONFIG_PATH, defaultConfig);
  }
  config = JSON.parse(fs.readFileSync(CONFIG_PATH, 'utf-8'));
  return config;
}

function get() {
  if (!config) load();
  return config;
}

function save(newConfig) {
  ensureDataDir();
  config = newConfig;
  fs.writeFileSync(CONFIG_PATH, JSON.stringify(config, null, 2));
}

function update(partial) {
  const current = get();
  const updated = { ...current, ...partial };
  save(updated);
  return updated;
}

module.exports = { load, get, save, update };
```

- [ ] **Step 3: Commit**

```bash
git add src/config.js
git commit -m "feat: add config module with file persistence"
```

---

### Task 3: Implement logger module

**Files:**
- Create: `src/logger.js`

- [ ] **Step 1: Create src/logger.js**

```javascript
const MAX_LOGS = 100;
const logs = [];

function log(entry) {
  logs.unshift({
    timestamp: new Date().toISOString(),
    ...entry
  });
  if (logs.length > MAX_LOGS) {
    logs.length = MAX_LOGS;
  }
}

function getLogs() {
  return logs;
}

function clear() {
  logs.length = 0;
}

module.exports = { log, getLogs, clear };
```

- [ ] **Step 2: Commit**

```bash
git add src/logger.js
git commit -m "feat: add in-memory request logger (last 100)"
```

---

### Task 4: Implement rate limiter

**Files:**
- Create: `src/rate-limiter.js`

- [ ] **Step 1: Create src/rate-limiter.js**

```javascript
const config = require('./config');

const requestTimestamps = [];
let activeRequests = 0;
const queue = [];

function cleanup() {
  const now = Date.now();
  while (requestTimestamps.length > 0 && now - requestTimestamps[0] > 60000) {
    requestTimestamps.shift();
  }
}

function canProceed() {
  const cfg = config.get().rate_limit;
  cleanup();
  if (requestTimestamps.length >= cfg.max_requests_per_minute) {
    return { allowed: false, reason: 'Rate limit exceeded (max per minute)' };
  }
  if (activeRequests >= cfg.max_concurrent) {
    return { allowed: false, reason: 'Max concurrent requests reached' };
  }
  return { allowed: true };
}

function acquire() {
  requestTimestamps.push(Date.now());
  activeRequests++;
}

function release() {
  activeRequests--;
  processQueue();
}

function enqueue(resolve, reject) {
  if (queue.length >= 10) {
    reject(new Error('Queue full'));
    return;
  }
  queue.push({ resolve, reject });
}

function processQueue() {
  if (queue.length === 0) return;
  const check = canProceed();
  if (check.allowed) {
    const { resolve } = queue.shift();
    acquire();
    resolve();
  }
}

function getStats() {
  cleanup();
  return {
    requests_last_minute: requestTimestamps.length,
    active_requests: activeRequests,
    queued: queue.length
  };
}

module.exports = { canProceed, acquire, release, enqueue, getStats };
```

- [ ] **Step 2: Commit**

```bash
git add src/rate-limiter.js
git commit -m "feat: add rate limiter with concurrency control and queue"
```

---

### Task 5: Implement auth middleware

**Files:**
- Create: `src/auth.js`

- [ ] **Step 1: Create src/auth.js**

```javascript
const config = require('./config');

function apiKeyAuth(req, res, next) {
  const apiKey = req.headers['x-api-key'];
  if (!apiKey) {
    return res.status(401).json({ success: false, error: 'Missing X-API-Key header' });
  }
  const cfg = config.get();
  const keyEntry = cfg.api_keys.find(k => k.key === apiKey);
  if (!keyEntry) {
    return res.status(401).json({ success: false, error: 'Invalid API key' });
  }
  req.apiKeyName = keyEntry.name;
  next();
}

function adminAuth(req, res, next) {
  if (req.session && req.session.admin) {
    return next();
  }
  res.status(401).json({ success: false, error: 'Not authenticated' });
}

function adminLogin(req, res) {
  const { username, password } = req.body;
  const cfg = config.get();
  if (username === cfg.admin.username && password === cfg.admin.password) {
    req.session.admin = true;
    return res.json({ success: true });
  }
  res.status(401).json({ success: false, error: 'Invalid credentials' });
}

function adminLogout(req, res) {
  req.session = null;
  res.json({ success: true });
}

module.exports = { apiKeyAuth, adminAuth, adminLogin, adminLogout };
```

- [ ] **Step 2: Commit**

```bash
git add src/auth.js
git commit -m "feat: add API key and admin session auth middleware"
```

---

### Task 6: Implement scraper module

**Files:**
- Create: `src/scraper.js`

- [ ] **Step 1: Create src/scraper.js**

```javascript
const { chromium } = require('playwright');
const config = require('./config');

let browser = null;

async function launch() {
  const cfg = config.get().browser;
  browser = await chromium.launch({
    headless: true,
    args: ['--no-sandbox', '--disable-setuid-sandbox', '--disable-dev-shm-usage']
  });
  console.log('Browser launched');
  browser.on('disconnected', () => {
    console.log('Browser disconnected, relaunching...');
    browser = null;
    setTimeout(launch, 1000);
  });
}

async function fetchPage(url, format = 'html') {
  if (!browser) {
    await launch();
  }

  const cfg = config.get().browser;
  const context = await browser.newContext({
    userAgent: cfg.user_agent,
    viewport: { width: 1280, height: 720 }
  });

  const page = await context.newPage();
  const startTime = Date.now();

  try {
    const response = await page.goto(url, {
      waitUntil: 'domcontentloaded',
      timeout: cfg.timeout_ms
    });

    // Wait extra time for Cloudflare challenges / JS rendering
    await page.waitForTimeout(cfg.wait_after_load_ms);

    const statusCode = response ? response.status() : 0;
    let content;

    if (format === 'text') {
      content = await page.innerText('body');
    } else {
      content = await page.content();
    }

    const elapsed = Date.now() - startTime;

    return {
      success: true,
      url,
      [format === 'text' ? 'text' : 'html']: content,
      status_code: statusCode,
      elapsed_ms: elapsed
    };
  } catch (error) {
    const elapsed = Date.now() - startTime;
    return {
      success: false,
      url,
      error: error.message,
      elapsed_ms: elapsed
    };
  } finally {
    await context.close();
  }
}

async function close() {
  if (browser) {
    await browser.close();
    browser = null;
  }
}

module.exports = { launch, fetchPage, close };
```

- [ ] **Step 2: Commit**

```bash
git add src/scraper.js
git commit -m "feat: add Playwright scraper with browser pool and auto-restart"
```

---

### Task 7: Implement Express server with all routes

**Files:**
- Create: `src/server.js`

- [ ] **Step 1: Create src/server.js**

```javascript
const express = require('express');
const cookieSession = require('cookie-session');
const crypto = require('crypto');
const path = require('path');
const config = require('./config');
const scraper = require('./scraper');
const { apiKeyAuth, adminAuth, adminLogin, adminLogout } = require('./auth');
const rateLimiter = require('./rate-limiter');
const logger = require('./logger');

// Load config
config.load();

const app = express();
app.use(express.json());

// Session for admin
app.use(cookieSession({
  name: 'scraper-admin',
  keys: [crypto.randomBytes(32).toString('hex')],
  maxAge: 24 * 60 * 60 * 1000 // 24 hours
}));

// Serve static admin page
app.use('/admin', express.static(path.join(__dirname, '..', 'public')));

// Health check (no auth)
app.get('/health', (req, res) => {
  res.json({ status: 'ok', stats: rateLimiter.getStats() });
});

// Fetch endpoint (requires API key)
app.get('/fetch', apiKeyAuth, async (req, res) => {
  const url = req.query.url;
  const format = req.query.format || 'html';

  if (!url) {
    return res.status(400).json({ success: false, error: 'Missing url parameter' });
  }

  try {
    new URL(url);
  } catch {
    return res.status(400).json({ success: false, error: 'Invalid URL' });
  }

  // Check whitelist
  const cfg = config.get();
  if (cfg.whitelist.length > 0) {
    const hostname = new URL(url).hostname;
    const allowed = cfg.whitelist.some(domain =>
      hostname === domain || hostname.endsWith('.' + domain)
    );
    if (!allowed) {
      return res.status(403).json({ success: false, error: `Domain ${hostname} not in whitelist` });
    }
  }

  // Check rate limit
  const check = rateLimiter.canProceed();
  if (!check.allowed) {
    logger.log({ url, api_key: req.apiKeyName, status: 429, elapsed_ms: 0 });
    return res.status(429).json({ success: false, error: check.reason });
  }

  rateLimiter.acquire();
  try {
    const result = await scraper.fetchPage(url, format);
    logger.log({
      url,
      api_key: req.apiKeyName,
      status: result.success ? result.status_code : 'error',
      elapsed_ms: result.elapsed_ms
    });
    res.json(result);
  } catch (error) {
    logger.log({ url, api_key: req.apiKeyName, status: 'error', elapsed_ms: 0 });
    res.status(500).json({ success: false, error: error.message });
  } finally {
    rateLimiter.release();
  }
});

// Admin login
app.post('/admin/login', adminLogin);
app.post('/admin/logout', adminLogout);

// Admin API (requires login)
app.get('/admin/api/config', adminAuth, (req, res) => {
  const cfg = { ...config.get() };
  // Don't expose admin password in API response
  cfg.admin = { ...cfg.admin, password: '********' };
  res.json(cfg);
});

app.put('/admin/api/config', adminAuth, (req, res) => {
  const current = config.get();
  const updates = req.body;

  // Merge carefully - don't allow overwriting admin password with masked value
  if (updates.admin) {
    if (updates.admin.password === '********') {
      updates.admin.password = current.admin.password;
    }
  }

  const updated = config.update(updates);
  const safe = { ...updated };
  safe.admin = { ...safe.admin, password: '********' };
  res.json(safe);
});

app.get('/admin/api/logs', adminAuth, (req, res) => {
  res.json(logger.getLogs());
});

// Start server
const PORT = process.env.PORT || 3000;

async function start() {
  await scraper.launch();
  app.listen(PORT, () => {
    console.log(`Scraper proxy running on port ${PORT}`);
  });
}

start().catch(err => {
  console.error('Failed to start:', err);
  process.exit(1);
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('Shutting down...');
  await scraper.close();
  process.exit(0);
});
```

- [ ] **Step 2: Commit**

```bash
git add src/server.js
git commit -m "feat: add Express server with fetch, health, and admin routes"
```

---

### Task 8: Create admin dashboard HTML

**Files:**
- Create: `public/admin.html`

- [ ] **Step 1: Create public directory**

```bash
mkdir -p public
```

- [ ] **Step 2: Create public/admin.html**

This is a single HTML file with embedded CSS and JavaScript. Dark mode, responsive, no external dependencies.

The file should contain:
- Login form (shown when not authenticated)
- Dashboard with 5 sections: API Keys, Rate Limiting, Whitelist, Logs, Admin Credentials
- All sections use fetch() to call the admin API endpoints
- Auto-refresh logs every 10 seconds
- Dark mode with monospace fonts for technical data
- Ability to add/remove API keys, update rate limits, add/remove whitelist domains, change admin credentials

Full HTML content (see implementation step below for actual code).

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Scraper Proxy Admin</title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; background: #1a1a2e; color: #e0e0e0; min-height: 100vh; }
  .login-container { display: flex; justify-content: center; align-items: center; min-height: 100vh; }
  .login-box { background: #16213e; padding: 2rem; border-radius: 8px; width: 320px; }
  .login-box h2 { margin-bottom: 1rem; color: #4ecca3; }
  input, button { width: 100%; padding: 0.6rem; margin: 0.3rem 0; border: 1px solid #333; border-radius: 4px; font-size: 14px; }
  input { background: #0f3460; color: #e0e0e0; }
  input:focus { outline: none; border-color: #4ecca3; }
  button { background: #4ecca3; color: #1a1a2e; border: none; cursor: pointer; font-weight: bold; }
  button:hover { background: #3ba88a; }
  button.danger { background: #e74c3c; color: white; }
  button.danger:hover { background: #c0392b; }
  button.small { width: auto; padding: 0.3rem 0.8rem; font-size: 12px; }
  .dashboard { max-width: 900px; margin: 0 auto; padding: 1rem; }
  .header { display: flex; justify-content: space-between; align-items: center; padding: 1rem 0; border-bottom: 1px solid #333; margin-bottom: 1rem; }
  .header h1 { color: #4ecca3; font-size: 1.4rem; }
  .section { background: #16213e; border-radius: 8px; padding: 1.2rem; margin-bottom: 1rem; }
  .section h3 { color: #4ecca3; margin-bottom: 0.8rem; font-size: 1rem; }
  .row { display: flex; gap: 0.5rem; align-items: center; margin: 0.4rem 0; flex-wrap: wrap; }
  .tag { background: #0f3460; padding: 0.3rem 0.6rem; border-radius: 4px; font-family: monospace; font-size: 13px; display: inline-flex; align-items: center; gap: 0.4rem; }
  .tag .remove { cursor: pointer; color: #e74c3c; font-weight: bold; }
  table { width: 100%; border-collapse: collapse; font-size: 13px; font-family: monospace; }
  th, td { padding: 0.4rem 0.6rem; text-align: left; border-bottom: 1px solid #333; }
  th { color: #4ecca3; }
  .status-ok { color: #4ecca3; }
  .status-err { color: #e74c3c; }
  .badge { background: #4ecca3; color: #1a1a2e; padding: 0.15rem 0.5rem; border-radius: 10px; font-size: 11px; font-weight: bold; }
  .badge.off { background: #666; }
  .inline-form { display: flex; gap: 0.5rem; margin-top: 0.5rem; }
  .inline-form input { width: auto; flex: 1; }
  .inline-form button { width: auto; }
  .hidden { display: none; }
  .msg { padding: 0.5rem; border-radius: 4px; margin: 0.5rem 0; font-size: 13px; }
  .msg.ok { background: #1a4a3a; color: #4ecca3; }
  .msg.err { background: #4a1a1a; color: #e74c3c; }
</style>
</head>
<body>

<!-- Login -->
<div id="login" class="login-container">
  <div class="login-box">
    <h2>Scraper Proxy</h2>
    <input type="text" id="username" placeholder="Username" autocomplete="username">
    <input type="password" id="password" placeholder="Password" autocomplete="current-password">
    <button onclick="login()">Login</button>
    <div id="login-msg"></div>
  </div>
</div>

<!-- Dashboard -->
<div id="dashboard" class="dashboard hidden">
  <div class="header">
    <h1>Scraper Proxy Admin</h1>
    <button class="small danger" onclick="logout()">Logout</button>
  </div>

  <!-- API Keys -->
  <div class="section">
    <h3>API Keys</h3>
    <div id="keys-list"></div>
    <div class="inline-form">
      <input type="text" id="new-key-name" placeholder="Nome chiave (es. Job Monitor)">
      <button class="small" onclick="addKey()">+ Aggiungi</button>
    </div>
  </div>

  <!-- Rate Limiting -->
  <div class="section">
    <h3>Rate Limiting</h3>
    <div class="row">
      <label>Max richieste/min:</label>
      <input type="number" id="rate-rpm" style="width:80px">
      <label>Max concorrenti:</label>
      <input type="number" id="rate-concurrent" style="width:80px">
      <button class="small" onclick="saveRateLimit()">Salva</button>
    </div>
  </div>

  <!-- Whitelist -->
  <div class="section">
    <h3>Whitelist Domini <span id="wl-badge"></span></h3>
    <div id="whitelist-list"></div>
    <div class="inline-form">
      <input type="text" id="new-domain" placeholder="es. indeed.com">
      <button class="small" onclick="addDomain()">+ Aggiungi</button>
    </div>
  </div>

  <!-- Admin Credentials -->
  <div class="section">
    <h3>Credenziali Admin</h3>
    <div class="row">
      <input type="text" id="admin-user" placeholder="Username" style="width:200px">
      <input type="password" id="admin-pass" placeholder="Nuova password" style="width:200px">
      <button class="small" onclick="saveAdmin()">Aggiorna</button>
    </div>
    <div id="admin-msg"></div>
  </div>

  <!-- Logs -->
  <div class="section">
    <h3>Log Richieste (ultime 100)</h3>
    <table>
      <thead><tr><th>Timestamp</th><th>URL</th><th>API Key</th><th>Status</th><th>ms</th></tr></thead>
      <tbody id="logs-body"></tbody>
    </table>
  </div>
</div>

<script>
let cfg = {};

async function api(method, path, body) {
  const opts = { method, headers: { 'Content-Type': 'application/json' } };
  if (body) opts.body = JSON.stringify(body);
  const r = await fetch(path, opts);
  if (r.status === 401 && path !== '/admin/login') { showLogin(); return null; }
  return r.json();
}

function showLogin() {
  document.getElementById('login').classList.remove('hidden');
  document.getElementById('dashboard').classList.add('hidden');
}

function showDashboard() {
  document.getElementById('login').classList.add('hidden');
  document.getElementById('dashboard').classList.remove('hidden');
  loadConfig();
  loadLogs();
}

async function login() {
  const u = document.getElementById('username').value;
  const p = document.getElementById('password').value;
  const r = await api('POST', '/admin/login', { username: u, password: p });
  if (r && r.success) showDashboard();
  else document.getElementById('login-msg').innerHTML = '<div class="msg err">Credenziali errate</div>';
}

async function logout() {
  await api('POST', '/admin/logout');
  showLogin();
}

async function loadConfig() {
  cfg = await api('GET', '/admin/api/config');
  if (!cfg) return;
  renderKeys();
  document.getElementById('rate-rpm').value = cfg.rate_limit.max_requests_per_minute;
  document.getElementById('rate-concurrent').value = cfg.rate_limit.max_concurrent;
  renderWhitelist();
  document.getElementById('admin-user').value = cfg.admin.username;
}

function renderKeys() {
  const el = document.getElementById('keys-list');
  el.innerHTML = cfg.api_keys.map((k, i) =>
    `<div class="row"><span class="tag">${k.name}: <code>${k.key}</code> <span class="remove" onclick="removeKey(${i})">✕</span></span></div>`
  ).join('');
}

function renderWhitelist() {
  const el = document.getElementById('whitelist-list');
  const badge = document.getElementById('wl-badge');
  if (cfg.whitelist.length === 0) {
    badge.innerHTML = '<span class="badge off">OFF — tutti i domini</span>';
    el.innerHTML = '<p style="font-size:13px;color:#888">Nessuna restrizione attiva</p>';
  } else {
    badge.innerHTML = '<span class="badge">ON</span>';
    el.innerHTML = cfg.whitelist.map((d, i) =>
      `<span class="tag">${d} <span class="remove" onclick="removeDomain(${i})">✕</span></span> `
    ).join('');
  }
}

async function addKey() {
  const name = document.getElementById('new-key-name').value.trim();
  if (!name) return;
  const key = 'sk-' + Array.from(crypto.getRandomValues(new Uint8Array(16))).map(b => b.toString(16).padStart(2, '0')).join('');
  cfg.api_keys.push({ key, name, created: new Date().toISOString().split('T')[0] });
  await api('PUT', '/admin/api/config', { api_keys: cfg.api_keys });
  document.getElementById('new-key-name').value = '';
  loadConfig();
}

async function removeKey(index) {
  if (cfg.api_keys.length <= 1) return alert('Serve almeno una API key');
  cfg.api_keys.splice(index, 1);
  await api('PUT', '/admin/api/config', { api_keys: cfg.api_keys });
  loadConfig();
}

async function saveRateLimit() {
  const rpm = parseInt(document.getElementById('rate-rpm').value);
  const conc = parseInt(document.getElementById('rate-concurrent').value);
  await api('PUT', '/admin/api/config', { rate_limit: { max_requests_per_minute: rpm, max_concurrent: conc } });
  loadConfig();
}

async function addDomain() {
  const d = document.getElementById('new-domain').value.trim().toLowerCase();
  if (!d) return;
  cfg.whitelist.push(d);
  await api('PUT', '/admin/api/config', { whitelist: cfg.whitelist });
  document.getElementById('new-domain').value = '';
  loadConfig();
}

async function removeDomain(index) {
  cfg.whitelist.splice(index, 1);
  await api('PUT', '/admin/api/config', { whitelist: cfg.whitelist });
  loadConfig();
}

async function saveAdmin() {
  const u = document.getElementById('admin-user').value.trim();
  const p = document.getElementById('admin-pass').value;
  const updates = { admin: { username: u } };
  if (p) updates.admin.password = p;
  else updates.admin.password = '********';
  await api('PUT', '/admin/api/config', updates);
  document.getElementById('admin-msg').innerHTML = '<div class="msg ok">Aggiornato</div>';
  setTimeout(() => document.getElementById('admin-msg').innerHTML = '', 3000);
}

async function loadLogs() {
  const logs = await api('GET', '/admin/api/logs');
  if (!logs) return;
  const el = document.getElementById('logs-body');
  el.innerHTML = logs.map(l => {
    const st = (l.status === 'error' || l.status >= 400) ? 'status-err' : 'status-ok';
    const url = l.url.length > 60 ? l.url.substring(0, 60) + '...' : l.url;
    return `<tr><td>${l.timestamp.replace('T',' ').substring(0,19)}</td><td>${url}</td><td>${l.api_key||'-'}</td><td class="${st}">${l.status}</td><td>${l.elapsed_ms}</td></tr>`;
  }).join('');
}

// Auto-refresh logs
setInterval(loadLogs, 10000);

// Check if already logged in
api('GET', '/admin/api/config').then(r => { if (r) showDashboard(); });
</script>
</body>
</html>
```

- [ ] **Step 3: Commit**

```bash
git add public/admin.html
git commit -m "feat: add admin dashboard with dark mode UI"
```

---

### Task 9: Create Dockerfile

**Files:**
- Create: `Dockerfile`

- [ ] **Step 1: Create Dockerfile**

```dockerfile
FROM mcr.microsoft.com/playwright:v1.52.0-noble

WORKDIR /app

COPY package.json ./
RUN npm install --production

COPY . .

RUN mkdir -p /app/data

EXPOSE 3000

CMD ["node", "src/server.js"]
```

- [ ] **Step 2: Commit**

```bash
git add Dockerfile
git commit -m "feat: add Dockerfile based on Playwright image"
```

---

### Task 10: Create GitHub repo and push

- [ ] **Step 1: Create remote repository**

```bash
gh repo create Spettacolo83/scraper-proxy --public --description "Playwright-based scraping proxy with admin dashboard" --clone=false
```

- [ ] **Step 2: Add remote and push**

```bash
git remote add origin https://github.com/Spettacolo83/scraper-proxy.git
git branch -M main
git push -u origin main
```

---

### Task 11: Deploy on EasyPanel

This task is done manually on EasyPanel (https://panel.followtheflowai.com):

- [ ] **Step 1: Create project "scraper" on EasyPanel**

Click "+ Nuovo" on the dashboard → name it "scraper"

- [ ] **Step 2: Add app from GitHub**

In the scraper project, click "+" → App → GitHub → select `Spettacolo83/scraper-proxy` → branch `main` → Build method: Dockerfile

- [ ] **Step 3: Configure domain**

Go to app settings → Domains → add `scraper.followtheflowai.com` → enable HTTPS

- [ ] **Step 4: Configure volume**

Go to app settings → Volumes → mount `/app/data` as a persistent volume

- [ ] **Step 5: Set port**

Ensure the app exposes port 3000

- [ ] **Step 6: Deploy and verify**

Trigger build. Once running, verify:
```bash
curl https://scraper.followtheflowai.com/health
```
Expected: `{"status":"ok","stats":{"requests_last_minute":0,"active_requests":0,"queued":0}}`

- [ ] **Step 7: Access admin dashboard**

Go to `https://scraper.followtheflowai.com/admin` → login with admin / changeme → change password immediately → create API key "Job Monitor" → copy the generated key.

---

### Task 12: Integrate with Job Monitor

**Files (in job-monitor repo):**
- Modify: `config/sites.json` — re-enable Cloudflare sites with `use_proxy: true`
- Modify: `routine/prompt.md` — add proxy fetch instructions

- [ ] **Step 1: Add SCRAPER_API_KEY to GitHub secrets**

```bash
gh secret set SCRAPER_API_KEY --repo Spettacolo83/job-monitor --body "<the-api-key-from-step-11>"
```

- [ ] **Step 2: Update sites.json — re-enable Cloudflare sites with use_proxy flag**

For each disabled site, set `"enabled": true` and add `"use_proxy": true`. Remove `"disabled_reason"`.

Sites to re-enable with proxy:
- indeed_uk, indeed_it, indeed_de, indeed_es, indeed_fr
- infojobs_it, infojobs_es
- stepstone, glassdoor, monster, honeypot

Sites to keep WITHOUT proxy (they work with curl):
- remoteok, remotive, jobicy

Sites to add `use_proxy: true` (JS-rendered):
- welcometothejungle, landingjobs

- [ ] **Step 3: Update routine/prompt.md — add proxy fetch method**

In Step 2, add after the HTML sites section:

```markdown
### Sites with proxy (use_proxy: true)
For sites marked with `"use_proxy": true` in sites.json, use the scraper proxy instead of direct curl:

For each keyword in the site's `keywords` array:
\`\`\`bash
curl -s --max-time 60 "https://scraper.followtheflowai.com/fetch?url=$(python3 -c 'import urllib.parse; print(urllib.parse.quote(\"{base_url}?{param}={keyword}\", safe=\"\"))')" \
  -H "X-API-Key: ${SCRAPER_API_KEY}"
\`\`\`

The proxy returns JSON with an `html` field. Parse the HTML from the response to extract job listings.
Note: proxy requests take 3-8 seconds each, so use a longer timeout (60s).
```

- [ ] **Step 4: Commit and push job-monitor changes**

```bash
cd /Users/stefanorussello/Documents/Projects/FollowTheFlow/LinkedIn-Lavoro
git add config/sites.json routine/prompt.md
git commit -m "feat: integrate scraper proxy — re-enable 11 Cloudflare-protected sites"
git push origin main
```

- [ ] **Step 5: Test end-to-end**

Trigger a manual Job Monitor run and verify that:
- Indeed and other previously-blocked sites now return results
- Notifications include jobs from these new sources
- seen_jobs.json is updated and pushed
