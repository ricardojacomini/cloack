# Superuser — ColdFront / Django admin (cluster ops + identity)

## Who you are + scope

You have `is_superuser=True` on your Django `User`. You typically also
belong to the LDAP `cn=admin` group (cluster sudoers) and the
`helpdesk-admin` group (helpdesk privileges). You approve / deny
allocations, drain or resume Slurm nodes, reset stale OTP credentials,
rotate the local `/admin/` break-glass password, and read the qcluster
audit log for every async task.

You typically also operate the cluster (deploy via `start.sh`, manage
Ansible bundles) — for that side read [`admin.md`](admin.md). For
helpdesk-side work see [`staff.md`](staff.md). You authenticate via
Entra Federation if you are JHED; if you are an `ext-*` user you use
TOTP via the Keycloak account console (same as every other regular
[`user.md`](user.md)).

## Quick start

### 1. Approve / deny an allocation request

```
/admin/allocation/allocation/?status__name=New
  → open the allocation
  → review IO, storage, backup, Weekly Cap
  → action "Approve" or "Deny" (deny requires a reason)
```

Approval triggers `on_allocation_activated`, which provisions the
Slurm account hierarchy via REST in real time. ToS gate still applies:
PI association is created only if they have accepted the current ToS.

### 2. Drain / resume a Slurm node

```
/admin/arch_sync/slurmnode/
  → select node → action "Drain selected nodes" / "Resume"
  → reason is logged on the SlurmNode row
```

Behind the scenes this calls `slurmrestd` (or `scontrol` as fallback)
on the right cluster. The action is also surfaced on the live
`/cluster/<id>/` page for staff with the cluster in their scope.

### 3. Reset a stale OTP credential

```
docker exec keycloak bash -c '/opt/keycloak/bin/kcadm.sh ...'
```

See the recipe in [`keycloak/README.md`](../../keycloak/README.md)
"Clear a stale OTP credential" — it requires:

- `kcadm config credentials` once per session (writes
  `/tmp/kcadm.config`).
- `kcadm get users -q username=$USER` to fetch the user id.
- `kcadm delete users/$UID/credentials/<cred-id>` to drop the OTP row.

Alternative — force the user to re-enrol on their next login:

```bash
kcadm update users/$UID --config /tmp/kcadm.config -r jhu \
  -s 'requiredActions=["CONFIGURE_TOTP"]'
```

### 4. Rotate the break-glass `/admin/` password

```bash
bash scripts/rotate-admin-password.sh
```

Generates a 32-char password, applies it via `manage.py changepassword`
inside the coldfront container, appends an entry to the gitignored
audit log, and prints the password once for paste into the team vault.
See [`docs/break-glass-admin.md`](../break-glass-admin.md) for the
full runbook (vault location, quarterly rotation cadence,
`admin_login_alert_task` alert wiring).

### 5. Read the qcluster audit trail

```
/admin/django_q/task/
/admin/?app=admin&model=logentry   → Home → Administration → Log entries
```

Every `async_task()` call uses the `task_completion_hook` so the
admin LogEntry table records when each scheduled task ran, success or
failure, duration, and result snippet.

## Reference matrices

### Scheduled tasks (django-q)

| Task | Schedule | Purpose |
|---|---|---|
| `sync-ldap-daily` | 02:00 daily | Full LDAP sync (PIs, members, groups, home dirs). |
| `sync-slurm` | every 15 min | Safety net for the signal-based real-time provisioning. |
| `collect-sacct-periodic` | every 15 min | Pull completed jobs from sacct into `SlurmJob`. |
| `check-core-hours-reset` | 01:00 daily | Trigger scheduled period resets on allocations. |
| `cleanup-task-queue` | hourly :30 | Purge stuck / old qcluster tasks. |
| `sync-totp-status` | every 30 min | Mirror Keycloak TOTP enrolment state into `UserProfile.totp_enrolled`. |
| `refresh-slurm-jwt` | every 6 h (:17) | Re-issue the slurmrestd JWT to `/etc/slurm/slurm_jwt.token`. |
| `sync-bare-metal-topology` | every 10 min | `import_slurm_cluster --update` per bare-metal cluster — Service Health reflects newly-added nodes within 10 min. |

### LDAP privilege groups (per affiliation tree)

| Group | UID/GID | What it grants |
|---|---|---|
| `cn=admin,ou=Groups,ou=jhu` | reserved | Cluster sudoers; `sudo` on every Slurm node. Also auto-imports as Django `is_staff=True` via `seed_initial_data.sync_admin_group_to_is_staff`. |
| `cn=helpdesk-admin,ou=Groups,ou=jhu` | per affiliation | Helpdesk admin (manage queues, change ticket status, edit categories). |
| `cn=helpdesk-staff,ou=Groups,ou=jhu` | per affiliation | Helpdesk staff (work tickets, classify, reply). |
| Cluster (LDAP admin) Django Group | n/a | Mirrors `cn=admin` membership for the Django side. Manage at `/admin/auth/group/`. |
| `ARCH_SUPERUSERS` env var | n/a | Comma-separated usernames auto-promoted to `is_superuser=True` when they land in `cn=admin`. |

### Cluster Scope + Affiliation Scope (delegated staff admin)

| Scope | M2M target | Controls |
|---|---|---|
| Cluster Scope | `Resource` (type Cluster) | Drain / resume nodes, view cluster detail, regenerate the per-cluster Ansible bundle |
| Affiliation Scope | `Affiliation` | Visibility into users, PIs, projects, allocations of those affiliations only (Manage Users page, Promote PI, scoped list views) |
| `elevate_to_superuser` | bool | Lets a non-superuser staff approve / deny allocations within their scope |

### Common Django Admin actions

| Path | Action | What it does |
|---|---|---|
| `/admin/allocation/allocation/` | Approve / Deny | Approves an allocation, fires real-time Slurm provisioning |
| `/admin/arch_sync/slurmnode/` | Drain / Resume | Slurm node state via REST |
| `/admin/arch_sync/slurmcluster/` | Action "Action import from slurmrestd" / "Update from slurmrestd" / "Check drift" | Mirror Slurm topology into ColdFront DB |
| `/admin/user/userprofile/` | Edit | Set `is_pi`, `is_superuser`, `elevate_to_superuser`, `can_view_billing_stats` |
| `/admin/auth/group/` → "Cluster (LDAP admin)" | Add member | Publishes to LDAP `cn=admin` via the `m2m_changed` signal |
| `/admin/django_q/task/` | View | Live scheduled task table |
| `/admin/?app=admin&model=logentry` | View | Audit trail (all qcluster + admin actions) |

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Allocation approved but Slurm account not created | ToS not accepted by the PI, OR `on_allocation_activated` raised silently | Check the qcluster Log entries; check `TOSAcceptance` rows; manual fallback: `docker exec coldfront python manage.py sync_slurm` |
| `/admin/` login alert email not received | `ARCH_ADMIN_ALERT_EMAILS` not set in `.env`, OR recipient flagged the SMTP | Set the env var; check `admin_login_alert_task` LogEntry for the dispatch result |
| `cn=admin` membership change not visible on login node | SSSD cache; `sss_cache -E` runs at end of every `run_ldap_sync` but TTL = 60 s | Wait one minute, or run `sss_cache -E` manually on the login node |
| New user created but no UserProfile | The Postgres trigger `tr_auth_user_after_insert_fn` is missing or out-of-date (e.g. `totp_enrolled` column added after trigger) | Re-run `coldfront/init/ldap_sync_postgres_run_it_once.py`; see iter-4 fix `1bffe47` |
| Scheduled task missing from qcluster | django-q scheduler did not pick up the new schedule (entry added but qcluster not restarted) | `docker restart qcluster` |
| OTP re-enrol flow does not prompt | User does not have `CONFIGURE_TOTP` in their `requiredActions` and `defaultAction=true` only applies to new users | Use the `kcadm update users` snippet from Quick start §3 to add the required action manually |
| `helpdesk-admin` LDAP membership changed but Django group did not | The signal `_sync_helpdesk_ldap_groups` only fires on `m2m_changed` for `auth.User_groups` — the inverse direction does not auto-sync | Run `coldfront python manage.py sync_ldap` to reconcile on the next pass |
| Allocation in "Pending review" forever | Staff scope does not include the requesting affiliation | Add the affiliation to the staff's `StaffResourceAssignment.affiliations`, OR escalate to a superuser |

## Where to go next

- `/admin/` — Django Admin (Home → Administration → Log entries for
  the qcluster task audit trail).
- [`docs/break-glass-admin.md`](../break-glass-admin.md) — full
  break-glass `/admin/` rotation runbook (vault, audit log, alerts).
- [`keycloak/README.md`](../../keycloak/README.md) — kcadm recipes
  (OTP credentials, theme re-apply, realm settings inspection).
- `/staff/help/` — in-app help with operational notes.
- [`admin.md`](admin.md) — cluster ops side.
- [`README.md`](README.md) — the role index.
