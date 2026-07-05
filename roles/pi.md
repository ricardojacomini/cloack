# PI — managing a project, its allocations, and billing

## Who you are + scope

You are a **Principal Investigator** with an ARCH Portal account that has the
PI flag on your `UserProfile`. You own one or more **Projects**, request
**Allocations** against them (default scavenger / paid Pay-Per-Use /
condo Hardware-Credit), set Weekly Caps, add users to your projects,
and read the billing report monthly. Authentication-wise you are a
normal user — read [`user.md`](user.md) for the login flow and the
job-submission basics.

If you also operate the cluster (deploy services, configure Slurm) you
are a different role — read [`admin.md`](admin.md). If you handle
helpdesk tickets you are also Staff — see [`staff.md`](staff.md).

## Quick start

### 1. Accept the Terms of Service

Before you can do anything as a PI you must accept the current ToS.
You will see a banner at the top of every portal page until you do.
Click the banner → check the box → submit. ToS acceptance is also
required for every user you add to your projects.

You can re-read what you agreed to at any time: on `/user/user-profile/`
the **Terms of Service** row shows `Accepted vX` as a link that opens the
exact version you accepted (read-only, with a Print button).

### 2. Create a project and request an allocation

```
/project/create/                       → create a new project
/project/<id>/                         → project detail page
  Click "Request Resource Allocation"  → choose a resource (Cluster)
```

A project with empty / `default` abbreviation is the **scavenger-only**
default project: it cannot host paid allocations. To request a paid
allocation, give the project a real abbreviation (`proj1`, `alpha`,
etc.) when you create it.

For paid allocations, fill in:

- **IO Number** — your funding source. Allocations without an IO
  number are treated as no-charge regardless of partition.
- **storage_tb** / **backup_tb** if you need storage and/or backup.
- **Weekly Cap ($)** **or** **Weekly Cap (hours)** to bound spend.

### 3. Add users to a project

```
/project/<id>/ → "Add Users"
```

Users must already have an ARCH Portal account (registered via
`/user/jhu/register/` or `/user/schmidt/register/`) and they must have
accepted the current ToS. Once you add them, the Slurm associations
provision in real time via the post-save signal — they can submit
jobs against the allocation immediately.

### 4. Set or update a Weekly Cap

```
/allocation/<id>/ → click the Weekly Cap field → edit → save
```

- Dollar cap takes precedence over hours cap if both are set.
- One cap is enforced **per allocation** (so users sharing the
  allocation share the cap).
- The gauge below the Core Usage gauge shows current spend vs cap.

### 5. Read the billing report

```
/billing/report/                        → all-PI view (staff-scoped)
/billing/report-by-pi/                  → per-PI view, your projects only
/user/user-profile/ → "My Usage & Billing"  → PDF download
```

Storage and backup are billed monthly regardless of compute activity.
GPU jobs charge GPU only — their CPU hours appear for transparency but
are excluded from the total due (the GPU-only billing rule, sourced in
`billing_views.calculate_allocation_charges`).

## Reference matrices

### Allocation lifecycle

| State | Set by | What you can do |
|---|---|---|
| New | PI on request | Edit IO / storage / backup; cancel before review |
| Pending review | Auto on submit | Cannot edit; ColdFront staff approve / deny |
| Active | Staff approval | Add users, set Weekly Cap, submit jobs |
| Renewal Requested | PI | Same as Active, with a renewal note for staff |
| Expired / Archived | Auto on end_date | Read-only; usage history preserved |

### Billing model decision table

| If… | …pick |
|---|---|
| Default scavenger PI project (no abbreviation, no IO) | **No Charge** — auto-set by `_ensure_allocation_billing` signal |
| Schmidt PI (`ssci-*`) | **No Charge** — funded by the Schmidt programme |
| Paid project, IO number, no upfront hardware purchase | **Pay-Per-Use** |
| Paid project, IO number, you bought hardware credit | **Condo** (Hardware-Credit holder; `credits_used` gated by `billing_type=credit_holder`) |

### Cost Center setup (multi-IO)

You can pool multiple IO numbers under a single project / allocation:

```
/billing/cost-centers/                  → manage your Cost Centers
  → "Add Cost Center"
  → Link Cost Center to a project       → ProjectCostCenter M2M
  → Assign a Cost Center to an allocation → AllocationCostCenter M2M
```

ColdFront enforces the Slurm gate: a paid JHU allocation without a
**pay-role** Cost Center linked at the project AND allocation level is
silently skipped by `slurm_sync` (no Slurm account created).

### Weekly Cap: dollars vs hours

| Cap | Units | When to use |
|---|---|---|
| Dollar cap | `$` (converted to `GrpTRESMins=billing=X` via `$X × 1000 × 60`) | Most allocations. Predictable monthly spend bound. |
| Hours cap | core-hours | Hardware-Credit / academic allocations where dollars are not the budget unit. |

Per partition the dollar value translates differently because of
`TRESBillingWeights`. A `med` partition with `CPU=1.0` costs more
per core-hour than `low` with `CPU=0.5`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Banner says "Accept Terms of Service" but I already accepted | Old ToS version superseded by a new one | Visit `/user/user-profile/` and accept again; everything you provisioned before stays valid |
| Adding a user to a project does nothing visible in Slurm | They have not accepted ToS yet — the post-save signal skips provisioning until they do | Email the user with the ToS link; once they accept, the immediate-provisioning signal fires |
| "Request Resource Allocation" button is not shown | This project is a `default` project (no abbreviation) | Create a new project with a real abbreviation; default projects are scavenger-only by design |
| Allocation is Active but jobs fail with `Invalid account` | The Slurm account hasn't been provisioned yet, or the user's ToS gate has not cleared | Wait for the next `sync_slurm` (every 15 min) or trigger it via Admin actions on the allocation |
| Storage charge is $0 but I set `storage_tb=0.20` | `0.20 TB × $0.025 = $0.005`, which rounds to $0.00 | This is expected at small sizes; ratio is correct |
| Billing report shows nothing for the past month | The "Computing Billing" button on `/billing/report/` was never pressed | This is by design: `SlurmAccountUsagePeriod` rows are written only when staff explicitly clicks Compute Billing |
| Hardware Credit pool reports 0 hours despite usage | The allocation's `billing_type` is not `credit_holder` | Set `billing_type=credit_holder` on the AllocationBilling row via Django Admin, or via the Project Detail page if you have staff scope |

## Where to go next

- `/user/help/` — in-app help with live updates.
- [`coldfront/BILLING_SETUP_GUIDE.md`](../../coldfront/BILLING_SETUP_GUIDE.md)
  — full billing reference (rates, periods, invoices, Cost Centers).
- [`user.md`](user.md) — login + day-to-day job submission.
- [`README.md`](README.md) — the role index.
