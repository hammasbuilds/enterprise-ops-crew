# enterprise-ops-crew

[![ci](https://github.com/hammasbuilds/enterprise-ops-crew/actions/workflows/ci.yml/badge.svg)](https://github.com/hammasbuilds/enterprise-ops-crew/actions/workflows/ci.yml)
![python](https://img.shields.io/badge/python-3.12-blue)
![deps](https://img.shields.io/badge/runtime%20deps-zero-success)
[![license](https://img.shields.io/badge/license-MIT-green)](LICENSE)

We built enterprise-ops-crew, a multi-agent back office that resolves tickets across four
enterprise systems (HRMS, ITSM, CRM, ERP) and stops before anything it cannot undo. The agents
are distinct roles with distinct authority (triage, resolver, approver), execution follows
declared playbooks rather than prompts, SLAs are measured in business hours, and the daily
report says what the crew finished and why it escalated the rest. It is for teams piloting
back-office automation who need every action to be auditable and every irreversible one to
wait for a person.

- In a product: the approval-gate workflow (refunds and credit-limit changes, four-eyes
  approval, audit trail) has a full UI in
  [Analyst-in-a-Box](https://github.com/hammasbuilds/analyst-in-a-box) under **Workflows**.

## What it does

- **Triage** with visible rules: category, priority, and abstention below a confidence floor,
  so a ticket nobody understood goes to a human instead of being guessed.
- **Playbooks**, not prompts: each category runs an ordered list of operations, so the audit
  answer is "playbook billing/v1, step 1 ran, step 2 stopped for approval by m.manager".
- **Risk declared per operation** (`READ`, `WRITE`, `EXTERNAL`, `IRREVERSIBLE`) by whoever
  wrote it; anything above the autonomous ceiling raises an approval request and moves
  nothing. `approve()` continues the same ticket, `reject()` escalates it with no action.
- **Missing information escalates**: a playbook never guesses an account number.
- **Outages are handed off**: an incident is logged in ITSM and escalated to on-call, never
  reported as resolved by the crew.
- **Business-hours SLAs**: weekends, working days, holidays and time waiting on the customer
  are excluded; burn (`0.9` = act now) is reported before the breach.
- **Validated status transitions**: illegal moves raise; reopening clears the resolution time.
- **Daily report**: autonomous resolution rate over tickets the crew finished with,
  escalations with reasons, tickets awaiting approval, at-risk SLAs, irreversible actions taken.

| System | Operations |
|---|---|
| HRMS | lookup employee `READ`, book leave `WRITE` |
| ITSM | reset password `WRITE`, open incident `WRITE`, grant access `IRREVERSIBLE` |
| CRM | lookup customer `READ`, send email `EXTERNAL` |
| ERP | lookup invoice `READ`, issue refund `IRREVERSIBLE` |

## Sample input / output

`python demo.py`: four tickets through the mock registry the tests use (no model, no network):

```
INPUT
   "Cannot log in, password reset needed"
   "I would like to book annual leave"   facts={'days': 3}
   "Production down, all users affected"
   "Refund for invoice 4421 please"   facts={'invoice_id': '4421', 'amount': 4200}

OUTPUT
   ticket                                         category     priority  status         steps
   ------------------------------------------------------------------------------------------------
   Cannot log in, password reset needed           access       NORMAL    RESOLVED       6
   I would like to book annual leave              hr           NORMAL    RESOLVED       7
   Production down, all users affected            incident     URGENT    ESCALATED      6
   Refund for invoice 4421 please                 billing      NORMAL    AWAITING_APPROVAL 7

   HELD FOR APPROVAL   Refund for invoice 4421 please
      blocked on       erp_issue_refund
      arguments        {'invoice_id': '4421', 'amount': 4200}

   calls actually made   ['itsm_reset_password', 'hrms_lookup_employee', 'hrms_book_leave', 'itsm_open_incident', 'erp_lookup_invoice']
   side effects          []
```

The refund stops at `AWAITING_APPROVAL`: `erp_lookup_invoice` did run, and `side_effects` is
empty, so no money moved. The outage is logged and handed to on-call.

## Results

- 40 tests, no ERP licence, no model, no network, covering transitions, triage (including word
  boundaries: `accessory` is not `access`), business hours (after-hours, weekends, holidays,
  multi-day, burn), autonomy, the approval gate, reporting and the audit trail.
- The key assertion is on side effects, not on the reported status:

```python
def test_an_irreversible_action_stops_for_a_human():
    assert result.status is Status.AWAITING_APPROVAL
    assert c.systems.side_effects == []       # no money moved
```

- Business hours, from the tests: Friday 16:00 to Monday 10:00 is 2 business hours; Monday
  09:00 to Wednesday 12:00 is 19.

## Tech stack

- Python 3.12, zero runtime dependencies (standard library dataclasses, enums, datetime)
- Mock HRMS / ITSM / CRM / ERP backends exposed as operations with declared risk; a real MCP
  server can implement the same `Operation` shape
- pytest, ruff, hatchling, uv, GitHub Actions

## Run it

```bash
git clone https://github.com/hammasbuilds/enterprise-ops-crew
cd enterprise-ops-crew
uv sync --all-groups
uv run pytest -q           # or: make test
uv run python demo.py
```

As a library:

```python
from crew import Crew, Ticket
from crew.systems import build_default_registry

crew = Crew(systems=build_default_registry())

t = crew.intake(Ticket(subject="Cannot log in, password reset needed", requester="e-1001"))
crew.work(t.id)                       # resolved autonomously

r = crew.intake(Ticket(subject="Refund for invoice 4421", requester="c-77"),
                invoice_id="4421", amount=4200)
crew.work(r.id)                       # AWAITING_APPROVAL, no money moved
crew.approve(r.id, approver="manager")

crew.daily_report()                   # resolution rate, escalations, at-risk SLAs
crew.sla_status(r.id)                 # business hours, not wall clock
```

- Facts a playbook needs (`invoice_id`, `amount`, `days`) are passed to `intake()` explicitly;
  they are not extracted from the ticket text.
- Raising `autonomous_ceiling` on the registry removes the gate for that risk level.
- One SLA policy per crew.

## Licence

MIT, see [LICENSE](LICENSE).
