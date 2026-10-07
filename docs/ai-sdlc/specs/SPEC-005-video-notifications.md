# SPEC-005: Video Session & Notifications
- Status: approved
- Source intent: `intent/telehealth-therapy-app.md` (Status: approved)
- Created: 2026-10-07
- Supersedes: monolith `SPEC-001-telehealth-marketplace-mvp.md` §REQ-06/07

## 1. Scope
- In (intent §5 L26-28, §2 L11):
  - Video join window via provider link, email + SMS reminders (queued), ICS already in SPEC-003
  - Provider TBD (Zoom or Daily.co); no recording in MVP
- Out: recording, in-app chat, native calendar sync

## 2. Roles & Permissions
| Action | Patient | Therapist | Admin |
|---|---|---|---|
| Join video of own booking T-15m→end | Y | Y assigned | — |

Cross-user join → 403. Admin never joins via this link.

## 3. Domain Model
- `bookings.video_url NULL` set on confirm via `VideoLinkService::generate(booking)`; unguessable, non-enumerable.
- Notification jobs keyed by `booking_id` + `type` for dedupe.

## 4. Requirements
| ID | Requirement | Intent trace |
|----|-------------|--------------|
| REQ-06 | On confirm queue T-24h + T-1h email and T-1h SMS (if phone); retries without double-send | §5 L27, §4 L22 |
| REQ-07 | Join allowed T-15m to `ends_at`; otherwise 403 with start time; URL never listable | §2 L11, §5 L26 |

## 5. Acceptance Criteria (Gherkin)
```gherkin
Feature: REQ-07 video window
  Scenario: early join blocked
    Given confirmed booking starting in 2h
    When patient requests join
    Then response is 403 with start time

Feature: REQ-07 authorization
  Scenario: stranger blocked
    Given confirmed booking of patient A
    When patient B requests join
    Then response is 403

Feature: REQ-06 reminders queued
  Scenario: confirm queues reminders
    When booking is confirmed
    Then T-24h email, T-1h email, T-1h SMS jobs are queued once
    And duplicate confirm event does not duplicate jobs
```

## 6. Non-Functional Requirements
- Queue Redis (`SendBookingReminders`); SMS failure does not fail booking; `log` driver in tests.
- Times stored UTC, rendered in viewer timezone.
- Security: no recording default; provider BAA required before PHI scale (intent §4 L21).

## 7. Integrations
- Video: provider-agnostic `app/Services/VideoLinkService.php`; Zoom/Daily.co URL stored. Decision open (intent L38).
- Notify: Mail + SMS behind interface; queued.

## 8. Admin / Filament
- Booking detail shows video link + reminder delivery status (read-only).

## 9. Edge Cases
- Provider outage → join page shows fallback + support contact; reminder retry with backoff.

## 10. Conflicts / Open Questions
- Conflict: Zoom vs Daily.co + recording need — intent L38. Surfaced; default provider-agnostic service.

## 11. Test Plan
- `tests/Feature/VideoJoinTest.php`: early 403, window 200, cross-user 403, past-booking 403.
- `tests/Feature/ReminderTest.php`: jobs queued once, fake mail/SMS sent, SMS failure tolerated.
