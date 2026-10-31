# PHP2Shell – TryHackMe

**Focus:** Core Interpreter Vulnerabilities (CVE-2026-109618 & CVE-2026-109619), Memory Bailouts, ASLR Bypass, and Pre-Authentication Remote Code Execution

## Introduction

On 21 October 2026, the PHP project disclosed two vulnerabilities that allow an unauthenticated attacker to execute commands on almost any PHP server using ordinary HTTP requests. The flaws live inside the PHP runtime engine itself—below whatever framework or CMS happens to be installed—and affect all releases since PHP 5.4.0 (2012).

Tracked as **CVE-2026-109618** and **CVE-2026-109619** (both scored **CVSS 9.2 Critical**), these flaws affect persistent worker process environments (such as Apache `mod_php`). They were patched in PHP versions **8.5.12**, **8.4.27**, **8.3.36**, and **8.2.35**.

## Technical Background

### 1. The Entry Point: `session.upload_progress`

The attack abuses a default PHP feature designed to track file upload progress by writing metrics directly into `$_SESSION`.

Because PHP parses the request body inside early callbacks before initializing the script environment or executing application code, this mechanism runs **pre-execution**. It requires no `session_start()`, no valid authentication, and no specific application logic—any endpoint accepting `POST` requests is vulnerable.

### 2. The Engine Flaw: Memory Bailouts (`longjmp`) & CWE-824

When a request exceeds PHP's `memory_limit`, the engine executes an emergency `longjmp()` bailout to abort processing.

If this out-of-memory condition strikes after a memory block is freed but before its pointer is cleared, the bailout skips normal cleanup routines. This leaves behind a dangling pointer (**CWE-824: Access of Uninitialized Pointer**). In persistent architectures (like Apache `mod_php`), this dangling pointer remains in worker memory and can be manipulated by subsequent requests.

## Exploitation Chain

### Phase 1: Target Confirmation & Reconnaissance

Checking the container configuration and target endpoint:

```bash
docker ps --format '{{.Names}}  {{.Image}}  {{.Status}}'
# Target: php2shell-target running php:8.4.26-apache (Vulnerable)

curl -s http://127.0.0.1:8088/index.php
# Output: OK
```

The target page consists of a single line: `<?php echo "OK\n"; ?>`. Because the flaw exists within the runtime interpreter, application-layer code quality provides no protection.

### Phase 2: Memory Disclosure & Defeating ASLR (CVE-2026-109618)

To execute code, ASLR must first be defeated. **CVE-2026-109618** turns the dangling pointer into a process memory leak.

1. An out-of-memory bailout is triggered while processing a session ID, leaving a stale pointer.
2. A subsequent request reoccupies the freed memory space, inflating the session ID length descriptor.
3. When PHP returns the `Set-Cookie: PHPSESSID=...` header, it echoes up to ~67 KB of raw worker process memory back to the attacker.

Executing the exploit script leaks the required base addresses:

```bash
python3 exploit.py -u http://127.0.0.1:8088/index.php
```

```text
[*] stage 1: heap leak (attempt 1/3)
[+] chunk base = 0x70d1a2c00000
[*] stage 2: libphp leak (attempt 1/3)
[+] libphp base = 0x70d1a4911000
```

### Phase 3: Remote Code Execution (CVE-2026-109619)

With heap and library base addresses disclosed, **CVE-2026-109619** reoccupies the dangling reference with a forged internal structure (`HashTable`).

When the engine cleans up resources at the end of the request, it calls `pDestructor`. The exploit overwrites `pDestructor` with the leaked address of `system()` in `libphp`, passing attacker-controlled arguments.

#### Executing Proof of Concept:

```text
[*] stage 3: HTDATA=0x70d1a2c690d8 ZIF=0x70d1a4d76370
[+] W3 attempt 1
PROOF: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

#### Flag Retrieval:

```bash
python3 exploit.py -u http://127.0.0.1:8088/index.php -c 'cat /f>/out/p1' --proof-file out/p1
```

```text
PROOF: THM{[REDACTED]}
```

## Detection & Mitigation

### Remediation

* **Runtime Patching:** Upgrade PHP runtime packages to **8.5.12**, **8.4.27**, **8.3.36**, or **8.2.35** and restart all worker processes. Application-level updates (e.g., updating CMS plugins) will not patch core engine flaws.

### Interim Mitigations

* Disable upload progress tracking in `php.ini`:
  ```ini
  session.upload_progress.enabled = 0
  ```
* Disable opcode caching (`opcache.enable = 0`).
* Reject oversized `multipart/form-data` POST requests at the network edge / reverse proxy.

### Detection Indicators

* Exceptionally large `PHPSESSID` values inside `Set-Cookie` response headers.
* Frequent worker process crashes and out-of-memory errors in web server error logs.
* Abnormally high form field counts in incoming `multipart/form-data` requests carrying `PHP_SESSION_UPLOAD_PROGRESS`.

## Takeaway

The critical lesson from PHP2Shell is that security boundaries extend beyond application logic down to the underlying execution runtime. Flaws inside core memory management mechanisms can completely bypass application-layer defenses, WAFs operating as PHP modules, and traditional input sanitization.

## Disclaimer

**This write-up is for educational purposes only. The flag and other sensitive challenge details have been redacted in accordance with TryHackMe's write-up guidelines.**
