# Learning Unlimited — Google Summer of Code 2026

## ESP Website — Communications Improvements (Repo-Validated Final Draft)

| | |
|---|---|
| **Name** | Dhirender Choudhary |
| **GitHub** | github.com/Dhirenderchoudhary |
| **Email** | Dhirenderchoudhary0001@gmail.com |
| **Mentors** | Miles Calabresi, William Gearty |
| **Difficulty** | Hard |
| **Expected Size** | 350 hours |
| **Status** | For Review |
| **Date** | 18 March 2026 |

---

## Summary

The ESP Website communications panel works, but it lacks modern features expected in tools like MailChimp. This project adds:

1. Reusable email templates with management UI
2. Scheduled sending with date/time controls
3. Teacher-facing communications for enrolled students

The design is additive and preserves ESP's existing delivery path:

`MessageRequest -> dbmail_cron/cronmail -> TextOfEmail -> SMTP provider`

This minimizes risk, avoids infrastructure complexity, and keeps existing workflows stable.

---

## 1. Project Goal and Context

The comm panel is a core admin tool (recipient filtering, rich-text compose, preview, queued sending), but recurring communication tasks still need manual workarounds.

This project modernizes the experience while staying aligned with existing code architecture:

- Main comm flow handler: `esp/esp/program/modules/handlers/commmodule.py`
- Queue model and email objects: `esp/esp/dbmail/models.py`
- Scheduled processing hook via `processed_by`: `esp/esp/dbmail/cronmail.py`
- Current compose template with warnings and SmartText list: `esp/esp/program/modules/templates/commmodule/step2.html`
- Teacher module patterns and permissions: `esp/esp/program/modules/handlers/teacherclassregmodule.py`
- Admin module search registration: `esp/esp/program/modules/admin_search.py`

---

## 2. Expected Outcomes

This proposal directly targets the official expected outcomes:

1. Template selection UI in comm panel with preview thumbnails
2. Ability to create, edit, and manage reusable email templates
3. Scheduled email sending with date/time picker
4. Integration with SendGrid scheduling API (or cron-based alternative)
5. Teacher comm panel for emailing enrolled students
6. UI to view and manage scheduled/pending emails
7. Documentation for template creation and email scheduling

Implementation note: cron-based scheduling is the primary path because it already exists in ESP (`MessageRequest.processed_by` plus cron filtering), while direct SendGrid scheduling can be evaluated as optional/advanced mode if mentors prefer.

### Codebase validation notes

Repository scan confirms key assumptions:

- `MessageRequest.processed_by` is already nullable/indexed and used by cron eligibility checks.
- `cronmail.py` already processes requests with `(processed_by <= now) OR processed_by is null`.
- Existing comm warnings are already centralized in `get_mailer_warnings()`.
- `@needs_teacher` decorators and teach-module handler patterns are well established.
- `PersistentQueryFilter.create_from_Q()` is already heavily used in ESP modules.
- `AdminSearchEntry` registration flow is already in production use.

---

## 3. Feature Design

### Feature A: Template System (Issue #2885)

#### Planned deliverables

- `EmailTemplate` model for reusable subject/body content
- Template CRUD management pages for admins
- Template picker in compose flow (step 2)
- Preview thumbnail/list support for faster selection
- Optional program-scoped templates

#### Technical fit

- Compose/preview flow is centralized in `commmodule.py`.
- Existing rich text + SmartText rendering can be reused.
- Template pages can be indexed through `AdminSearchEntry`.

---

### Feature B: Scheduled Sending (Issue #3951)

#### Planned deliverables

- Date/time picker in compose flow
- Validation for future-only scheduling
- Scheduled queue page with pending message list
- Cancellation for queued-but-unsent messages
- Clear schedule confirmation text in preview/final screens

#### Technical fit

- Existing `MessageRequest.processed_by` already supports deferred execution.
- `cronmail.py` already respects `processed_by__lte=now`.
- No Celery/Redis requirement; zero mandatory infra changes.

#### SendGrid integration strategy

- Primary: native ESP cron scheduling
- Optional: evaluate SendGrid API scheduling for compatible send paths
- Fallback: if API constraints appear, continue with stable cron path

---

### Feature C: Teacher Comm Panel (Issue #3833)

#### Planned deliverables

- New teacher-facing comm module in teach workflow
- Class selection scoped to teacher-owned classes
- Recipient generation from enrolled students only
- Compose/preview/send flow using existing MessageRequest pipeline
- Permission hardening and rate limiting

#### Technical fit

- `@needs_teacher` and teach-module patterns already exist.
- Recipient scope can be built via `PersistentQueryFilter.create_from_Q()`.
- Existing pipeline preserves auditability and delivery behavior.

---

## 4. UI/UX Plan

### Compose step improvements

- Template picker with thumbnail/list preview
- Scheduling section with clear date/time controls
- Inline validation for past timestamps
- Existing mailer warnings preserved and extended
- Recipient scope badges and clearer labels

### Preview step improvements

- Show schedule summary when enabled
- Keep warnings visible before final queue action
- Consistent action order for safer final confirmation

### Teacher flow UX

- Read-only recipient scope to avoid accidental overreach
- Minimal compose surface for faster routine communication
- Consistent warning patterns with admin comm flow

---

## 5. Atomic PR Plan (Mentor-Review Friendly)

This plan avoids oversized PRs. Each PR is independently reviewable.

1. Template model + migration + admin registration
2. Template management views + URLs + permission checks
3. Template picker + preview/list UI integration in compose
4. Template endpoint + compose integration tests
5. Scheduling controls + backend validation
6. Scheduled queue management view + cancellation action
7. Scheduling tests (future/past/UTC behavior)
8. Teacher comm module skeleton + class scoping
9. Teacher compose/preview/send integration
10. Teacher permission/rate-limit hardening + tests
11. AdminSearchEntry registration for new views
12. End-to-end integration tests and docs polish

Each PR will include:

- Clear acceptance criteria
- Backward compatibility note
- Test additions and expected coverage
- Short rollback plan if regression appears

---

## 6. Testing and Validation Strategy

### Unit tests

- Template create/edit/archive behavior
- Scheduling datetime validation and storage correctness
- Teacher ownership and recipient filtering checks

### Integration tests

- Template select -> compose -> preview -> queue flow
- Scheduled message ignored until eligible cron window
- Teacher send reaches only enrolled students in owned classes

### Security tests

- Non-teacher access is denied for teacher module routes
- Forged class IDs are rejected server-side
- Recipient expansion cannot exceed allowed scope

---

## 7. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Teacher overreach in recipient selection | Server-side class ownership validation on every POST |
| Scheduling confusion across time zones | UTC storage, explicit local display, clear preview confirmation |
| Regressions in existing comm flow | Additive design and targeted regression tests |
| Scope expansion beyond 350 hours | Strict atomic PR boundaries and stretch-goal gating |
| Template misuse/formatting breakage | Validation plus preview-first workflow |

---

## 8. Additional Improvements

High-impact addons are detailed in sections 8A and 8B with priority, effort, dependencies, and scope guardrails.

---

## 8A. Addons for Review (Review Here)

This section is intentionally concise so mentors can evaluate optional scope quickly.

### Core Addons (high value, low risk)

1. **Recipient Diff Preview**
- Before final send, show how current recipients differ from the last similar send (added/removed/total).
- Benefit: catches accidental audience expansion before queueing.

2. **Required Variable Checker**
- Render the message against a small recipient sample (for example first 3 users) and block queueing if unresolved SmartText/template errors are detected.
- Benefit: avoids broken personalization in production emails.

### Stretch Addons (valuable, moderate effort)

3. **Delivery Health Panel**
- Admin dashboard widget/page showing queued, scheduled, sent, and failed counts (7/30-day windows).
- Benefit: gives operational visibility without a full analytics subsystem.

4. **Manual Retry for Failed Sends**
- Add retry actions for failed `TextOfEmail` records with controlled batching.
- Benefit: makes failed-delivery handling actionable for admins.

### Optional Addons (nice-to-have)

5. **Quiet Hours Scheduling Guardrail**
- After scheduling UI is complete, warn when emails are scheduled in configured quiet hours (default server-time window, e.g. 10pm-6am) and suggest the next safer slot.
- Benefit: improves recipient experience and reduces timing mistakes.

6. **Audience Send Frequency Reminder**
- Show an informational reminder such as "This is your Nth send to this audience in the last 7 days" based on exact recipient scope matches.
- Benefit: reduces duplicate-style send mistakes without fuzzy matching complexity.

---

## 8B. Prioritized Addon Roadmap (Mentor View)

To keep scope safe within 350 hours, addons will only be started after core milestones are stable.

| Addon | Priority | Effort | Depends On | Acceptance Signal |
|---|---|---|---|---|
| Recipient Diff Preview | High | Small-Medium | Core comm flow | Preview shows added/removed/total recipients before final queue |
| Required Variable Checker | High | Small | Template + compose integration | Submit blocked on unresolved placeholders unless explicitly confirmed |
| Delivery Health Panel | Medium | Medium | Scheduling + status data | Admin can see queued/scheduled/sent/failed metrics for 7/30 days |
| Manual Retry for Failed Sends | Medium | Medium | Failed delivery listing | Admin can retry selected failed sends with safety limits and rate limits |
| Quiet Hours Guardrail | Low-Medium | Small | Scheduling UI | Warning shown for scheduled time in configured quiet-hours window |
| Audience Send Frequency Reminder | Low | Small | Message history lookup | Reminder appears with exact audience-match send count in last 7 days |

### Addon execution order

1. Recipient Diff Preview
2. Required Variable Checker
3. Delivery Health Panel
4. Manual Retry for Failed Sends
5. Quiet Hours Guardrail
6. Audience Send Frequency Reminder

### Scope guardrail

- If schedule risk appears, only the first two addons (both High priority) will be included.
- Remaining addons stay explicitly optional and will be mentor-prioritized.

---

## 9. Feature Acceptance Criteria

### Templates

- Admin can create/edit/archive templates from UI.
- Selecting a template pre-fills subject/body in compose flow.
- Template usage does not break existing compose-from-scratch flow.
- Template management pages are discoverable in admin search.

### Scheduling

- Past datetime is rejected (client + server side).
- Future datetime is stored and message is not processed early.
- Scheduled queue page lists pending records accurately.
- Cancelled pending sends are clearly represented in UI/audit fields.

### Teacher comm panel

- Only teachers can access teacher comm routes.
- Teacher can only target enrolled students in own classes.
- Forged class IDs from other teachers are rejected.
- Teacher messages use the same audit/send pipeline as admin comm.

---

## 10. Relevant Skills and Learning Outcomes

This project directly develops:

- Django views, forms, and module handlers
- HTML email templating and rendering safety
- Email delivery systems (SendGrid and SMTP pathways)
- Cron-based scheduling and queued task processing
- Frontend JavaScript for date/time pickers and template interactions
- UX design for communication workflows
- Database schema design for templates and scheduling metadata

---

## 11. Timeline (12 Weeks)

### Community Bonding

- Confirm acceptance criteria with mentors
- Finalize model and route conventions
- Submit one preparatory small PR

### Weeks 1-4: Templates

- Data model, management UI, compose integration, tests

### Weeks 5-7: Scheduling

- Date/time controls, validation, queue management, tests

### Weeks 8-11: Teacher Comm Panel

- Teacher module, scoped recipients, hardening, tests

### Week 12: Finalization

- Documentation, regression checks, mentor feedback, final polish

---

## 11A. Prototype Artifacts and Screenshots

To support this proposal with concrete UX direction, a clickable prototype was prepared with representative screens.

### Prototype source

- `prototype/commpanel-prototype/index.html` (interactive screen prototype)
- `prototype/commpanel-prototype/styles.css` (visual system and responsive layout)
- `prototype/commpanel-prototype/app.js` (view switching and URL-state handling)

### Screenshot assets for proposal attachment

- `screenshots/prototype/prototype_01_admin_compose.png`
- `screenshots/prototype/prototype_02_template_library.png`
- `screenshots/prototype/prototype_03_scheduled_queue.png`
- `screenshots/prototype/prototype_04_teacher_flow.png`

### Prototype coverage map

- Admin compose flow with template picker and scheduling controls
- Template management interface with lifecycle actions
- Scheduled/pending queue management view
- Teacher-scoped communication flow with recipient guardrails

These prototype assets are illustrative and are intended to communicate interaction direction, not final production styling.

Prototype styling note: screens intentionally follow ESP's existing UI conventions (classic form/table/button patterns) rather than introducing a new visual system.

---

## 12. Success Metrics and Why This Proposal Is Strong

### Success metrics

- Reduced compose time for repeat campaigns (templates).
- Increased on-time delivery for reminders (scheduling).
- Reduced admin relays for teacher-to-student communication.
- Low regression rate in existing admin comm workflows.

### Why this proposal is strong

- Matches official project scope and expected outcomes
- Anchored in current ESP architecture and proven extension points
- Split into mentor-review-friendly atomic PRs
- Balances feature ambition with 350-hour execution realism
- Prioritizes safety, auditability, and maintainability

This plan brings ESP communications closer to modern email tooling while remaining practical for the GSoC timeline and codebase constraints.
