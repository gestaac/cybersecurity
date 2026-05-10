# 00 — Beginner Burp Suite Quick-Start

> **Read this once before tomorrow.** Burp is the most important tool you'll use Day 3. Without it you can only do what the website's UI lets you. With it, you can change *every single request* before it leaves your computer.

---

## What Burp does (1 paragraph)

Your browser talks to Juice Shop using HTTP requests. Normally you can't see or change them. Burp is a **middleman**: your browser sends every request to Burp first, Burp shows it to you, then forwards it to Juice Shop. You can pause, edit, or replay any request — which is how 90% of Juice Shop challenges are solved.

```
Firefox  ──→  Burp (port 8080)  ──→  Juice Shop (port 3000)
            ↑ you can see + edit here
```

---

## Step 1 — Start Burp (1 minute)

1. On Kali, open a terminal and type:
   ```bash
   burpsuite &
   ```
   Or click *Applications → 03 - Web Application Analysis → burpsuite*.
2. A welcome window appears. Click **"Temporary project"** → **Next**.
3. Click **"Use Burp defaults"** → **Start Burp**.
4. The main Burp window opens. You'll see tabs across the top: *Dashboard*, *Target*, **Proxy**, **Repeater**, etc. Those two bolded ones are what you'll use tomorrow.

✅ **Success looks like:** Burp window open, no error popups.

---

## Step 2 — Tell Firefox to use Burp (3 minutes, do once)

1. In Firefox: click the menu (≡) → **Settings**.
2. In the search bar at the top, type **proxy**.
3. Scroll to **Network Settings** → click **Settings...** button.
4. Choose **Manual proxy configuration**:
   - HTTP Proxy: `127.0.0.1`   Port: `8080`
   - ☑ tick **"Also use this proxy for HTTPS"**
   - No Proxy For: leave empty
5. Click **OK**.

✅ **Success looks like:** browse to `http://127.0.0.1:3000` → page loads. Switch to Burp → click *Proxy* tab → click **HTTP history** sub-tab → you see a list of requests. **Those are your Firefox requests being captured.**

❌ **If it doesn't work:** Burp not started yet, or wrong port number, or "Also use this proxy for HTTPS" not ticked.

---

## Step 3 — Import Burp's CA certificate (5 minutes, do once)

Without this, every HTTPS site shows a scary "certificate error" because Burp is technically a man-in-the-middle.

1. With Firefox proxying through Burp, type in URL bar: `http://burp` → press Enter.
2. Top-right corner of that page → click **"CA Certificate"** → a file `cacert.der` downloads.
3. In Firefox: Settings → search **"certificate"** → **View Certificates** → **Authorities** tab → **Import**.
4. Pick `cacert.der`.
5. ☑ tick **"Trust this CA to identify websites"** → **OK**.
6. Restart Firefox.

✅ **Success looks like:** browse to `https://www.google.com` → no warning, page loads cleanly.

---

## Step 4 — The Proxy → HTTP history pane (your most-used view)

This is where you live during the CTF.

1. Burp → **Proxy** tab → **HTTP history** sub-tab.
2. Every request your browser made appears as a row.
3. Columns to know: **Method** (GET/POST/PUT/DELETE), **URL**, **Status** (200/404/500/etc.), **Length** (bytes).
4. Click any row → bottom pane shows **Request** (left) + **Response** (right).
5. **You can read everything** the website is doing under the hood.

> **Key habit:** before doing anything in Juice Shop, click the URL once in Firefox, then look at HTTP history to understand what request fired. Half of "I'm stuck" moments resolve by just reading the request you already sent.

---

## Step 5 — Send to Repeater (modifying requests)

Repeater is how you change a request and replay it. This is what solves most challenges.

1. In *HTTP history*, find the request you want to mess with (e.g. the login POST).
2. **Right-click** the row → **Send to Repeater**.
3. Click the **Repeater** tab at the top — you're now in the request editor.
4. Edit the request body or headers (e.g. change a JSON field, add a header).
5. Click the orange **Send** button (top-left of Repeater).
6. The response appears on the right side.

> **Example you'll do tomorrow** (Zero Stars challenge): submit feedback in the UI → in HTTP history find POST `/api/Feedbacks` → Send to Repeater → in the request body change `"rating":1` to `"rating":0` → Send → server accepts → challenge solved.

---

## Step 6 — Anatomy of a request you'll edit

Most Juice Shop challenges look like this in Repeater:

```
POST /api/Feedbacks HTTP/1.1
Host: 127.0.0.1:3000
Authorization: Bearer eyJhbGc...                ← your JWT token
Content-Type: application/json
Cookie: token=eyJhbGc...
Content-Length: 78

{"comment":"hello","rating":3,"captchaId":1,"captcha":"42"}   ← body (JSON)
```

What you'll typically change:
- **Body fields** — the JSON values at the bottom (`rating`, `UserId`, `BasketId`, etc.)
- **Authorization header** — the JWT token (for JWT forging)
- **URL parameters** — things after the `?` in the URL (e.g. `?bid=2`)

To edit: just click in the text area and type. Hit **Send** to replay.

---

## Step 7 — Decoder tab (for tokens + base64)

Tabs at the top → **Decoder**.

- Paste any base64 / URL-encoded / JWT string.
- Click **Decode as → Base64** (or pick the right format).
- The plain text appears below.
- For JWTs specifically: paste the whole token, decode each segment separately (split by dots).

You'll use this for the JWT Forging challenge.

---

## Step 8 — Quick troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Firefox says "the proxy server is refusing connections" | Burp not running OR proxy off in Firefox settings | Start Burp; or switch Firefox proxy off |
| HTTPS sites show certificate error | CA cert not imported | Re-do Step 3 |
| HTTP history is empty | Firefox proxy not pointing to 127.0.0.1:8080 | Re-check Step 2 |
| Repeater shows no response | You edited something invalid (e.g. broken JSON) | Check the request body for missing quotes/commas |
| Burp is slow / laggy | Too many requests in history | *Proxy → HTTP history → right-click → Clear history* |

---

## The 6 Burp moves you'll use tomorrow (memorise)

> 🧠 **MEMORIZE WITH TEAMMATE — these 6 moves.** Quiz each other before bed: Member A says a number, Member B describes the move. Then swap.

1. **F12 in Firefox** to find the URL/endpoint a button calls
2. **Right-click in HTTP history → Send to Repeater** to copy a request
3. **Edit the request body** in Repeater (JSON fields)
4. **Edit the Authorization header** in Repeater (for JWT forging)
5. **Click Send** to replay the modified request
6. **Read the response** to confirm the attack worked

That's it. Six moves cover ~90% of Juice Shop.

---

## What if Burp won't start at all?

Quick fallback: use **OWASP ZAP** (already on Kali). Same idea, different UI:
```bash
zaproxy &
```
- ZAP's "Quick Start → Manual Explore" replaces Burp's HTTP history.
- ZAP's "Sites tree → right-click → Open in Requestor" replaces Burp's Repeater.

Both tools do the same job. If one breaks, switch to the other.

---

## Final tip

**Don't try to learn every Burp feature.** You only need:
- Proxy → HTTP history (see traffic)
- Repeater (modify + replay)
- Decoder (decode tokens)

Ignore Intruder, Scanner, Sequencer, Comparer, etc. for tomorrow. Save them for after the competition.
