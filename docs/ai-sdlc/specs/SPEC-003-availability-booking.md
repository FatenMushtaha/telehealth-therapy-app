# SPEC-003: Availability & Booking
- Status: approved
- Source intent: `intent/telehealth-therapy-app.md` (Status: approved)
- Created: 2026-10-07
- Supersedes: monolith `SPEC-001-telehealth-marketplace-mvp.md` §REQ-03/04/08/10

## 1. Scope
- In (intent §2 L11-12):
  - Therapist weekly availability CRUD, patient booking with atomic hold, cancellation + hold expiry, ICS download
  - Depends on SPEC-002 (approved therapist), feeds SPEC-004 (payment) + SPEC-005 (video/reminders)
- Out: Stripe logic (SPEC-004), video URL generation (SPEC-005), guided matching

## 2. Roles & Permissions
| Action | Patient | Therapist | Admin |
|---|---|---|---|
| Set own availability/rates | — | Y own | — |
| Book / view own bookings | Y own | — | Y all read |
| Accept/reschedule assigned | — | Y assigned | — |
| Cancel own/assigned booking | Y own | Y assigned | Y |

## 3. Domain Model
- `availability_slots{id, therapist_id FK, starts_at UTC, ends_at UTC, status:open|held|booked|cancelled, UNIQUE(therapist_id,starts_at), INDEX(status,starts_at)}` — 15–120 min, future only.
- `bookings{id, patient_id FK, therapist_id FK, slot_id FK UNIQUE, status:pending|confirmed|cancelled|completed|no_show, price_cents, cancel_policy_version}`.
- `booking_status_changes{id, booking_id, from, to, actor_id, created_at}`.

Invariants: one booking per slot; past slots immutable; price snapshot at booking.

## 4. Requirements
| ID | Requirement | Intent trace |
|----|-------------|--------------|
| REQ-03 | Create/edit availability; overlapping ranges rejected; past edits rejected | §2 L12 |
| REQ-04 | Booking `open→held→confirmed` atomically; concurrent same-slot yields exactly 1 confirmed (409 loser); hold expires 15m | §2 L11 |
| REQ-08 | Cancel before 24h cutoff → slot `open`, payment flagged refund-pending (SPEC-004 executes refund); after cutoff → flagged per policy | Open Q L39 |
| REQ-10 | ICS download for confirmed bookings | §5 L28 |

## 5. Acceptance Criteria (Gherkin)
```gherkin
Feature: REQ-03 overlap guard
  Scenario: overlapping availability rejected
    Given existing slot 10:00-11:00
    When therapist creates 10:30-11:30
    Then response is 422 overlap

Feature: REQ-04 no double-booking
  Scenario: concurrent confirm same slot
    Given slot S is open
    When two patients confirm S concurrently
    Then one receives confirmed and one receives 409 slot-taken

Feature: REQ-08 cancellation cutoff
  Scenario: pre-cutoff cancel reopens slot
    Given confirmed booking >24h away
    When patient cancels
    Then slot is open and payment flagged refund-pending

Feature: REQ-10 ICS
  Scenario: download ICS
    Given confirmed booking
    When participant downloads ICS
    Then file contains UID, DTSTART UTC, DURATION
```

## 6. Non-Functional Requirements
- Concurrency: DB transaction + `SELECT ... FOR UPDATE` or unique-constraint guard in `app/Actions/Bookings/CreateBooking.php`.
- Queue: `ExpireHeldSlots` job via Redis.
- Validation: Form Requests; `ends_at > starts_at`; timezone stored UTC, displayed in therapist tz.

## 7. Integrations
- Emits `BookingConfirmed` event → SPEC-004/005 listeners. No direct Stripe/video calls here.

## 8. Admin / Filament
- `BookingResource`: filter status/date, cancel action → `CancelBooking` Action. Read-only slot history.

## 9. Edge Cases
- Delete slot with bookings blocked; DST display; therapist reschedule = cancel + new hold (no silent move).

## 10. Conflicts / Open Questions
- Open: refund/no-show fee — intent L39. Assumed 24h cutoff, no fee, pending decision.

## 11. Test Plan
- `tests/Feature/AvailabilityTest.php`: overlap 422, past 422.
- `tests/Feature/ConcurrentBookingTest.php`: 1 win + 1 409.
- `tests/Feature/CancellationTest.php`: pre/post cutoff, ICS assertions.
