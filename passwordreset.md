The two phases
Password reset has two separate requests, often minutes apart, connected only by a token that travels through the user's email inbox.

Phase 1 — "I forgot my password" (request a link)
Example: Sarah (sarah@acme.com) can't log in.


1. Sarah → /forgot-password, types "sarah@acme.com", clicks "Send reset link"
         │
2. Frontend server action → POST /api/password-reset/request
         │   body: { email: "sarah@acme.com" }
         │   header: X-Client-Host: localhost:3000   (so backend knows the URL to build)
         ▼
3. Backend requestReset():
     a. Look up user by email
     b. Invalidate any OLD unused tokens for Sarah (set used_at = now)
     c. Generate raw token:    a1b2c3...  (32 random bytes → 64 hex chars)
     d. Hash it:               sha256(a1b2c3...) = 9f8e7d...
     e. Store ONLY the hash in DB:
id	user_id	token_hash	expires_at	used_at
1	42 (Sarah)	9f8e7d...	now + 1 hour	NULL

     f. Build link with the RAW token (not the hash):
        https://localhost:3000/reset-password?token=a1b2c3...
     g. Email that link to sarah@acme.com via SES
         │
4. Backend ALWAYS responds 200 { ok: true }  ← even if email doesn't exist
Key idea: the raw token only ever exists in the email. The database holds only its hash. So even someone who steals a DB dump can't reverse 9f8e7d... back into a working link.

Why always return 200? If "no account found" gave a different response, an attacker could probe emails to discover who has accounts (account enumeration). Same response either way = no leak.

Phase 2 — "Here's my new password" (use the link)
Example: Sarah opens her email, clicks the link.


5. Browser → /reset-password?token=a1b2c3...
         │   Page reads the token from the URL into a hidden form field
         ▼
6. Sarah types new password "MyNewPass99" twice, clicks "Reset password"
         │   (frontend checks: ≥6 chars, both fields match)
         ▼
7. Frontend server action → POST /api/password-reset/confirm
         │   body: { token: "a1b2c3...", password: "MyNewPass99" }
         ▼
8. Backend resetPassword():
     a. Hash the incoming raw token:  sha256("a1b2c3...") = 9f8e7d...
     b. Find the DB row WHERE token_hash = "9f8e7d..."
     c. Validate, in order:
          - row exists?         no  → reject "invalid_token"
          - used_at is NULL?    no  → reject (already used)
          - expires_at > now?   no  → reject (expired, >1h old)
     d. bcrypt-hash the new password (cost 12)
     e. In ONE transaction:
          - UPDATE dashboard_users SET password = <bcrypt hash> WHERE id = 42
          - UPDATE password_reset_token SET used_at = now() WHERE id = 1
         │
9. Backend → 200 { ok: true }
         ▼
10. Frontend redirects → /login?reset=1
        "Password updated. Sign in with your new password."
Now Sarah logs in normally — LoginService does bcrypt.compare("MyNewPass99", storedHash) → match.

Why each safety rule exists
Rule	What it prevents
Store hash, not raw token	DB leak can't be turned into working reset links
used_at (single-use)	A leaked/old email link can't be replayed twice
expires_at (1 hour)	An old forwarded email link stops working
Invalidate old tokens on new request	If Sarah clicks "forgot" 3 times, only the newest link works
Uniform 200 response	Attacker can't discover which emails have accounts
bcrypt cost 12	Stored passwords stay slow-to-crack (same as registration)
Two token representations — the crux
This is the one thing people find confusing, so to be explicit:


RAW token  "a1b2c3..."   →  lives in: the email link, the URL, the confirm request
HASH       "9f8e7d..."   →  lives in: the database only
The backend never stores the raw token and never logs it. On confirm, it hashes whatever the user brought back and looks for a matching hash. It's the same trick as storing password hashes instead of passwords — the token is essentially a one-time password delivered by email.

A few failure examples
Link clicked after 2 hours → expires_at < now → "invalid or expired" → request a new one.
Same link clicked twice → first sets used_at; second sees used_at != NULL → rejected.
Someone guesses a URL → their fake token hashes to something not in the table → rejected.
Email doesn't exist (typo) → Phase 1 silently does nothing but still returns 200, so no email is sent and nothing leaks.
Want me to also add a diagram comment block into the service file, or is this explanation enough?

