# Staff — Helpdesk operator handling tickets

## Who you are + scope

You are a **Helpdesk staff member** with the `helpdesk-staff` LDAP group
on your account (or, for team leads, `helpdesk-admin`). You triage the
queue, work tickets through the skill-based assignment matrix, escalate
on FQR / SLA breaches, and read the analytics dashboards. You authenticate
the same way every other user does — see [`user.md`](user.md) for the
login flow.

If you also approve allocations or run cluster ops, you are also a
**Superuser** — read [`superuser.md`](superuser.md). If you author or
read tickets without working them, you are a regular [`user.md`](user.md).

## Quick start

### 1. Triage a new ticket

```
helpdesk → /helpdesk/                   → dashboard with open tickets
  → click ticket → ticket detail view
  → Set Priority, Queue, Owner
  → Assign by skill                     → use the skill-match panel on the right
```

The ticket view banner shows urgency (SLA / FQR / strike / overdue).
The right-hand context sidebar shows the submitter's past tickets,
their PI / project / department, GPU / partition references found in
the title / description, similar open tickets, and suggested
collaborators ranked by skill match.

### 2. Respond + classify

```
"Send email as <ResponseTemplate>"      → picks a template
  → {name}/{your_name}/{issue}/{ticket_number}/{survey_url}/{signature}
    are auto-substituted
  → choose the email class (FQR / strike 1-3 / LQR)
  → server-side scrub catches any token you didn't fill in
```

The classification advances the `TicketFollowUpTracker` and feeds the
analytics dashboard. Internal notes use `public=False`; customer
emails use `public=True` — collaborators are forced to internal-only.

### 3. Escalate

```
ticket → "Escalate" action               → applies the StaffMember.policy
                                           (3-strike, FQR / SLA breach)
                                          → reassigns to the next staff in
                                            the routing matrix
                                          → logs the routing decision
                                            for audit
```

`/extensions/analytics/routing/` shows the live decision audit trail.

### 4. Read the analytics

```
/dashboard/                              → 3 tabs:
                                            • Department / Project
                                            • PI
                                            • Resource (GPU / CPU / Storage / Backup)
/reports/                                → 8 stock helpdesk reports rendered
                                            inline as Chart.js bar / line charts
/extensions/skill-matrix/                → skills × categories grid
/extensions/calendar/                    → workload calendar
/extensions/schedule/                    → engineer × day × slot grid +
                                            Time Allocations
```

### 5. Run a public feedback ticket through the Feedback queue

Tickets created via the **public** submit form with a category whose
`show_on_submit=True` are auto-routed to the **Feedback & Suggestions**
queue (slug `feedback`, owner = `ARCH_SUPERUSERS[0]`). They do not
pollute General Support — review them on a separate cadence.

## Reference matrices

### Ticket statuses

| ID | Status | Open? | Notes |
|---|---|---|---|
| 1 | Open | Yes | New, untouched. |
| 2 | Reopened | Yes | Customer or staff reopened after Resolved / Closed. |
| 3 | Resolved | No | Customer can still reply. |
| 4 | Closed | No | Locked. |
| 5 | Duplicate | No | Customer redirected to canonical ticket. |
| **6** | **Archived** | **No** | **ARCH Portal addition.** Resolved / Closed → Archived. Single exit: Archived → Reopened. |

### Priority + SLA windows

| Priority | First Quality Response (FQR) | SLA |
|---|---|---|
| 1 — Critical | 30 minutes | 4 hours |
| 2 — High | 2 hours | 1 business day |
| 3 — Normal | 1 business day | 3 business days |
| 4 — Low | 2 business days | 5 business days |
| 5 — Very Low | best-effort | best-effort |

Breaches trigger escalation per `StaffMember.policy` (3-strike rule
shipped by default).

### Skill matrix routing

| Field | What it does |
|---|---|
| `StaffSkill(technology_domain, ticket_category)` | A staff's competence per (domain, category) pair |
| `cognitive_process` | Bloom's taxonomy 1-6; **SME threshold** is the singleton `HelpdeskSMEConfig.cognitive_threshold` (default 6 = Evaluation) |
| `is_mentor` | Admin override per (staff, category); promotes a staff to SME regardless of `cognitive_process` |
| `TicketCategory.show_on_submit` | Surface this category on the public submit form |
| `TicketCategory.default_queue` | Auto-route the ticket to a specific queue (e.g. `feedback`) |
| `TicketCategory.default_assignee` | Auto-route to a product owner |

### Helpdesk extensions

| URL | What it is |
|---|---|
| `/extensions/skill-matrix/` | Skills × categories grid; admin-editable |
| `/extensions/calendar/` | Workload calendar (assigned hours per staff per week) |
| `/extensions/schedule/` | Engineer × day × 30-min-slot grid + Time Allocations |
| `/extensions/schedule/request/` | Staff submits a weekly allocation request |
| `/extensions/schedule/my-requests/` | Staff's own request history |
| `/extensions/schedule/admin/requests/` | Admin reviews + approves / denies requests |
| `/extensions/deletion-requests/` | Superuser queue for justified ticket deletions |
| `/extensions/trello/` | Trello browser (Kanban / Cards / Lists / Gantt) |
| `/extensions/analytics/routing/` | Routing decision audit dashboard |

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| New status 6 (Archived) does not show on a tab I expected | Archived is not in `OPEN_STATUSES = (1, 2)` | Archived is a closed status by design — it shows on the All / Closed tabs only |
| Skill matrix shows 0 mentors despite recent edits | `hd_sme_threshold` cache TTL = 5 min | Wait, or `cache.delete("hd_sme_threshold")` from the Django shell |
| Customer email goes out with `{name}` not substituted | The classification dropdown was bypassed (typed directly into the editor) | Pre-save scrub catches this and re-renders the template; review the ticket and resend if needed |
| Public feedback ticket routed to General Support | The category does NOT have `default_queue` set | Edit the `TicketCategory` in Django Admin and set `default_queue=Feedback` |
| Collaborator cannot add a comment | Server-side guard: collaborator can only add `public=False` notes | This is by design; only the owner sends customer emails |
| Reports page is slow | 8 sub-reports rendered inline as Chart.js | The two open-by-default cards (Queue-by-Status, User-by-Status) are the hot path; collapse the rest if needed |

## Where to go next

- `/staff/help/` — in-app help (Staff Help Page).
- `/extensions/` — admin UI for skill matrix, schedule grid,
  analytics, deletion-request queue.
- [`superuser.md`](superuser.md) — if you also approve allocations
  or run cluster ops.
- [`README.md`](README.md) — the role index.
