# Break-glass `admin` account — vault and rotation procedure

This document is the operational runbook for the local Django `admin`
superuser on the ColdFront portal. After the passwordless rollout
(`auth-passwordless-model`), this account is the **only** password-backed
entry point left in the stack, so its credential handling deserves a
dedicated procedure.

See also:
- `keycloak/ldap-providers/README.md` — passwordless rollout (`editMode`,
  ROPC rollback note)
- project memory `auth-passwordless-model` — full rollout design

## Why a vault matters for this user

The `admin` password is the **only** password-based credential left in
the stack. If it leaks:

- `/admin/` is exposed to the internet (no IP allowlist)
- No second factor on the Django admin login form
- An attacker with the password gets full Django admin → full ColdFront
  control

So the password **is** the security control. Where it lives matters.

## What NOT to do

- Plain-text `.env` on the server (anyone with SSH can `cat` it)
- Slack / email / Confluence "where we kept it"
- Sticky note on the monitor (yes, this happens)
- Git (even a private branch — irrecoverable history)
- "I remember it" (single-point-of-failure on one person)

## Vault options

| Tool | Notes |
|------|-------|
| **1Password Business** | Shared vault, audit log, per-item sharing, password expiration. JHU may have an institutional license. |
| **Bitwarden Enterprise** | Open-source, can self-host inside JHU infra. Organizations + Collections + audit log. |
| **HashiCorp Vault** | More robust, with APIs. Overkill for a single account — pays off only if you have other secrets to centralise. |
| **LastPass Enterprise** | Common in academic environments. Audit logs weaker than 1Password / Bitwarden but functional. |
| **JHU IT secret store** | Worth asking — if JHU has an institutional offering, align the stack with it. |

For a tiny admin pool (2–3 people), 1Password or Bitwarden with a single
shared item is enough.

## Initial setup

1. Generate a strong password:
   ```bash
   openssl rand -base64 32
   # example: bU+xn4q6jK8m7Y3pL2fH9wRtVcZ1aE0sN5oI4dQ
   ```
2. Apply it inside the ColdFront container:
   ```bash
   docker exec -it coldfront python manage.py changepassword admin
   # paste the generated password
   ```
3. Create a vault entry with:
   - **Item name**: `ARCH ColdFront break-glass admin (prod)`
   - **Username**: `admin`
   - **Password**: the generated string
   - **URL**: `https://arch.jhu.edu/admin/`
   - **Notes**: creation date + rotation procedure pointer + usage policy
   - **Tags**: `break-glass`, `arch`, `quarterly-rotation`
4. Share the **item** (not the whole vault) with every authorised
   admin — typically 2–3 people. Avoid sharing with the broader team
   even if they appear in `ARCH_SUPERUSERS`; JHED admins should use
   Entra MFA via the portal for routine ops.
5. Test the `/admin/` login with the new password before closing.

## Quarterly rotation (~ every 90 days)

1. Recurring calendar event (Google Calendar / Outlook) every 3 months.
2. On the day: generate a new password (`openssl rand`), apply via
   `manage.py changepassword admin`, update the vault entry, test the
   login.
3. Log the rotation somewhere persistent — date + who ran it.
4. Notify the other admin(s) sharing the vault item.

## Using the password (real break-glass)

1. Keycloak is down, Entra is down, no OIDC path works.
2. Retrieve the password from the vault — this leaves an entry in the
   vault audit log (who accessed, when).
3. Log in at `/admin/`.
4. Perform the emergency operation.
5. Log out.
6. **Rotate immediately**: any in-anger use is treated as a potential
   compromise by the team — rotate before closing the incident.

## Why quarterly

90 days is the industry default for privileged accounts that are rarely
used (NIST SP 800-63B suggests 60–90 days for this profile):

- **Too short (< 30 days)**: rotation friction, the team stops respecting
  the cadence.
- **Too long (> 180 days)**: if the credential leaked at any point, an
  attacker has a long window to harvest.
- **90 days**: sweet spot for a credential that sits idle and is rarely
  retrieved.

Scheduling options:

- Recurring Google Calendar event every 3 months for everyone in the
  admin pool.
- Vault-native expiration where supported (1Password, Bitwarden have
  "expires in X days" fields).
- A cron job on the server that sends a reminder email (more formal but
  overkill for a single account).

## Minimal next-step plan

If choosing a vault is not done yet:

1. Create the item in whatever the admin pool already shares
   (1Password personal tier with sharing, Bitwarden organisation, etc.).
2. Add a recurring 90-day event in the shared calendar.
3. Capture the rotation steps as a Claude skill (e.g.
   `/rotate-admin-password`) or as a script under `scripts/`.
4. When JHU IT formalises an institutional secret manager, migrate the
   item over.

The important part is **not** keeping the password in plain-text
`.env`. Strong password + auditable storage + at least two people with
access covers 90% of the risk.

## Related env vars

- `ARCH_ADMIN_ALERT_EMAILS` (in `start.sh` init) — recipients of the
  `admin_login_alert_task` email that fires on every successful
  `/admin/` login. Configure with the admins who should see those
  notifications.
- `AXES_FAILURE_LIMIT`, `AXES_COOLOFF_HOURS` (in
  `coldfront/custom/config/auth.py`) — lockout policy on top of the
  password.
- `ADMIN_LOGIN_RATE_MAX`, `ADMIN_LOGIN_RATE_WINDOW` (in
  `coldfront/custom/core/user/middleware.py`) — per-IP throttle for the
  login URL.
