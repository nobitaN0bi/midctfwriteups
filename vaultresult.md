# ScalerBank CTF — Challenge Results Vault

| #  | Challenge            | Vulnerability              | Flag                                | 
|----|----------------------|----------------------------|-------------------------------------|
| 1  | Vault Access         | SQL Injection (Auth Bypass)| `FLAG{vault_66ddc28c}`              | 
| 2  | Search & Steal       | Reflected XSS              | `FLAG{reflect_92c06356}`            | 
| 3  | Unsigned Access      | JWT `alg: none` Bypass     | `FLAG{jwt_1ff94e06}`                |  
| 4  | Import Statement     | XXE (XML External Entity)  | `FLAG{xxe_6308af43}`                | 
| 5  | Statement Mailer     | OS Command Injection       | `FLAG{cmdi_17fd3354}`               |  
| 6  | Email Hijack         | CSRF                       | `FLAG{csrf_1c0c47d4}`               |  
| 7  | Welcome Card         | SSTI (Jinja2)              | `FLAG{ssti_1e29a80d}`               | 
| 8  | Self Service         | Password Reset Logic Flaw  | `FLAG{reset_c40ce1a6}`              | 
| 9  | Account Nickname     | DOM-Based XSS              | `FLAG{domxss_fa903fe3}`             | 
| 10 | Balance Check        | Blind SQL Injection        | `FLAG{blind_7141c2af}`              | 

| # | Challenge | Key Insight |
|---|-----------|-------------|
| 1 | Vault Access | SQL comment `--` in username field bypasses password check |
| 2 | Search & Steal | Reflected XSS via `\| safe` filter; submit to bot for admin cookie steal |
| 3 | Unsigned Access | Custom JWT decoder accepts `alg: none` — forge token with admin role |
| 4 | Import Statement | lxml parser with `resolve_entities=True`; `&xxe;` entity reads `/app/data/flag_xxe.txt` |
| 5 | Statement Mailer | `shell=True` in subprocess; semicolon injects commands |
| 6 | Email Hijack | CSRF token in form is hardcoded/never validated server-side |
| 7 | Welcome Card | `render_template_string()` concatenates user input; Jinja2 SSTI |
| 8 | Self Service | Reset token checked for validity but NOT bound to username |
| 9 | Account Nickname | `location.hash` written to `innerHTML` without sanitization |
| 10 | Balance Check | Boolean-based blind SQLi; Active/Inactive binary response |

## Configuration Used
- **STUDENT_ID:** `23bcs10019` (uppercased to `23BCS10019` in code)
- **SALT:** `XAkhCcTj64_7t-XonRvOuQq-3CZFPDfG0-yj8SUAX6k` (baked into Docker images)
- **Admin Password:** `B4nkAdm1n2026` (restored after C8 exploit; was changed to `hacked123`)
- **App URL:** `http://localhost:3000`

## Summary
**9/10 challenges solved.** C6 (CSRF attack via bot) requires hosting an external HTML page with auto-submitting form, reachable from inside the Docker network.
