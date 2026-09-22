# CLOACK — Staff / Admin CLI & Troubleshooting Guide

Console commands to operate and troubleshoot the CLOACK stack (ColdFront +
OpenLDAP + Keycloak + Slurm + PostgreSQL). Everything here is run from the
**host** where the stack runs (dev laptop or `mgmt0x`), against the running
containers.

> **Golden rules**
> - **Preview before you destroy.** Anything that removes users/accounts/homes
>   has a `--dry-run` (or a UI preview). Always run it first.
> - **Files & DB via commands, never hand-patched.** Use management commands and
>   `start.sh` subcommands — no manual SQL / no editing volumes by hand.
> - **`coldfront <cmd>`, not `manage.py`.** Inside the container the wrapper is
>   `coldfront`.
> - **Runtime module path is `coldfront.plugins.arch_sync`** (the repo's
>   `coldfront/custom/plugins/arch_sync/` is deployed into site-packages at
>   container start) — **not** `coldfront.custom.plugins.arch_sync`.
> - **Never add `-M <cluster>`** to `sacct` / `sreport` / `squeue` / `scontrol`
>   / `sacctmgr` — the cluster is auto-resolved.

---

## 0. Conventions

Run a Django management command inside the ColdFront container:

```bash
docker exec coldfront coldfront <command> [args]
```

Open a Django shell (one-liner):

```bash
docker exec coldfront coldfront shell -c "from coldfront.plugins.arch_sync.models import SlurmCluster as C; print(list(C.objects.values_list('name', flat=True)))"
```

Tail a service log:

```bash
./start.sh log coldfront          # or: keycloak | openldap | qcluster | postgres | helpdesk
docker logs --tail 50 <container>
```

Health snapshot of everything:

```bash
./start.sh doctor                 # 14-check reconcile/health report
bash scripts/verify-deploy.sh     # authoritative post-deploy gate (exit code = red checks)
docker ps --format '{{.Names}}\t{{.Status}}'
```

---

## 1. User & Account Management

### Remove specific users (by name) — **preview first**

Preview (does **not** delete; shows PI-cascade risk):

```bash
docker exec coldfront coldfront delete_users \
  -u ssci-asap7772 ext-nargatoff ext-mwaugh --dry-run
```

Actually delete (four-step teardown: Slurm associations + home dir
`/home/<aff>/<user>` + LDAP entry + ColdFront `User`/`UserProfile`):

```bash
docker exec coldfront coldfront delete_users -u ext-nargatoff ext-mwaugh --yes
```

| Flag | Effect |
|------|--------|
| `-u`, `--username` | One or more usernames (repeatable). |
| `--dry-run` | Preview only, incl. PI-cascade risk. |
| `--yes` | Required to actually delete (unless `--dry-run`). |
| `--keep-home` | Do not remove home directories. |
| `--keep-slurm` | Do not remove Slurm associations. |
| `--force-pi` | Allow deleting a **PI** (cascade-deletes their projects/allocations). Refused otherwise. |

> Refuses system accounts (`admin`, `root`, `coldfront`). Deleting a PI without
> `--force-pi` is blocked because it would cascade-delete their projects and
> allocations. Shared per-PI `/scratch` is never touched.

### Remove an identity that exists only in LDAP / Slurm (no ColdFront `User`)

Legacy migration entries and alias-named duplicates (a `first.last` account next
to the real one) often have no ColdFront `User` behind them, so `delete_users`
answers "No matching users". Use the host-run wrapper, which walks
ColdFront → Slurm → LDAP → Keycloak in the safe order and skips what is absent:

```bash
./scripts/clear_user.sh <user>            # diagnose + DRY-RUN: read every "match" / "would delete" line
./scripts/clear_user.sh <user> --yes      # apply
```

Rules that came out of a real incident:

- **Run the dry-run first and read it — and again after any change to the
  script or the target.** A `--yes` without a reviewed dry-run turns any filter
  bug into identity loss.
- The Keycloak step deletes **username matches only** (`<user>` or `<user>@…`).
  A row that matches only by e-mail is printed as `SKIP`: that is normally the
  person's *real* account carrying the alias as its e-mail. Never delete it
  automatically; if it truly is a zombie, remove it by hand after checking.
- Home directories are left alone on this path (only `delete_users` removes homes).

### Remove a whole course (class) cohort

Every account named `<course>-*` (disposable class accounts). Preview → confirm:

```bash
docker exec coldfront coldfront delete_course_users --course en540 --dry-run
docker exec coldfront coldfront delete_course_users --course en540 --yes
```

Same flags as `delete_users` (`--keep-home`, `--keep-slurm`). The helper refuses
a blank/invalid course code, so a bare `-` can never match the whole table.

### Import users

New users from CSV (`full_name,email,uid`):

```bash
docker exec coldfront coldfront import_users_csv --csv-file /path/in/container/users.csv
# Disposable course accounts (<course>-<uid> + per-course email):
docker exec coldfront coldfront import_users_csv --csv-file users.csv --course en540
```

Or use the UI: **Import Users** page (`/billing/import-users/`) — **New Users**
(CSV) and **Migration** (LDIF slapcat dump) modes.

### Export users (directory → CSV)

On the **Import Users** page, the **Export Users** card streams a CSV with
`full_name,email,uid,is_pi,members,member_of` (filterable by scope and
affiliation). `members` = a PI's LDAP group members; `member_of` = the PI(s) a
user belongs to. Direct URL:

```
/billing/import-users/?export=csv&scope=all&affiliation=schmidt
```

---

## 2. LDAP / Directory

ColdFront is the source of truth; LDAP is pushed from it.

```bash
# Reconcile ColdFront → LDAP (users, homes, PI groups, admin/helpdesk groups)
docker exec coldfront coldfront sync_ldap

# Repair drifted/duplicate GIDs
docker exec coldfront coldfront repair_ldap_gids

# Full LDAP reinit from ColdFront (CLI) — reimports LDIF, forces sync
./start.sh ldap-reinit dev

# Inspect LDAP directly
docker exec openldap slapcat -n 1 | less           # dump the data DB (LDIF)
# phpLDAPadmin UI is also available as a service
```

Backup LDAP before risky changes:

```bash
docker exec openldap slapcat -n 1 > migration/ldap-$(date +%F).cat
```

> **SSSD cache** on login nodes is flushed at the end of every `sync_ldap`
> (`sss_cache -E`); group changes (`admin`, `helpdesk-admin`) become visible to
> `id`/`sudo` within ~60 s. A shell already open keeps its group list until the
> user re-SSHes.

---

### Groups with two `cn` values (Keycloak `Expected String but attribute 'cn' has more values`)

Legacy migration entries stored the PI's real name as a **second** `cn` on the
group (`cn: ssci-huan` + `cn: Huan Zhang`). Harmless (Keycloak, SSSD and the sync
use the RDN value), but every hourly LDAP sync logs one WARN per group. The sync
never re-adds the value, so a one-off cleanup is final (validated on dev 2026-09-12,
8 groups):

```bash
# 1. Preview — one delete per redundant value, RDN value untouched
docker exec openldap sh -c 'slapcat 2>/dev/null' | awk '
/^dn: /{dn=$0; sub(/^dn: /,"",dn); n=0; grp=(dn ~ /ou=Groups/); rdn=dn; sub(/,.*/,"",rdn); sub(/^cn=/,"",rdn)}
/^cn: /{ if(grp){n++; v[n]=$0; sub(/^cn: /,"",v[n])} }
/^$/{ if(grp && n>1){ for(i=1;i<=n;i++) if(v[i]!=rdn){ print "dn: " dn; print "changetype: modify"; print "delete: cn"; print "cn: " v[i]; print "" } }; n=0; grp=0 }' > /tmp/fix_group_cn.ldif
cat /tmp/fix_group_cn.ldif
# 2. Apply (read the preview first) + verify nothing is left
docker cp /tmp/fix_group_cn.ldif openldap:/tmp/ && docker exec openldap sh -c \
  'ldapmodify -x -H ldap://localhost -D cn=root,dc=arch,dc=cluster -w "$LDAP_ROOT_PASS" -f /tmp/fix_group_cn.ldif; rm -f /tmp/fix_group_cn.ldif'
docker exec openldap sh -c 'slapcat 2>/dev/null' | awk '/^dn: /{dn=$0;n=0;g=(dn ~ /ou=Groups/)} /^cn: /{if(g)n++} /^$/{if(g&&n>1)print dn;n=0;g=0}'   # expect no output
```

## 3. Keycloak

### Status & admin password

```bash
./start.sh keycloak status          # health + master-admin auth probe
./start.sh keycloak reset-admin     # cure master-admin password drift (hard reset, any KC version)
```

### Admin REST from the host

`kcadm.sh` inside the keycloak container fails against `https://localhost:8443`
(`PKIX path building failed`: self-signed cert). Point it at the internal HTTP
listener, or use the curl-based wrapper:

```bash
# shell helper: kcadm over the internal http listener
kc() { docker exec keycloak sh -c '/opt/keycloak/bin/kcadm.sh config credentials --server http://localhost:8080 --realm master --user "$KEYCLOAK_ADMIN" --password "$KEYCLOAK_ADMIN_PASSWORD" >/dev/null && /opt/keycloak/bin/kcadm.sh "$@"' kcadm "$@"; }
kc get realms --fields realm

# or raw admin REST through a throwaway keycloak-config container (curl -k + jq)
./scripts/kcq.sh GET 'admin/realms/<realm>/users?username=<user>&exact=true'
```

### Inspect / repair one user

```bash
kc get users -r <realm> -q username=<user> -q exact=true --fields id,username,email,federationLink,requiredActions
kc get users -r <realm> -q email=<address> --fields id,username,federationLink      # who holds an e-mail
kc get users/<id>/federated-identity -r <realm>          # SSO (Entra) link; empty until the next SSO login
kc get users/<id>/credentials -r <realm> --fields type   # "otp" present = TOTP enrolled
kc update users/<id> -r <realm> -s 'requiredActions=["CONFIGURE_TOTP"]'   # force TOTP enrolment at next browser login
```

`federationLink` set = LDAP-federated user (a mirror of the LDAP entry). Empty =
local user created by the SSO broker at first login (`<jhed>@<domain>`).

### Recover a deleted federated user

The Keycloak row is only a mirror: LDAP, ColdFront, home and Slurm are untouched.
Re-import first, then have the person log in **via SSO** (auto-link by e-mail),
then force TOTP — in that order. An SSO login before the re-import creates a
local duplicate that blocks the LDAP import forever.

```bash
bash keycloak/sync-ldap-federation.sh      # full-sync both realms (the username search above also imports lazily)
kc get users -r <realm> -q username=<user> -q exact=true --fields id,federationLink
# person logs in via SSO → federated-identity lists the IdP again
kc update users/<id> -r <realm> -s 'requiredActions=["CONFIGURE_TOTP"]'
```

Lost with the row: TOTP credential (re-enrol), sessions, Keycloak-local
attributes. If the admin console (master-realm brokering) then refuses the
person, delete their stale shadow user in realm `master` and log in again.

### Brute-force lockout (`error="user_temporarily_disabled"`)

```bash
docker logs keycloak --since 3h 2>&1 | grep '<user>' | grep -oE 'error="[a-z_]+"' | sort | uniq -c
kc get attack-detection/brute-force/users/<id> -r <realm>     # numFailures, numTemporaryLockouts, disabled
kc get realms/<realm> --fields bruteForceProtected,failureFactor,waitIncrementSeconds,maxFailureWaitSeconds,permanentLockout
kc delete attack-detection/brute-force/users/<id> -r <realm>  # clear it (it also expires on its own)
```

Several attempts within the same second mean a client retrying on its own
(VS Code Remote SSH, autossh, a saved password). Ask the person to stop it
before clearing the lockout.

### LDAP full-sync health

```bash
docker logs keycloak --since 8h 2>&1 | grep 'Sync all users finished'              # one line per realm, hourly
docker logs keycloak --since 2h 2>&1 | grep 'Failed during import' | head -3 | cut -c1-400
```

`ModelDuplicateException … email … already exists … Existing user is '<x>@<domain>'`
is the structural case, and it is repairable — see the next subsection.

### Local `<jhed>@<domain>` users blocking the LDAP import (converge to federated)

**Cause.** A JHED person who signs in through Entra BEFORE ColdFront and LDAP
know them gets a LOCAL Keycloak user named after their UPN (`<jhed>@jh.edu`).
ColdFront later creates `User <jhed>` and, at the next `sync_ldap`, an LDAP entry
`uid=<jhed>` with the SAME `mail`. Every hourly import of that entry then
collides on e-mail (`duplicateEmailsAllowed=false`) and fails, permanently. The
count grows by one for every new JHED who logs in before ColdFront knows them.

**Impact.** Portal, helpdesk and JHED Device-Flow SSH are unaffected. What breaks:
Open OnDemand maps `preferred_username` raw, so `<jhed>@jh.edu` 404s; the LDAP
`admin-role` / `helpdesk-role` mappers never evaluate local users, so a
post-migration admin gets neither master-console brokering nor ROPC SSH; and the
hourly error lines bury real Keycloak faults.

**Repair.** Delete the broker-local row once an LDAP entry with an equivalent
e-mail exists, then look the username up again: the local miss falls through to
the LDAP provider and imports the entry as a federated row. `idp-auto-link`
re-binds Entra at the person's next login. Nothing touches ColdFront, LDAP, home
directories or Slurm.

**Precondition, checked once per run.** The repair DELETEs the row that carries
the brokered username, so it is only sound while two realm-level objects hold:
the `entra-jhu` IdP is on the `first-broker-auto-link` flow, and its
`entra-username` mapper mints the BARE JHED
(`${CLAIM.preferred_username | localpart}`). On a UPN-shaped template the
brokered username is `<jhed>@jh.edu`, which no LDAP uid can equal, so after the
DELETE the username probe can never match and Keycloak falls through to
`IdpCreateUserIfUniqueAuthenticator`'s FIRST probe — `getUserByEmail`, on the
person's **mutable** default alias. `idp-auto-link` then binds whatever that
probe found, with no cross-check and no page to stop on. `--apply` refuses with
exit 2 (`RELINK PRECONDITION NOT MET`, nothing deleted); a dry run continues and
prints the refusal, because a census is read-only and is exactly the diagnostic
you need on the host whose realm is wrong. Fix it via the admin API — never by
clearing `/opt/keycloak/data/.config_done`:

```bash
./scripts/kcq.sh GET 'admin/realms/jhu/identity-provider/instances/entra-jhu' \
  | jq -r '[.alias,.firstBrokerLoginFlowAlias,.enabled,.trustEmail]|@tsv'
./scripts/kcq.sh GET 'admin/realms/jhu/identity-provider/instances/entra-jhu/mappers' \
  | jq -r '.[]|select(.name=="entra-username")|[.config.template,.config.syncMode]|@tsv'
```

```bash
docker exec coldfront coldfront converge_keycloak_federation                  # census (dry-run)
docker exec coldfront coldfront converge_keycloak_federation --user nwong21   # one person
docker exec coldfront coldfront converge_keycloak_federation --json > /tmp/census.json
docker exec coldfront coldfront converge_keycloak_federation --apply --limit 100 --batch-size 25
docker exec coldfront coldfront converge_keycloak_federation --apply --full-sync   # last batch
```

`--limit` caps the WORK, not the scan: at most N rows deleted under `--apply`,
or N convertible users reported in a dry run. The scan continues past
already-federated users, so repeated batches make progress rather than
re-walking the same alphabetical prefix. A deleted row whose re-import then
defers still consumes the budget, because the row is already gone.

Dry-run is the default. Realm `jhu` only: the command refuses anything else,
because the Schmidt realm is clean and its duplicate-identity story is a
different shape (one person, two different e-mails, two LDAP entries, violating
no constraint). Exit codes: 0 ran, 1 hard errors, 2 precondition failure.

**Skip reasons.** Every one of these is a gate doing its job, not a fault:

| Reason | Meaning | Override |
|---|---|---|
| `username_not_jhed_pattern`, `system_username` | not a bare JHED handle (`ext-`, `ssci-`, course-wrapped, `admin`/`root`/`coldfront`) | none, by design |
| `no_django_user`, `no_ldap_entry` | nothing to federate onto | none |
| `no_kc_local_user`, `already_federated` | nothing to convert | none |
| `is_service_account` | a client service account | none |
| `kc_disabled`, `django_inactive` | deliberately disabled; a re-import would come back ENABLED | none, deliberately |
| `email_mismatch` | Keycloak e-mail is not the LDAP `mail` | fix the Django e-mail, `sync_ldap`, re-run |
| `ldap_mail_not_unique` | two identities share the address; freeing it could auto-link the wrong person | resolve the duplicate first |
| `alias_held_by_other_kc_user` | some OTHER Keycloak row holds an e-mail alias this person presents — typically a legacy `<first>.<last>`-named row that owns the JHED's current default alias | **none, and no `--force-*` reaches it**: free the address first (ColdFront-first on prod: edit `User.email`, `sync_ldap`, `triggerChangedUsersSync`) |
| `has_sessions` | live session; converting logs them out | `--force-sessions`, named users, off-hours |
| `has_local_credential` | a local password, TOTP or webauthn key would be destroyed | `--allow-otp-loss` for an OTP-only row (they re-enrol); `--allow-credential-loss` destroys passwords too and therefore **requires `--user`** |
| `import_deferred` | the row was deleted but the re-import did not complete; the hourly sync or their next login finishes it | none needed |

**Hourly convergence.** `KEYCLOAK_FEDERATION_CONVERGE_ENABLED=True` (host-local
`${BASE_ETC}/coldfront/coldfront.env`, never the tracked template) arms a task
that converges up to `KEYCLOAK_FEDERATION_CONVERGE_MAX_PER_RUN` (default 50) per
hour. The task is always scheduled and returns `SKIP` while the flag is off, so
arming it needs only the env flip and a qcluster recreate — `env_file` is read at
container CREATE, not at restart.

**Converting an admin.** A `cn=admin` member with an enrolled TOTP loses it
(`--allow-credential-loss`). Confirm their pubkey break-glass first, convert,
have them log into the portal to auto-link, then re-plant enrolment:

```bash
./scripts/kcq.sh PUT "admin/realms/jhu/users/<new-id>" '{"requiredActions":["CONFIGURE_TOTP"]}'
```

ROPC SSH stays refused for them until that enrolment completes.

### Repoint the public host (`ARCH_PUBLIC_HOST` → OIDC/Keycloak URLs)

`ARCH_PUBLIC_HOST` in `.env` drives every browser-facing OIDC URL
(`KEYCLOAK_HOSTNAME`, `*_REDIRECT_URI`, `KEYCLOAK_ACCOUNT_BASE`, ColdFront's
`OIDC_OP_AUTHORIZATION_ENDPOINT`). A plain `detect_public_host.py` **never**
overwrites an existing value — operator-override wins, so it prints `[kept]` and
a stale/internal FQDN (e.g. one `socket.getfqdn()` baked in once) sticks. Force
a clean re-derivation with the wrapper:

```bash
./scripts/fix_public_host.sh mgmt02.mgmt.ai.cluster --dry-run   # review
./scripts/fix_public_host.sh mgmt02.mgmt.ai.cluster             # apply to files
# then, when ready: ./start.sh init prod && ./start.sh start prod  (reconcile)
```

It backs up `.env`/`coldfront.env`/`helpdesk.env`, `--force`-re-derives all OIDC
vars + themes, and scans for leftover old-host references. Editing the files does
**not** touch the running Keycloak (realm `frontendUrl` + client redirectUris) —
the reconcile (`init`+`start`, or the script's `--apply <mode>`) re-applies it.

### Clear cache / pick up theme & realm changes

Theme cache is **off** in this stack (`KC_SPI_THEME_CACHE_*=false`), so staged
theme edits are picked up on restart:

```bash
# Restage themes from the repo → Keycloak bind mount, then restart
./start.sh doctor                   # reconcile restages themes (idempotent)
docker restart keycloak
```

Re-apply **realm config** (login theme, `registrationAllowed`, OIDC redirect
URIs) — clears the one-shot marker and re-runs the config container:

```bash
docker exec -u 0 keycloak rm -f /opt/keycloak/data/.config_done
docker compose -f docker-compose-dev.yml run --rm keycloak-config   # dev
# (prod: use the prod compose file; `./start.sh update` does this at step 6.5)
```

### Dev-only: unpoison HSTS (portal unreachable after visiting KC over https)

```
# In the browser: visit https://<host>:40443/ once (gets max-age=0)
# or chrome://net-internals/#hsts → delete the host entry
```

---

## 4. TOTP / Passwordless Auth

The TOTP-enrolment banner shows for Schmidt + `ext-*` users while
`UserProfile.totp_enrolled = False`.

```bash
# Refresh totp_enrolled from Keycloak for all in-scope users (clears the banner
# for anyone who has actually enrolled; forces enrolment for those who haven't)
docker exec coldfront coldfront sync_totp_status

# Single user
docker exec coldfront coldfront sync_totp_status --username ssci-huan

# Inspect only — do not persist changes
docker exec coldfront coldfront sync_totp_status --dry-run
```

> Runs on a 30-min schedule too. If a user's banner won't clear after they
> enrolled, run the single-user form to poll Keycloak immediately.

---

## 5. Slurm

### Cluster lifecycle (Docker-managed, dev)

```bash
./start.sh slurm status             # nodes + job queue across clusters
./start.sh slurm create [name]      # generate overlay + DB record + register Resource
./start.sh slurm start              # start cluster (auto-builds images if missing)
./start.sh slurm restart            # slurmdbd → slurmctld → nodes, correct order
./start.sh slurm stop
./start.sh slurm destroy            # stop + remove containers/volumes/overlay
./start.sh slurm test-jobs          # seed jobs per user×account×partition, then collect_sacct
```

### Sync & accounting

```bash
docker exec coldfront coldfront sync_slurm              # ColdFront → Slurm (accounts/QOS/caps)
docker exec coldfront coldfront collect_sacct           # import completed jobs (REST default)
docker exec coldfront coldfront collect_sacct --clear   # wipe + re-import
docker exec coldfront coldfront collect_sacct --current-week
docker exec coldfront coldfront export_qos_config --cluster <name>   # regen qos_config.lua
```

### Users blocked by `MaxJobs` / `GrpJobs = 0`

The sync deactivates a user removed from an allocation with `MaxJobs=0` on the
user association, and clears `MaxJobs` **and** `GrpJobs` when they are active
again. Inventory of explicitly blocked associations (`WOPLimits` shows the stored
value instead of the parent's):

```bash
sacctmgr -nP show assoc format=Cluster,Account,User,MaxJobs,GrpJobs WOPLimits | awk -F'|' '$3!="" && ($4=="0"||$5=="0")'
squeue -h -t PD -O JobID,UserName,Account,Reason | grep -E 'AssocGrpJobsLimit|AssocMaxJobsLimit'
```

Every row must be a user who is *not* active on that allocation in ColdFront. An
active user still listed means the sync could not clear it:
`docker logs qcluster 2>&1 | grep reactivated` shows the attempt every 15 min.

### Pending jobs stuck in `Reason=InvalidQOS` after a QOS pin or retirement

Changing an account's QOS (a tier pin via the `QoS`/`DefaultQoS` attributes, or
retiring a QOS) does not touch jobs already queued: they keep the QOS they were
submitted with and stall. Migrate them, **keeping the time limit in the same
command** (`scontrol update … qos=` alone resets it to the partition maximum):

```bash
squeue -h -t PD -O JobID:12,UserName:14,Account:22,qos:8,Reason:14,TimeLimit | awk '$5=="InvalidQOS"'
NEWQ=$(sacctmgr -nP show assoc where account=<acct> user=<user> format=DefaultQOS)
squeue -h -t PD -A <acct> -u <user> -O JobID:12,qos:8,TimeLimit | awk '$2=="<old_qos>"{print $1, $3}' | \
  while read j t; do scontrol update job $j qos=$NEWQ TimeLimit=$t; done
# who changed the association, and when
sacctmgr -P show transactions Accounts=<acct> Start=YYYY-MM-DD format=TimeStamp,Action,Actor,Where,Info
```

### Node drain / resume, reload config

```bash
docker exec <cluster>-slurmctld sinfo
docker exec <cluster>-slurmctld scontrol update NodeName=cn-01 State=DRAIN Reason="maint"
docker exec <cluster>-slurmctld scontrol update NodeName=cn-01 State=RESUME
docker exec <cluster>-slurmctld scontrol reconfigure     # reload slurm.conf
```

### Regenerate a cluster's `slurm.conf` from the DB

```bash
docker exec coldfront coldfront shell -c "from coldfront.plugins.arch_sync.cluster_manager import write_slurm_conf; from coldfront.plugins.arch_sync.models import SlurmCluster as C; write_slurm_conf(C.objects.get(name='skipjack'))"
docker restart skipjack-slurmctld
```

### `/etc/slurm` is a directory bind — edit in place, no `.nfs`

The Slurm service containers bind the whole per-cluster dir
`${BASE_DATA}/slurm/<cluster>` **as** `/etc/slurm` (a **directory** bind, not
per-file). This is inode-proof: rewriting any file **in place** — `vim` (backup
`~`), **`git`** (`pull`/`checkout`/`reset`), `sed -i`, or the DB writer — is
picked up live. `scontrol reconfigure` re-reads the fresh file, and the old
single-file-bind failure mode (a stranded inode kept alive as a `.nfsXXXX`,
`Device or resource busy`, controller stuck on stale config) is gone.

```bash
# Safe: edit the mirror in place, then reload — no recreate, no .nfs
vim ${BASE_DATA}/slurm/skipjack/slurm.conf
docker exec skipjack-slurmctld scontrol reconfigure
```

Because a directory bind **masks** image-baked files, the dir must be a
**complete** `/etc/slurm`. `verify-deploy.sh` check 14 asserts this per role
(controller vs slurmd). Re-stage the static payload (statics + `prolog.sh`/
`epilog.sh` + `cli_filter.lua` + seeds for `qos_config.lua`/`rates.lua`) any time:

```bash
./start.sh stage-slurm-cluster-conf <cluster>   # root on the host; safe on skipjack
```

`qos_config.lua` (billing toggles) and `rates.lua` (rate catalog) are **exported
per-cluster** into `${BASE_DATA}/slurm/<cluster>/` from the DB and are only
**seeded** (not overwritten) by staging — never hand-edit them; change the toggle
/ `BillingRate` in the UI and the signal re-exports + reconfigures.

#### Stage 4 rollout (skipjack, prod live) — one-time recreate

Converting a live cluster from the old per-file binds to the directory bind is a
one-time **CONTAINER RECREATE**, not a restart (the mount shape changes). Do it in
a maintenance window. The Slurm images are **unchanged** — only the `coldfront`
image (which carries the overlay generator) and the overlay itself change.

Two commands are easy to confuse — one is required, the other is forbidden:

| Command | Stage 4? | Why |
|---------|:---:|-----|
| `./start.sh start --from-ghcr prod` | ✅ | Refreshes the **main stack** — pulls the new `coldfront:prod` that carries the dir-bind generator (prod bakes custom code into the image; `docker restart coldfront` does **not** pick it up). Does **not** touch the `skipjack-slurm` project. |
| `./start.sh slurm start --from-ghcr prod` | ❌ | The live-rerun trap — regenerates/rebrings the cluster and has clobbered accounting (`DbdHost`) + interactive (`SrunPortRange=0-0`). Never run it on a live host. |
| `docker compose --project-name skipjack-slurm … up -d --force-recreate skipjack-slurmctld slurmrestd` | ✅ | Applies the dir-bind with the **current** `:prod` images (no re-pull, no full `slurm start`). |

Pre-req: Stage 2 published `:dev`+`:sha` and Stage 3 `promote`d → `coldfront:prod`
already carries the change.

```bash
ssh mgmt02 && cd /opt/mprov/cloack && git pull origin dev

# 1. Main stack: pull the new coldfront:prod (dir-bind generator is baked in).
#    Safe — normal main-stack refresh; does NOT touch the slurm cluster.
./start.sh start --from-ghcr prod

# 2. Assemble the /etc/slurm payload in the per-cluster dir (adds job_submit.lua
#    + qos_config.lua that used to come from shared/). Runs as root on the host.
./start.sh stage-slurm-cluster-conf skipjack
ls ${BASE_DATA}/slurm/skipjack/       # job_submit.lua qos_config.lua rates.lua cli_filter.lua prolog.sh epilog.sh …

# 3. Regenerate ONLY the slurm overlay (dir-binds) via the now-updated coldfront.
docker exec coldfront coldfront shell -c \
  "from coldfront.plugins.arch_sync import cluster_manager as m; from coldfront.plugins.arch_sync.models import SlurmCluster as C; m.write_compose_overlay(C.objects.get(name='skipjack'))"

# 4. Surgical RECREATE of the core with the SAME :prod images (NEVER slurm start
#    --from-ghcr). Service KEYS: slurmctld is BRANDED (skipjack-slurmctld);
#    slurmrestd/slurmdbd are UNBRANDED (only container_name is branded).
docker compose --project-name skipjack-slurm --env-file <env> -f docker-compose-slurm-skipjack.yml \
  up -d --force-recreate --no-deps skipjack-slurmctld slurmrestd    # slurmdbd unchanged

# 5. Verify (the .nfs disappears on recreate).
docker inspect skipjack-slurmctld --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}' | grep /etc/slurm
docker exec skipjack-slurmctld ls /etc/slurm
docker exec skipjack-slurmctld scontrol ping && docker exec skipjack-slurmctld sinfo
bash scripts/verify-deploy.sh
```

After this the "edit in place" flow above holds forever.

### Import an existing cluster's topology (nodes/partitions/QOS from slurmrestd)

```bash
docker exec coldfront coldfront import_slurm_cluster --cluster skipjack --check   # preview
docker exec coldfront coldfront import_slurm_cluster --cluster skipjack --update  # apply
```

### DB ↔ slurm.conf direction, and closing the conf→DB loop (bare-metal / `.arch`)

Two directions, two sources of truth:

| Flow | Command | Source of truth |
|------|---------|-----------------|
| **DB → conf** (dev/Docker) | `write_slurm_conf(cluster)` | the ColdFront **DB** |
| **conf → DB** (bare-metal `.arch`) | `import_slurm_cluster --update` (reads the **live** cluster via slurmrestd, not the file) | the **`slurm.conf` file** |

There is **no file→DB parser**. When you hand-edit the bare-metal
`/etc/slurm/slurm.conf` (i.e. `slurm.conf.template.arch`), close the loop so
billing/gauges/allocations stay accurate — this is the **post-deploy step**:

```bash
# 1. edit /etc/slurm/slurm.conf on the controller
# 2. apply to the live cluster
scontrol reconfigure
# 3. dry-run the DB diff
docker exec coldfront coldfront import_slurm_cluster --cluster skipjack --update --check
# 4. write the DB
docker exec coldfront coldfront import_slurm_cluster --cluster skipjack --update
```

`--update` upserts `SlurmPartition`/`SlurmQOS`/`SlurmNode` and overwrites
Slurm-owned fields (`max_time`, `allow_qos`, `priority`, `cpus`, `real_memory`),
but **never** ColdFront policy fields (`billing_rate_tier`, `is_free`,
`tres_billing_weights`, `kind`, `hardware`, `tier_code`). **Global settings**
(auth, accounting, `JobSubmitPlugins`, JWT, `SelectType`, `ClusterName`,
`SlurmUser`) are the **immutable zone** of `.arch` — not modelled in the DB and
never imported back. Step 3 (`--check`) is automated as **check 13** of
`scripts/verify-deploy.sh` (WARN-only, bare-metal clusters); keep step 4
(`--update`) an explicit operator action (it mutates the DB).

**Wiring `.arch` as `/etc/slurm/slurm.conf` (Ansible, Stage 4):** by default the
`slurm_config` role deploys the DB-generated per-cluster bundle. To deploy the
hand-maintained static file instead, set in `clusters/<name>/group_vars/all.yml`:

```yaml
slurm_conf_src: "{{ playbook_dir }}/../../slurm/conf/slurm.conf.template.arch"
```

The role deploys that path verbatim (owner `slurm:slurm`, mode 0644) and notifies
`reconfigure slurm`. Leave `slurm_conf_src` unset for the DB-generated default.

> **Pitfall — empty partition fatals slurmctld:** a partition with **no compute
> nodes** used to render `Nodes=` empty and kill slurmctld with
> `fatal: Invalid node names in partition <p>` (`sinfo` → "Unable to contact
> slurm controller"). `generate_slurm_conf` now **skips** empty-node partitions
> (WARN per skip). To bring up a control-plane-only cluster (bare-metal compute
> added later), leave the partitions defined and regenerate — they self-suppress
> until nodes are registered as `SlurmNode(node_type="compute")`.

### Bare-metal node bring-up — troubleshooting (Stage 4 / `.arch`)

Field notes from standing up bare-metal GPU nodes (e.g. skipjack `ga1[38-43]`,
8× A100 each). The order below is the order failures surface: **config parses →
node authenticates (munge) → node registers (GRES) → jobs run (storage).** Fix
them in that order — a node that can't authenticate never gets its GRES checked.

**1) `slurm.conf` — one directive = one physical line.**
```text
error: _parse_next_key: Parsing error at unrecognized key: RealMemory
fatal: Unable to process configuration file
```
A `NodeName=` / `PartitionName=` split across lines — via a trailing `\`
continuation **or** a bare newline — is not reliably rejoined by Slurm's parser
(`slurmd` **and** `pam_slurm_adopt` both die; the latter also closes SSH sessions
right after key-accept). The wrapped `RealMemory=…` / `OverSubscribe=…` is read as
a bogus top-level key. **Fix:** collapse every directive onto a single physical
line — never wrap. In `slurm.conf.template.arch` the commented bring-up blocks are
also one-line each, so uncomment by deleting the leading `# ` only.

**2) The node parses its LOCAL file — deploy the file `slurm_conf_src` points at.**
A bare-metal `slurmd` reads its own `/etc/slurm/slurm.conf` (no `--conf-server`).
`scontrol reconfigure` on the **container** controller reloads the *controller's*
config — it does **not** rewrite a node's local file. And if the playbook is run
with `-e slurm_conf_src=/…/clusters/<name>/slurm.conf`, **that** staged file is
deployed — editing the repo `.arch` template does nothing for that path. Fix the
file `slurm_conf_src` actually resolves to, then re-run **without** `--check`.
Verify the deployed copy before restarting:
```bash
# on the node: line-initial RealMemory or a trailing backslash = still wrapped
ssh <node> "grep -nE '^\\s*RealMemory=|\\\\$' /etc/slurm/slurm.conf" || echo clean
ssh <node> 'slurmd -C'          # prints node config if the file parses
```

**3) GRES — `INVAL`, `gres/gpu count reported lower than configured (0 < 8)`.**
slurmd is not built `--with-nvml` (scaffold default `slurm_enable_nvml: false`),
so it cannot auto-detect GPUs — it needs an explicit `gres.conf`. Confirm the
hardware, then stage a per-cluster `gres.conf` (the `slurm_client` role copies
`clusters/<name>/gres.conf` → `/etc/slurm/gres.conf`):
```bash
ssh <node> 'nvidia-smi -L; ls -l /dev/nvidia[0-7]'   # expect 8 GPUs + 8 device files
cat > clusters/skipjack/gres.conf <<'EOF'
AutoDetect=off
NodeName=ga1[38-43] Name=gpu Type=a100 File=/dev/nvidia[0-7]
EOF
```
`Type=` (lowercase) must match slurm.conf `Gres=gpu:<type>:<n>`. After deploy:
`systemctl restart slurmd` on the node, then
`scontrol update NodeName=<n> State=RESUME`.

**4) Munge — `down*` (unreachable) vs `down` (reachable).**
In `sinfo`, the **asterisk** is the tell: `down*` = slurmctld can't reach/authenticate
the node (munge); plain `down` = the node registered fine and is just parked →
`scontrol update NodeName=<n> State=RESUME`.
```text
error: Munge decode failed: Invalid credential
auth/munge: _print_cred: ENCODED: Wed Dec 31 19:00:00 1969   # epoch-0 = key mismatch
Protocol authentication error
```
Cause: the node's `/etc/munge/munge.key` ≠ the controller's, **or** the key was
replaced but `munged` wasn't restarted (it caches the key at daemon start). A
`--limit <onenode>` run leaves the rest stale. Fix directly (don't rely on the
Ansible handler firing):
```bash
# compare every node + controller — all must match
for n in ga139 ga140 ga141 ga142 ga143; do echo -n "$n "; ssh $n md5sum /etc/munge/munge.key; done
docker exec <cluster>-slurmctld md5sum /etc/munge/munge.key
# push the good key + RESTART munged (mandatory) + slurmd
for n in …; do ssh $n 'chown munge:munge /etc/munge/munge.key && chmod 400 /etc/munge/munge.key && systemctl restart munge && systemctl restart slurmd'; done
# ground truth: credential minted on the node must decode in the controller
ssh <node> 'munge -n' | docker exec -i <cluster>-slurmctld unmunge | grep STATUS   # STATUS: Success (0)
```

**5) `srun --pty` hangs right after `[BILLING] loaded!`.**
The job launched (slurmstepd + the SPANK plugin ran — that's the BILLING line),
but `bash` blocks on startup because its home (`/weka/home/…`) isn't mounted on
the node. **Storage is outside the Slurm Ansible scope**, so a node with healthy
Slurm can still have no `/weka`. Discriminate:
```bash
srun -p a100 hostname                       # works instantly? → task launch + IO OK
srun -p a100 --chdir=/tmp --pty bash -c id  # works but plain --pty hangs? → it's home
ssh <node> 'mount | grep -i weka; timeout 5 stat /weka/home/jhu/<user> || echo HOME_UNREACHABLE'
```
`srun hostname` OK + `--chdir=/tmp` OK + Weka unmounted → mount WekaFS on the node.
If `srun hostname` **also** hangs, it's the srun stdio reverse-path (`SrunPortRange`
/ firewall between the node and the login host), not the home directory.

### Rebrand the Slurm core to GHCR on a live host

Rename the running core containers from local `arch/*` images to
`ghcr.io/jhu-arch/cloack/*:prod` **without** touching `slurm.conf` (same image
bits — `arch/*` and the GHCR tags share the same image ID). The 2026-07-06 saga
came from re-running `slurm start`, which re-runs `write_slurm_conf` and clobbers
the served config (`SrunPortRange=0-0` / `DbdHost` breakage). Do **not** do that.

> **Prod repo is not rw-mounted:** base `docker-compose.yml` only mounts
> `./ansible` and `./slurm` under `/opt/arch` (dev mounts `.:/opt/arch:rw`). So
> `write_compose_overlay` run inside coldfront writes to an **ephemeral container
> path**, not the host overlay — you must `docker cp` it out.

```bash
# 1. regen core-only overlay with GHCR env passed via `docker exec -e`
#    (NOT os.environ set mid-python — that does not apply)
docker exec -e CLOACK_FROM_GHCR=1 -e CLOACK_IMAGE_TAG=prod coldfront python -c \
  "import django,os; os.environ.setdefault('DJANGO_SETTINGS_MODULE','coldfront.config.settings'); django.setup(); \
   from coldfront.plugins.arch_sync.models import SlurmCluster; \
   from coldfront.plugins.arch_sync.cluster_manager import write_compose_overlay; \
   print(write_compose_overlay(SlurmCluster.objects.get(name='skipjack'), core_only=True))"
# 2. copy the overlay out to the host
docker cp coldfront:/opt/arch/docker-compose-slurm-skipjack.yml ./docker-compose-slurm-skipjack.yml
# 3. validate
docker compose -p skipjack-slurm --env-file .env -f docker-compose-slurm-skipjack.yml config -q
# 4. guard-diff vs a backup of the RUNNING overlay — ONLY image lines (arch/*→ghcr:prod)
#    and configless mounts (slurm.conf :ro, cgroup→<cluster>/, +plugstack) may change.
#    If a compute node / port / container_name changes → STOP.
# 5. recreate the core with GHCR (controller-recreate blip only; no write_slurm_conf)
docker compose -p skipjack-slurm --env-file .env -f docker-compose-slurm-skipjack.yml up -d
```

> **NEVER `./start.sh slurm start --from-ghcr` on a live host** — it re-runs
> `write_slurm_conf` and clobbers the served `slurm.conf` (the 2026-07-06 saga).

### Configless node bring-up

Under configless (`SlurmCluster.configless=True`), the containerized controller
serves the `.conf` set (`slurm.conf` `:ro`, `gres.conf`, `cgroup.conf`,
`plugstack.conf`) on host port **6817**. Bare-metal compute nodes run
`slurmd --conf-server <host>:6817`; login nodes run `sackd --conf-server
<host>:6817` (23.11+ client-config daemon — gives `srun`/`sacct`/`sinfo` config
with no local file). Client `.lua`, `prolog.d`/`epilog.d`, and SPANK `.so`
binaries **stay local** (configless serves `.conf`, never binaries).

> **Per-cluster mount files must exist before `docker compose up -d`:**
> `slurm.conf`, `gres.conf`, `cgroup.conf`, `plugstack.conf` must all exist as
> **files** in `${BASE_DATA}/slurm/<cluster>/` (e.g.
> `/opt/mprov/cloack/var/slurm/skipjack/`). A missing mount source makes Docker
> create a phantom **empty directory** there and slurmctld dies reading it as a
> dir. Stage any missing one: `cp slurm/conf/<file> ${BASE_DATA}/slurm/<cluster>/`.

After a controller recreate, nodes show `unk` (UNKNOWN) briefly while configless
`slurmd` re-registers — normal blip, wait ~1 min then re-check `sinfo -N`. If the
controller log warns `Node <X> appears to have a different slurm.conf than the
slurmctld` (CONF_HASH mismatch), that node has a stale cached/local `slurm.conf`
— make the config actually converge (do **not** just set `DebugFlags=NO_CONF_HASH`):

```bash
ssh <node> 'systemctl restart slurmd'                      # re-fetch via --conf-server
docker exec <cluster>-slurmctld scontrol update NodeName=<X> State=RESUME
```

### `docker compose up -d` is idempotent

With the overlay file **and** `.env` unchanged, `up -d` is a no-op — it only
recreates containers whose resolved config (image tag / env / volume / port)
diverged; unchanged ones stay `Running` (no node blip), and the one-shot
`munge-init` re-runs harmlessly. `.env` is in the config hash, so an `.env` change
**will** recreate the affected containers. Preview without touching anything:

```bash
docker compose -p <cluster>-slurm --env-file .env -f docker-compose-slurm-<cluster>.yml up -d --dry-run
```

> **NEVER `docker compose build` / `up --build` on a pull host (mgmt02)** — it
> recreates the local `arch/*` images and drifts from the tested GHCR artifacts.
> Builds happen on the Stage 1/2 build host → publish → mgmt02 **pulls**. Also:
> `In Use (U)` in `docker image ls` is per image-**ID**, not per tag — many tags
> (`arch/*`, `ghcr :dev/:prod/:rc/:sha`) share one ID, so pruning redundant tags
> frees ~0 disk.

---

## 6. Billing

```bash
docker exec coldfront coldfront collect_billing            # compute charges
docker exec coldfront coldfront sync_job_usage             # write SlurmAccountUsagePeriod rows (the "Compute Billing" button)
docker exec coldfront coldfront generate_billing_report    # CSV/report artifacts
docker exec coldfront coldfront recalculate_billing        # recompute from scratch
docker exec coldfront coldfront shadow_compare_weekly_cap_usage --threshold 2.0   # sreport vs REST diff
```

> `SlurmAccountUsagePeriod` (the "Usage & Charges" rows) is written **only** by
> `sync_job_usage` (the Compute Billing button) — the scheduled sync and the
> `test-jobs` seeder never upsert billing periods.

---

## 7. Stack Lifecycle & Health

```bash
./start.sh init dev            # bootstrap dirs, generate .env
./start.sh start dev           # build + start all services
./start.sh update              # PROD day-to-day: git pull + migrate + restart (non-destructive)
./start.sh restart dev         # stop, rebuild images, start (keeps data)
./start.sh rebuild dev         # rebuild images --no-cache
./start.sh stop dev
./start.sh destroy dev         # remove containers (keeps volumes)
./start.sh deploy dev -y       # DESTRUCTIVE bootstrap/reset (wipes everything)
./start.sh clean-all           # full cleanup: stop + remove arch volumes/images only
```

> `update` = non-destructive (preserves DB/LDAP/volumes/jobs).
> `deploy` = bootstrap/reset (wipes) — use only on a fresh machine or an
> intentional nuke.

Post-deploy verification (silent stdout ≠ healthy):

```bash
bash scripts/verify-deploy.sh   # container status, logs, sacct State, node health,
                                # stuck PD jobs, CLI-fallback, django-q audit, OIDC secret
```

### Migrations & schedules

```bash
docker exec coldfront coldfront migrate
docker exec coldfront coldfront setup_schedules      # reconcile django-q cron dict
docker restart coldfront                             # redeploy custom/ overlay + reload code
```

### Database & backups

The nightly stack backup is `scripts/backup.sh` (cron `/etc/cron.d/cloack-backup`,
daily 00:30, installed by `./start.sh init` on Linux root hosts). It writes to
**`${BASE_DATA}/backup/`** (singular — on mgmt02 `/opt/mprov/cloack/var/backup/`),
NOT to the legacy `/opt/mprov/cloack/backups/` folder, which only holds one-off
manual dumps from July 2026.

```bash
ls -lah ${BASE_DATA}/backup/                 # postgres_<ts>.sql.gz + ldap_<ts>.ldif.gz + slurmacct_<ts>.sql.gz per night
tail -20 ${BASE_DATA}/backup/backup.log      # one "done — N component(s) failed" line per run; N must be 0
cat /etc/cron.d/cloack-backup                # 30 0 * * * root <repo>/scripts/backup.sh >> .../backup.log
bash scripts/backup.sh                       # run one now (exit code = failed components)
RETENTION_DAYS=30 bash scripts/backup.sh     # default 14; retention only prunes the .gz the script itself wrote
```

> **Never pipe a `postgres_*.sql.gz` into the live postgres.** `pg_dumpall`
> carries `\connect` + `setval()` — it replays into the real DBs and rewinds
> every sequence (→ `duplicate key` across the portal). Inspect/restore only in
> a throwaway container (`docker run --rm --name pg-restore postgres:16` + `psql -f`),
> and after ANY restore/import with explicit PKs run
> `docker exec coldfront coldfront resync_sequences` (also `--check`).

Size per database (all DBs share the one `postgres` container; the `admin`
superuser connects over the trusted local socket, no password):

```bash
docker exec postgres psql -U admin -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database WHERE NOT datistemplate ORDER BY pg_database_size(datname) DESC;"
```

```text
  datname   | pg_size_pretty
------------+----------------
 slurm_jobs | 600 MB      # arch_sync_slurmjob — jobs imported by collect_sacct
 coldfront  | 284 MB
 helpdesk   | 257 MB
 keycloak   | 25 MB
 postgres   | 7519 kB
 admin      | 7519 kB
```

(mgmt02, 2026-09-12 — the compressed nightly dump was 149 MB and growing
~3.5 MB/day; `slurm_jobs` is the usual leader.)

Largest tables per DB, with the write/vacuum counters that tell real data
from churn (`n_tup_upd` = rows rewritten since the stats were last reset;
`n_tup_hot_upd` ≈ `n_tup_upd` means those rewrites were cheap in-page HOT
updates — WAL/IO cost, not disk growth; `n_dead_tup` ≫ `n_live_tup` with an
old `last_av` means autovacuum is not keeping up):

```bash
for db in slurm_jobs coldfront helpdesk; do
  echo "=== $db ==="
  docker exec postgres psql -U admin -d "$db" -c "
    SELECT relname,
           pg_size_pretty(pg_total_relation_size(relid)) AS total,
           pg_size_pretty(pg_relation_size(relid))       AS heap,
           pg_size_pretty(pg_indexes_size(relid))        AS idx,
           n_live_tup, n_dead_tup, n_tup_upd, n_tup_hot_upd,
           to_char(last_autovacuum,'MM-DD HH24:MI') AS last_av, autovacuum_count
    FROM pg_stat_user_tables
    ORDER BY pg_total_relation_size(relid) DESC LIMIT 6;"
done
docker exec postgres psql -U admin -c "SHOW autovacuum_vacuum_scale_factor;" -c "SHOW autovacuum_naptime;"
```

What it showed on mgmt02 (2026-09-12, defaults `scale_factor=0.2`, `naptime=1min`):

```text
=== slurm_jobs ===
      relname       | total  |  heap  |  idx   | n_live_tup | n_dead_tup | n_tup_upd | n_tup_hot_upd | last_av     | autovacuum_count
 arch_sync_slurmjob | 592 MB | 247 MB | 345 MB |     577218 |      89577 |   8496697 |       8360001 | 09-11 10:51 | 1
=== coldfront ===
 allocation_historicalallocationattributeusage | 148 MB | 104 MB | 45 MB | 1284409 | 1247 |     0 | 0 |             | 0
 arch_sync_apitokenusage                       |  67 MB |  20 MB | 46 MB |  160554 | 1601 |     0 | 0 |             | 0
 django_admin_log                              |  28 MB |  25 MB |  3 MB |   78241 |   28 |     0 | 0 |             | 0
 django_session                                |  11 MB |   3 MB |  7 MB |    5538 |    9 | 53212 | 1 | 09-13 00:40 | 48
=== helpdesk ===
 django_session                         | 190 MB | 131 MB | 59 MB |   26207 |  167 |   113 |    0 |             | 0
 hd_kb_document                         |  23 MB |  13 MB |  4 MB |       0 |    0 |     0 |    0 |             | 0
 helpdesk_extensions_assignmentdecision |  11 MB |   7 MB |  4 MB |   41653 |   22 |     0 |    0 |             | 0
```

How to read it:

- **`arch_sync_slurmjob` is real data, not bloat.** 577 k jobs × ~450 B/row of
  heap is the expected row width. The 8.5 M rewrites (15× the row count) are
  the 15-min `collect_sacct` re-upserting every job in its window with
  `update_or_create` — 98 % of them HOT, so they cost WAL and CPU, not disk.
  `idx` > `heap` because the model carries 15 indexes; the single-column ones
  on `account`/`user`/`partition`/`qos` are shadowed by the composite
  `(<col>, start_time)` indexes and could be dropped. The nightly dump grows
  with skipjack's job volume (whale days ≈ 20–37 k jobs).
- **`allocation_historicalallocationattributeusage`** (upstream ColdFront
  `simple_history` on `AllocationAttributeUsage`) gets one row per gauge write —
  every allocation, every 15-min `sync_slurm` — and nothing prunes it: 1.28 M
  rows in ~45 days.
- **`arch_sync_apitokenusage`** (`APITokenUsage`, Staff → API Token Usage) is
  one row per authenticated `/api/v1/` request, mostly the helpdesk polling;
  no retention either.
- **helpdesk `django_session` 190 MB for 26 k rows** — `clearsessions` is not
  scheduled anywhere in the stack (nor on coldfront). Helpdesk
  `SESSION_COOKIE_AGE` is 1 day, so nearly all of those rows are expired
  OIDC sessions (~5 KB each: the id/access tokens live in the session).
  `hd_kb_document` shows `n_live_tup=0` only because it was never ANALYZEd.
- `django_admin_log` (78 k rows) is the django-q audit trail — permanent by
  design (`cleanup_task_queue` purges `Task`, not `LogEntry`).

Remediation (all read-safe except the `VACUUM FULL`, which takes a brief
exclusive lock on that one table — sessions/history tables, fine at any hour):

```bash
# 1. Expired sessions — removes only expire_date < now(); run on both apps
docker exec helpdesk python /opt/helpdesk/manage.py clearsessions
docker exec coldfront coldfront clearsessions
docker exec postgres psql -U admin -d helpdesk  -c "VACUUM (FULL, ANALYZE) django_session;"
docker exec postgres psql -U admin -d coldfront -c "VACUUM (FULL, ANALYZE) django_session;"

# 2. simple_history (django-simple-history 3.12): first drop consecutive rows
#    where nothing changed, then age out; --dry prints counts only
docker exec coldfront coldfront clean_duplicate_history --auto --dry
docker exec coldfront coldfront clean_duplicate_history --auto
docker exec coldfront coldfront clean_old_history --auto --days 90 --dry
docker exec coldfront coldfront clean_old_history --auto --days 90
docker exec postgres psql -U admin -d coldfront -c "VACUUM (FULL, ANALYZE) allocation_historicalallocationattributeusage;"

# 3. Refresh planner stats so n_live_tup stops reading 0 on never-vacuumed tables
docker exec postgres psql -U admin -d helpdesk -c "ANALYZE;"
```

Both `clearsessions` and the history pruning should become scheduled tasks
(django-q on coldfront, cron/qcluster on helpdesk) — until then re-run this
block whenever the nightly dump size jumps.

Slurm accounting (MariaDB in `slurm-db`) is dumped by the same script; its
own archive/purge policy lives in `slurmdbd.conf` (jobs 24 mo, steps/events
12 mo, usage 36 mo → `/var/lib/slurm/archive`).

### Edge topology — the two NPM layers (HTTP)

Edge/prod only. Two reverse proxies sit in front of the stack and they own
different things; knowing which is which is most of the troubleshooting.

The **JHU NPM** (`status`, `162.129.223.99`) is shared campus infrastructure —
it also fronts services that are not CLOACK. It holds the public DNS names, the
Let's Encrypt certificates, and forwards every CLOACK host to one address:
`https://172.16.1.2:443` (mgmt02).

The **in-stack NPM** (the `npm` compose service on mgmt02) receives all of them
on that single address and dispatches by `Host`/SNI to the right container:
`portal.*` → `coldfront:8000`, `auth.*` → `keycloak:8443`, `helpdesk.*` →
`helpdesk:8000`, `ood.*` → `ood:80`. Its whole configuration — proxy hosts,
certificate, the SSH TCP stream — is derived from `CLOACK_DOMAIN_JHU` /
`CLOACK_DOMAIN_SCHMIDT` and pushed in from git, so a fresh host reproduces the
edge with no manual clicks. In stages 1–3 there is no JHU NPM and the in-stack
one *is* the edge; that is what lets dev and staging exercise the same paths
under their own domains.

> **Forward ports differ per layer and are easy to get backwards.** On the JHU
> NPM the target is `https://172.16.1.2:443` — the in-stack NPM. On the in-stack
> NPM the target is the container's own port, e.g. `http://ood:80`. Never point
> the in-stack NPM at the OOD container's `:443`: it listens there, but that is
> Apache's stock `ssl.conf` `<VirtualHost _default_:443>` serving
> `/var/www/html` with a build-time self-signed cert (`CN=buildkitsandbox`) —
> both OnDemand vhosts are `*:80`. Measured with the same `Host:` header, `:80`
> answers `302 -> /pun/sys/dashboard` and `:443` answers 403 with the Rocky test
> page.

**Which layer produced an error.** Each NPM writes per-proxy-host logs under
`/data/logs/proxy-host-<id>_{access,error}.log`, and the access line carries
`$upstream_status $status`:

```
- 302 302 - GET https ood.<domain> "/"                    upstream and proxy agree
- 502 502 - GET https ood.<domain> "/pun/sys/dashboard"   the UPSTREAM returned 502
```

A 502 with `upstream_status` also 502 means that nginx relayed someone else's
error — look one hop further in, not at buffers or TLS. A proxy that generated
the error itself logs it in its own `_error.log`; an empty error log next to a
502 in the access log means the error came from upstream.

**Certificates.** A name missing from the SAN list of the cert that terminates
TLS produces a browser warning no proxy setting can fix. The in-stack cert is
minted at `init` for the service FQDNs of both domains; the public-facing one
lives on the JHU NPM. Issuing a new Let's Encrypt cert there needs the name to
resolve in **public** DNS first — certbot runs `--authenticator webroot`
(HTTP-01), so an unpublished name fails the challenge in a few seconds and NPM
surfaces it only as a red "Internal Error" on the SSL tab; the real reason is in
`docker logs <npm>` and `/tmp/letsencrypt-log/letsencrypt.log`.

### Email intake (arch-mta) — inbound mail → Helpdesk tickets

Edge/prod only (off in dev). Flow: MX → JHU NPM `:25` stream → `172.16.1.2:25`
(in-stack NPM) → `arch-mta:25` → `/maildrop/<queue>/new` → `get_email` cron → ticket.

```bash
./start.sh mta create                 # stage Maildirs + multi-SAN cert + mta.env
./start.sh mta start                   # BUILD arch/mta locally, then start (edge profile)
./start.sh mta start --from-ghcr prod  # PROD: PULL ghcr.io/jhu-arch/cloack/mta:prod (no build)
./start.sh mta status                  # health + virtual_mailbox_domains
./start.sh mta log                     # tail postfix
docker exec helpdesk python manage.py get_email   # drain Maildir → tickets
```

> **`--from-ghcr` needs GHCR read auth.** The ghcr overlay maps `arch-mta` →
> `ghcr.io/jhu-arch/cloack/mta:${tag}` with `build: !reset null` (pull-only). A
> private package fails with `[mta] GHCR pull failed (auth?...)` — log in with a
> **`read:packages`** token first:
> `echo "$PAT" | docker login ghcr.io -u <user> --password-stdin`. The tag defaults
> to the running `coldfront` tag, else `dev`. Confirm it pulled (not built):
> `docker inspect arch-mta --format '{{.Config.Image}}'` → `…/cloack/mta:prod`.
> Inside a `deploy --from-ghcr` (`CLOACK_FROM_GHCR=1`) arch-mta already comes up
> no-build; the flag is only for a standalone `mta start`.

**NPM streams (two hops, PROXY protocol OFF unless real sender IP is required):**

| NPM | Incoming | Forward Host | Forward Port | PROXY |
|-----|----------|--------------|--------------|-------|
| JHU external (`status`, `162.129.223.99`) | `25` | `172.16.1.2` (mgmt02 green-zone) | `25` | off |
| in-stack (compose) | `25` | `arch-mta` | `25` | off |

> Only turn PROXY protocol on (both streams + `MTA_PROXY_PROTOCOL=1`, so
> `smtpd_upstream_proxy_protocol = haproxy`) if arch-mta must log the **real**
> sender IP instead of the NPM IP; chaining it across two NPM streams is fragile.
> A stale global `QUEUE_EMAIL_BOX_*` in the host `helpdesk.env` overrides every
> queue to `imap` and breaks intake — `ensure_host_paths` auto-scrubs it, or
> `sed -i '/^QUEUE_EMAIL_BOX_/d' ${BASE_ETC}/helpdesk/helpdesk.env` then
> `./start.sh helpdesk stop && start` (a bare `docker restart helpdesk` is NOT enough).

---

## 8. Troubleshooting Quick Reference

| Symptom | Likely cause / fix |
|---------|--------------------|
| `slurm_load_partitions: Unable to contact slurm controller` | slurmctld died — check `docker logs <cluster>-slurmctld`. Common: `fatal: Invalid node names in partition` (empty-node partition) → regenerate `slurm.conf` (§5). |
| `getaddrinfo(slurmdbd:6819) failed` | slurmctld can't resolve `slurmdbd` — shared core services down or not on the same docker network. |
| Login node shows empty `/home` (users missing) | Container mounts the **dev named volume** instead of the real WekaFS. Prod must bind-mount `${HOME_DATA}:/home` / `${SCRATCH_DATA}:/scratch`. Check `docker inspect <node> --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'`. |
| Keycloak login still shows "Register" after disabling it | Old theme/realm still live — restage themes + `docker restart keycloak`, and re-run `keycloak-config` to apply `registrationAllowed=false` (§3). |
| Portal unreachable (`Up (unhealthy)`, curl 000) after visiting KC | HSTS poisoning (dev http portal). Unpoison the browser (§3). |
| `502 Bad Gateway` (nginx) on `/pun/*` after a successful OOD login, anonymous requests fine | Stale per-user PUN socket. `/var/run` is the container's writable layer, so `passenger.sock` outlives the PUN. Confirm with `docker exec ood /opt/ood/nginx_stage/sbin/nginx_stage pun -u <user> -a "<url>"` → `bind() … 98: Address already in use`. Since the `pun_pre_hook` landed this self-heals: the hook drops the socket before staging when nothing is listening. If an older image is running, `docker restart ood` (the entrypoint clears stale PUN state at boot) or per user `nginx_stage nginx -u <user> -s stop` then remove the socket. |
| A proxied request fails and you do not know which NPM to blame | Read `$upstream_status $status` in `/data/logs/proxy-host-<id>_access.log`. Equal values mean the error came from upstream — the proxy only relayed it. An empty `_error.log` next to a 502 confirms that layer is innocent. |
| NPM host shows **Online** but the name is unreachable (`tlsv1 unrecognized name`) | "Online" is NPM's own bookkeeping, not nginx reality. A failed certificate request leaves the host without a `listen 443 ssl` block: check `grep -E "listen\|ssl_certificate" /data/nginx/proxy_host/<id>.conf`. Assign an existing certificate and save to regenerate it. |
| Bind-mounted file looks truncated in a container | virtiofs cache — `docker restart <container>`; if not fixed, full recreate. |
| `ModuleNotFoundError: No module named 'coldfront.custom'` | Wrong import path in `shell -c` — use `coldfront.plugins.arch_sync`, not `coldfront.custom.plugins.arch_sync`. |
| TOTP banner won't clear after enrolment | `coldfront sync_totp_status --username <u>` to poll Keycloak now. |
| `sacctmgr modify` silently fails | Missing `where`: `modify account where name=X set parent=Y -i`. |
| `Nodes go down` after container recreate | slurmd auto-resumes on reconnect; if stuck: `scontrol update NodeName=<n> State=RESUME`. |
| `_parse_next_key: unrecognized key: RealMemory` → `fatal: Unable to process configuration file` (slurmd/`pam_slurm_adopt`) | A `NodeName=`/`PartitionName=` wrapped across lines (`\` or bare newline). Collapse to one physical line; fix the file `slurm_conf_src` points at, not the repo template (§5, bring-up #1–2). |
| Bare-metal node still broken after `scontrol reconfigure` | Reconfigure on the container controller doesn't rewrite a node's **local** `/etc/slurm/slurm.conf`; re-deploy the file + `systemctl restart slurmd` on the node (§5, bring-up #2). |
| Node `INVAL` / `gres/gpu count reported lower than configured (0 < N)` | No `--with-nvml` build → needs explicit `/etc/slurm/gres.conf` (`Name=gpu Type=<t> File=/dev/nvidia[0-N]`); verify `nvidia-smi -L`, then RESUME (§5, bring-up #3). |
| `sinfo` shows `down*` (asterisk) / `Munge decode failed: Invalid credential` / epoch-0 `ENCODED` timestamp | munge key mismatch **or** `munged` not restarted after a key change. Match `md5sum /etc/munge/munge.key` node↔controller, push good key, `systemctl restart munge && slurmd` (§5, bring-up #4). `down` without `*` = reachable, just `State=RESUME`. |
| `srun --pty` hangs right after `[BILLING] loaded!` | Job launched but `bash` blocks — home (`/weka/home/…`) not mounted on the node (storage is outside Slurm's Ansible scope). `srun … hostname` + `--chdir=/tmp` isolate it (§5, bring-up #5). |
| Nightly paid Projects disappeared | Guarded now (a337776), but check `seed_initial_data` marker on the `${BASE_DATA}` bind-mount. |
| Pending jobs `Reason=InvalidQOS` right after a QOS/tier change | Queued jobs keep the QOS they were submitted with; migrate with `scontrol update job <id> qos=<new> TimeLimit=<same>` (§5). |
| qcluster logs `arch_sync: reactivated <user>@<acct>` every 15 min | The user association still shows `MaxJobs`/`GrpJobs=0` after the clear; inspect with `sacctmgr show assoc … WOPLimits` (§5). |
| Keycloak `error="user_temporarily_disabled"` | Brute-force lockout after repeated bad password/TOTP, usually a client retrying on its own; inspect/clear via `attack-detection/brute-force/users/<id>` (§3). |
| `kcadm.sh … PKIX path building failed` | Self-signed cert: use `--server http://localhost:8080` inside the container, or `scripts/kcq.sh` (§3). |
| `Sync all users finished: … N users failed sync!` | `ModelDuplicateException` on e-mail: a local broker-created `<jhed>@<domain>` user holds the LDAP entry's e-mail. Login unaffected, but OnDemand and LDAP role mappers are. Repair with `converge_keycloak_federation` (§3). |
| Deep link on the Schmidt portal (`portal.<schmidt>/project/`) redirects to `auth.<jhu>/realms/jhu` | Pre-`37c8c1ec` `LOGIN_URL` was hardcoded to `/oidc/jhu/authenticate/`; since then `/oidc/authenticate/` picks the realm from the host (`portal.<CLOACK_DOMAIN_SCHMIDT>` → schmidt). Needs `CLOACK_DOMAIN_SCHMIDT` in the root `.env` + `docker restart coldfront qcluster`. Verify: `curl -sI -H 'Host: portal.<schmidt>' http://127.0.0.1:8000/oidc/authenticate/ \| grep -i location` inside the coldfront container. |
| Keycloak WARN `Expected String but attribute 'cn' has more values '[<group>, <Real Name>]'` on `ou=Groups` | Legacy migration groups carry the PI's real name as a second `cn`. Harmless; one-off `ldapmodify delete: cn` cleanup (§2). |
| "There are no backups" — `/opt/mprov/cloack/backups/` only has July files | Wrong folder: the cron writes to `${BASE_DATA}/backup/` (singular, `var/backup/` on mgmt02); check `backup.log` for `0 component(s) failed` (§7). |
| Nightly `postgres_*.sql.gz` growing several MB/day | Usually `slurm_jobs` (`arch_sync_slurmjob`): compare `n_live_tup` vs `n_dead_tup`/`n_tup_upd` per table before assuming real growth (§7). |

---

*Generated for CLOACK staff/admins. Commands reflect the management commands in
`coldfront/custom/plugins/arch_sync/management/commands/` and the `start.sh`
subcommands. Keep in sync when commands change.*
