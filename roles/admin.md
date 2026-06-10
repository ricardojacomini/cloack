# Admin — cluster / DevOps operator running start.sh and Ansible

## Who you are + scope

You operate the ARCH Portal stack on **physical hosts**: the dev VM on
Proxmox, the **edumaster** bare-metal validation host, and the
production **portal** bare-metal host. You run `./start.sh deploy /
start / publish / promote`, you ship images to GHCR, you run
`ansible-playbook` for the bare-metal Slurm cluster install, and you
own the rotation of the GHCR PATs.

You are the only role that needs read-write access to GHCR. You
typically also have `is_superuser=True` in ColdFront — for the cluster
ops Django side read [`superuser.md`](superuser.md). For helpdesk
work see [`staff.md`](staff.md). You authenticate the same way every
other user does ([`user.md`](user.md)).

## Quick start

Five recipes you will run most days. Every command assumes you have
already `git clone`'d the repo at the canonical path on whatever host
you are on.

### 1. Local laptop dev (Stage 1)

```bash
cd ~/arch_jhu_dsai_schmidt
./start.sh deploy dev -y --paid-accounts   # full clean + reseed dev stack
bash scripts/verify-deploy.sh              # confirm 8/8 checks green
```

Use this when you want a clean known-good state. It wipes named
volumes and reseeds 4 test users (`ext-staff`, `ext-user`,
`ssci-staff`, `ssci-user`, password `123`).

### 2. Build and publish on the Proxmox VM (Stage 2)

```bash
ssh proxmox-vm
cd ~/jhu-arch
git pull origin dev
./start.sh start dev          # local build, no GHCR
bash scripts/verify-deploy.sh # optional smoke
./start.sh publish            # docker push :dev + :sha-<git-short>
```

Publish refuses with a clear error when the working tree is dirty
(the `:sha-*` tag would lie about content). Override with
`--tag <something-other-than-dev>` if you genuinely want a throwaway tag.

### 3. Promote on edumaster (Stage 3)

```bash
ssh edumaster
cd ~/jhu-arch
git pull origin dev                          # configs / templates only
./start.sh start --from-ghcr dev             # docker pull :dev, no build
bash scripts/verify-deploy.sh                # mandatory pre-flight
./start.sh promote                           # :dev -> :prod + :rc-<UTC>
# Rollback if a bad promotion slipped through:
./start.sh promote --rollback rc-2026-06-09-1430
```

`promote` runs `verify-deploy.sh` as a pre-flight gate by default; use
`--skip-verify` only as an emergency override. **Never** run `./start.sh
deploy --from-ghcr` — it is hard-refused. Stages 3 and 4 must never
populate test data (the deploy command's job).

### 4. Deploy on prod portal (Stage 4)

```bash
ssh portal
cd ~/jhu-arch
git pull origin dev                          # configs only
./start.sh start --from-ghcr prod            # docker pull :prod, no build
bash scripts/verify-deploy.sh
./start.sh ldap-reinit                       # only if LDAP needs re-seed
```

Portal can run Slurm via Docker (pull the slurm images) **or** via
bare-metal Ansible (see playbook recipes below). The compose overlay
controls which one is active.

### 5. Bare-metal Slurm via Ansible

```bash
cd ~/jhu-arch
./start.sh slurm ansible all                 # regenerate ansible/ tree
ansible-playbook -i ansible/clusters/<name>/inventory.ini \
  ansible/slurm-server/playbook.yml          # slurmctld + slurmdbd + slurmrestd
ansible-playbook -i ansible/clusters/<name>/inventory.ini \
  ansible/slurm-client/playbook.yml          # slurmd on every compute node
ansible-playbook -i ansible/clusters/<name>/inventory.ini \
  ansible/slurm-verify/playbook.yml          # read-only pre-flight (no changes)
```

`slurm-verify` is safe to run any time and reports a 14-check matrix
across OS, UIDs, munge, Slurm version, SPANK support, ports, SSSD TTL,
and PAM stack.

## Reference matrices

### 4-stage deploy pipeline

| Stage | Host | What you run | Result | Image flow |
|---|---|---|---|---|
| **1** Laptop coding | macOS arm64 | `./start.sh deploy dev -y --paid-accounts` | Full local dev stack with test data | `arch/<svc>` built locally |
| **1**→**2** | Laptop | `git push origin dev` | Source code is on GitHub | — |
| **2** dev VM build | Proxmox Linux amd64 | `git pull origin dev` | Code on the builder host | — |
| **2** | Proxmox | `./start.sh start dev` | Local build matching prod arch | `arch/<svc>` built locally on amd64 |
| **2** | Proxmox | `./start.sh publish` | `:dev` + `:sha-*` pushed to GHCR | `ghcr.io/jhu-arch/cloack/<svc>:dev` + `:sha-<git-short>` |
| **3** edumaster bare-metal | Linux amd64 | `git pull origin dev` | Configs / templates on edumaster | — |
| **3** | edumaster | `./start.sh start --from-ghcr dev` | Stack runs the `:dev` images, no build | `ghcr.io/...:dev` pulled |
| **3** | edumaster | `bash scripts/verify-deploy.sh` | 8-check validation result | — |
| **3** | edumaster | `./start.sh promote` | `:dev` retagged → `:prod` + `:rc-<UTC>` | `ghcr.io/...:prod` + `:rc-*` pushed |
| **4** portal bare-metal | Linux amd64 | `git pull origin dev` | Configs on portal | — |
| **4** | portal | `./start.sh start --from-ghcr prod` | Stack runs the `:prod` images | `ghcr.io/...:prod` pulled |
| **4** | portal | `./start.sh ldap-reinit` (manual) | LDAP re-seeded if needed | — |

Direction of artefacts:

```
git origin/dev                          GHCR registry
       ↓                                    ↓
laptop ──push──> origin            Stage 2 ──push :dev, :sha-*─> GHCR
       ↑                                    ↓
proxmox ──pull──> origin           Stage 3 ──pull :dev, push :prod, :rc-*─> GHCR
       ↑                                    ↓
edumaster ──pull──> origin         Stage 4 ──pull :prod
       ↑                                    ↑
portal ──pull──> origin            (portal never pushes)
```

### GHCR registry namespace and PATs

| Stage | PAT scope | Operations |
|---|---|---|
| 1 (laptop) | none | git push only |
| 2 (Proxmox) | `GHCR_BUILDER_PAT` (write) | `docker push :dev` + `:sha-*` |
| 3 (edumaster) | `GHCR_BUILDER_PAT` (write) | `docker pull :dev`, `docker push :prod` + `:rc-*` |
| 4 (portal) | `GHCR_CONSUMER_PAT` (read-only) | `docker pull :prod` only |

Registry path: `ghcr.io/jhu-arch/cloack/<image>:<tag>`.

Cross-org topology: the repo lives in the `cloack` GitHub org, packages
live in the `jhu-arch` org. The two PATs are **fine-grained**, scoped
to the `jhu-arch` org, and **SSO-authorized** for Johns Hopkins
University Enterprise. Rotation cadence: ≤ 1 year (fine-grained PAT
maximum lifetime).

### Image inventory (13 — 6 cloack apps + 7 slurm; cn-rockylinux is build-only)

| Track | Image | Built from | Promoted past `:dev`? |
|---|---|---|---|
| Cloack | `coldfront`, `helpdesk`, `keycloak`, `keycloak-config`, `openldap`, `phpldapadmin`, `postgres` | each service's `Dockerfile` | Yes — Stage 3 + 4 |
| Slurm (base) | `cn-rockylinux` | `slurm/base/Dockerfile` | **No** — build dep only, never pulled by runtime hosts |
| Slurm (services) | `slurmctld`, `slurmdbd`, `slurmrestd`, `slurm-munge`, `mariadb`, `slurmd` | `slurm/<svc>/Dockerfile` | Yes (in Docker-Slurm Mode A) — Stage 3 + 4 |

Note that `cn-rockylinux` is consumed at **build time** by the other
Slurm images (each does a `FROM arch/cn-rockylinux`). Once a Slurm
service image is pushed, the cn-rockylinux layers travel inside it —
runtime hosts (Stage 3 + 4) never pull cn-rockylinux as a standalone
image.

### Stage 4 dual-mode Slurm

| Mode | Slurm runs as | What portal pulls | When to use |
|---|---|---|---|
| **A — Docker Slurm** | containers | 6 cloack apps + 6 Slurm services | All-Docker portal, smaller HPC scale |
| **B — Bare-metal Slurm** | services on real hosts via Ansible roles | 6 cloack apps only | Real HPC scale, dedicated compute nodes, kernel-level features |

In Mode B the operator runs `ansible/slurm-server/playbook.yml` and
`ansible/slurm-client/playbook.yml` against the cluster inventory.
`cluster_manager.write_compose_overlay` honours
`CLOACK_FROM_GHCR=1` to rewrite Slurm image refs to `ghcr.io/...:prod`
in Mode A.

### `start.sh` subcommands by stage

| Subcommand | Stage(s) | What it does |
|---|---|---|
| `init [dev]` | 1, 2 | Bootstrap user/group/dirs/`.env`. Run once before first `start`. |
| `start [dev]` | 1, 2 | Build (or skip build) and start managed services. |
| `start --from-ghcr [dev\|prod]` | 3, 4 | Pull from GHCR, skip build. Refuses if `docker-compose-ghcr.yml` is missing. |
| `deploy [dev] -y [--paid-accounts]` | 1, 2 | Destructive: clean + init + start + populate test data. Hard-refused with `--from-ghcr`. |
| `publish [--tag <tag>] [--check]` | 2 | Retag local `arch/<svc>` as `ghcr.io/...:<tag>` + `:sha-<short>` and push. Refuses dirty working tree. |
| `promote [--from <tag>] [--rc <label>] [--check] [--rollback <rc-tag>] [--skip-verify]` | 3 | Pull `:dev`, run `verify-deploy.sh`, retag + push `:prod` + `:rc-<UTC>`. |
| `ldap-reinit [dev]` | 3, 4 | Re-initialise OpenLDAP from migration LDIF, force sync with ColdFront. |
| `slurm {create\|start\|stop\|restart\|status\|test-jobs\|ansible\|prod-sim}` | varies | Per-cluster Slurm operations. `ansible all` regenerates the bare-metal scaffold. |
| `helpdesk {start\|populate\|restart\|stop}` | 1, 2 | Helpdesk container lifecycle. |
| `doctor` | any | Read-only validation of `.env` ↔ realm config ↔ TLS cert ↔ Host header drift. |

### Ansible playbooks

| Playbook | What it does | When to run |
|---|---|---|
| `ansible/slurm-server/playbook.yml` | Compiles Slurm from source, installs slurmctld / slurmdbd / slurmrestd on the controller groups | Bare-metal install or major Slurm version bump |
| `ansible/slurm-client/playbook.yml` | Same build stack plus slurmd on every compute node and login node | New compute node, or after `slurm-server` |
| `ansible/slurm-verify/playbook.yml` | Read-only 14-check pre-flight (OS, UIDs, munge, Slurm version, SPANK, ports, SSSD TTL, PAM stack). No changes applied. | Any time you want a confidence check |
| `ansible/sssd-flush/playbook.yml` | Runs `sss_cache -E` on every login node | `sync_ldap` already triggers this per cluster; manual run useful after a large LDIF import |
| `cloack-deploy` RPM | Thin packaging of the Ansible tree + a `cloack-deploy` CLI wrapper | Sites that prefer an RPM-driven flow over `ansible-playbook` directly |

### Environment variables you will set on Stage 3 / Stage 4 hosts

Phase 2 (commit `a525237`) added multi-FQDN reverse-proxy support. The
following env vars are **opt-in**; leave them empty in single-FQDN dev
to keep current behaviour.

| Variable | Purpose |
|---|---|
| `ARCH_PUBLIC_HOST` | The portal FQDN (`portal.arch.jhu.edu`). Used everywhere. |
| `KEYCLOAK_HOSTNAME_JHU` | JHU realm FQDN (`auth.arch.jhu.edu`). Empty → single-FQDN. |
| `KEYCLOAK_HOSTNAME_SCHMIDT` | Schmidt realm FQDN. Empty → single-FQDN. |
| `KC_HOSTNAME` | Empty in multi-FQDN; Keycloak uses inbound Host instead. |
| `KC_HOSTNAME_BACKCHANNEL_DYNAMIC` | `true` in multi-FQDN deploys (trust `X-Forwarded-Host`). |
| `HELPDESK_SCRIPT_NAME` | `/helpdesk` in prod to route `portal.arch.jhu.edu/helpdesk` to the helpdesk container. Empty → no prefix. |
| `CLOACK_IMAGE_TAG` | Auto-set by `--from-ghcr`; override only for rollback (`sha-abc123` or `rc-YYYY-MM-DD-HHMM`). |
| `CLOACK_FROM_GHCR` | Auto-set by `--from-ghcr`; do not set by hand. |

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `keycloak-config Error... exit 5` on first deploy | `jq` could not index into an error response on `/auth/flows` (realm not imported yet) | Already fixed in commit `cd35aad`; if reappears, check the `_GHCR_IMAGE_REWRITE_RE` regex hasn't been broken by a new image name pattern |
| `keycloak-config Error... exit 3` on first deploy | `jq` tried `ascii_downcase` on a null `displayName` (Keycloak default executions list) | Fixed in `8443758` — re-pull the latest dev if you see this |
| `curl: URL malformed` in configure-keycloak.sh | Copied flow has a literal space in its sub-flow alias (`passwordless-browser forms`) | Fixed in `d0dfafe` — `${forms_sub_alias// /%20}` URL-encodes the space |
| `IntegrityError: null value in column "totp_enrolled"` | Postgres trigger `tr_auth_user_after_insert_fn` was created before the `totp_enrolled` column existed | Fixed in `1bffe47` — drop the trigger and re-run `ldap_sync_postgres_run_it_once.py` |
| Themes do not apply on first deploy (default Keycloak look) | `configure_realm_themes` ran before `import_keycloak_providers` so PUT returned 404 silently | Fixed in `b7efa22` — re-runs the idempotent block after realms are imported |
| `publish: working tree is dirty` | You have uncommitted source changes | Commit them, or use `./start.sh publish --tag wip-<something>` to bypass the SHA-tag promise |
| `deploy --from-ghcr is not allowed` | Stages 3 and 4 must never populate test data | Use `./start.sh start --from-ghcr <tag>` plus `./start.sh ldap-reinit` instead |
| `publish: no GHCR auth in ~/.docker/config.json` | Builder PAT not loaded | `echo $GHCR_BUILDER_PAT \| docker login ghcr.io -u <user> --password-stdin` |
| `--from-ghcr: docker-compose-ghcr.yml not found` | Phase 0 file missing in the repo on this host | Pull latest dev (`52b709f` introduced the file) |
| `pgrep -f "start.sh deploy"` loop never exits | The waiter's own command line contains the pattern, deadlock | Track `$DEPLOY_PID` directly with `wait $DEPLOY_PID`; see memory `feedback_pgrep_self_match_deadlock` |
| Slurm overlay still emits `arch/<svc>` after `--from-ghcr` | `CLOACK_FROM_GHCR` was not exported when `write_compose_overlay` ran | Re-run `./start.sh slurm create <name>` after exporting the var |
| `configure_realm_frontend_urls` did nothing visible | `KEYCLOAK_HOSTNAME_JHU` / `_SCHMIDT` are empty | Set them in `.env` for multi-FQDN, or leave them empty for single-FQDN — the function is a no-op then by design |

## Where to go next

- [`docs/ansible-remote-deploy.md`](../ansible-remote-deploy.md) — full
  bare-metal Ansible deploy guide, RPM install, idempotent re-runs.
- [`docs/break-glass-admin.md`](../break-glass-admin.md) — `/admin/`
  break-glass rotation (vault, audit log, alerts).
- [`slurm/README.md`](../../slurm/README.md) — full Slurm config
  reference (partitions, QOS, weekly cap, CLI tools).
- [`slurm/ANSIBLE.md`](../../slurm/ANSIBLE.md) — bare-metal Slurm install
  via the `cloack-deploy` RPM path.
- `.claude/plans/deploy-pipeline-4-stages.md` — the design plan behind
  the matrix on this page; read when the model needs to evolve.
- [`superuser.md`](superuser.md) — ColdFront / Django admin side
  (drain nodes, approve allocations, qcluster audit, OTP resets).
- [`README.md`](README.md) — the role index.
