# SPEC-001: Directory & Therapist Discovery
- Status: approved
- Source intent: `intent/telehealth-therapy-app.md` (Status: approved)
- Created: 2026-10-07
- Supersedes: monolith `SPEC-001-telehealth-marketplace-mvp.md` §REQ-01/02 (now split into SPEC-001..005)

## 1. Scope
- In (intent §1 L8, §2 L11, §6 L31):
  - Public directory + therapist profiles, filters, profile detail with next open slots
  - Responsive web only (intent §3 L16)
- Out:
  - Booking/checkout, availability editing, onboarding approval, video, reminders, matching quiz, messaging, ratings

## 2. Roles & Permissions
| Action | Patient | Therapist | Admin |
|---|---|---|---|
| Browse directory/profiles | Y | Y | Y |

Guest (unauthenticated) may browse directory; booking requires auth (covered SPEC-003).

## 3. Domain Model
- `users{id, name, email UNIQUE, role, phone NULL}` — intent §4 L20.
- `therapist_profiles{user_id PK/FK, headline, bio TEXT, specialties JSON, verification_status:pending|approved|rejected, rate_cents UNSIGNED, currency, timezone}` — intent §2 L12. Index `(verification_status)`.
- Read dependency on `availability_slots{therapist_id FK, starts_at UTC, ends_at UTC, status}` for next-slots display (owned by SPEC-003).

Invariants: only `approved` listed; license docs never exposed to patients.

## 4. Requirements
| ID | Requirement | Intent trace |
|----|-------------|--------------|
| REQ-01 | Directory lists only `approved` therapists with ≥1 future `open` slot; filters specialty, max rate, availability window; cached 60s Redis | §1 L8, §2 L11, §4 L22 |
| REQ-02 | Profile shows bio, specialties, rate, next 10 open slots; never exposes license docs | §6 L31 |

## 5. Acceptance Criteria (Gherkin)
```gherkin
Feature: REQ-01 directory gate
  Scenario: pending therapist hidden
    Given therapist with verification_status pending and open slots
    When patient searches directory
    Then therapist is excluded
    And approved therapists with open slots are listed

Feature: REQ-01 filters
  Scenario: filter by specialty and rate
    Given approved therapists with varying specialties and rates
    When I filter specialty=anxiety and max_rate=10000
    Then only matching profiles are returned

Feature: REQ-02 profile slots
  Scenario: next slots shown
    Given approved therapist with 15 future open slots
    When I view profile
    Then I see next 10 ordered by starts_at
    And no license document URL is present
```

## 6. Non-Functional Requirements
- Perf p95 <300ms via Redis cache + `with()` eager loads; indexes `(verification_status)`, `availability_slots(status,starts_at)` (intent §4 L22).
- Validation: filter inputs via Form Request; unknown specialty ignored, not 500.
- Auth: public read; no PII beyond profile.

## 7. Integrations
- None. Reads Redis cache; no Stripe/video.

## 8. Admin / Filament
- None in this spec (approval UI in SPEC-002). Directory is Blade + Tailwind, calls `app/Actions/Directory/*`, never queries inline.

## 9. Edge Cases
- No open slots → excluded from directory but profile directly accessible with empty state.
- Cache invalidation on approval/slot change (event → cache forget).

## 10. Conflicts / Open Questions
- Open: launch region affects specialty taxonomy — intent L36.

## 11. Test Plan
- `tests/Feature/DirectoryTest.php`: pending hidden, rejected hidden, filters, cache hit, license docs absent, next-10 limit.
- Env sqlite `:memory:`; `php artisan test --compact`.
