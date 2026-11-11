# MD2PDF: SSRF via HTML Injection in a Markdown-to-PDF Converter

**Category:** Web / SSRF
**Target services:** Port 80 (main site), Port 5000 (internal admin panel)
**Flag location:** `/admin` on localhost:5000
**Result:** Flag retrieved by making the server fetch its own admin page

---

## Summary

The MD2PDF converter advertised a secure markdown-to-PDF service. It rendered markdown headings and bold text, but it also passed raw HTML through to the PDF renderer without sanitizing it. The admin panel on port 5000 was restricted to localhost. Because the renderer runs on the server, an `<iframe>` pointing at `localhost:5000/admin` made the server fetch the page on our behalf, and the resulting PDF showed the flag.

---

## Reconnaissance

### Port scan

```bash
nmap -sV -p- <TARGET_IP>
```

Result: ports **80** and **5000** are open.

### Directory brute force

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

Result: `/admin` found.

### Accessing the admin page directly

Browsing to `http://<TARGET_IP>/admin` returned a message stating the page is only accessible from localhost. The restriction is enforced by the server based on the client's source address, so direct access from our machine is blocked.

---

## Identifying the Attack Surface

The converter accepted markdown input. Testing with basic syntax showed:

- `# Heading` and `**bold**` were rendered.
- Output was produced as a PDF.
- No backend behavior beyond conversion was visible.

Markdown converters often pass raw HTML through unless the input is sanitized. If the converter renders HTML, and the HTML is processed by a headless browser or a PDF engine running on the server, that engine makes its own network requests. Those requests come from the server, so they can reach services that only trust localhost.

This is **Server-Side Request Forgery (SSRF)**: the attacker causes the server to make requests to targets the attacker cannot reach directly.

---

## Exploitation

Submitted input:

```html
<iframe src="http://localhost:5000/admin"></iframe>
```

| Part | Purpose |
|------|---------|
| `<iframe>` | Tells the HTML renderer to load another page and embed it. |
| `src="http://localhost:5000/admin"` | Points at the restricted admin panel. `localhost` resolves to the server itself, so the request passes the localhost check. |

The converter's renderer requested `/admin` from inside the server, received the page, and embedded it in the generated PDF. Downloading the PDF showed the admin content, including the flag.

---

## Why It Worked

1. **Raw HTML was accepted.** The converter didn't strip or escape HTML tags.
2. **The renderer makes outbound requests.** Loading an `<iframe>` requires the server to fetch the URL.
3. **The access control trusted the source address.** The admin panel accepted any request coming from localhost, and the renderer's request came from localhost.
4. **No network-level restriction existed** on which internal URLs the renderer could reach.

---

## Mitigation

- **Sanitize or disable raw HTML** in markdown input. Use an allowlist of safe tags, or render markdown only.
- **Block outbound requests from the renderer** to loopback, private, and link-local address ranges (`127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`).
- **Run the renderer with restricted networking,** for example in a container or sandbox with no access to internal services, or with `--disable-local-file-access` style flags where available.
- **Don't rely on source-IP checks for admin access.** Require authentication on admin endpoints, because any server-side request can satisfy a localhost check.

---

## Key Takeaways

- A "secure" label on a tool doesn't mean its features are safe. Any feature that fetches content on the server needs review.
- Localhost-only restrictions don't protect against SSRF. The server is always "localhost" to itself.
- Testing with simple markup first, then trying HTML, is a fast way to find out what a renderer does with input.
