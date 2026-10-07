# SPEC-002: Therapist Onboarding & Verification
- Status: approved
- Source intent: `intent/telehealth-therapy-app.md` (Status: approved)
- Created: 2026-10-07
- Supersedes: monolith `SPEC-001-telehealth-marketplace-mvp.md` §REQ-09

## 1. Scope
- In (intent §2 L12, §6 L31):
  - Therapist registration, license submission, manual admin approve/reject in Filament with audit
  - Post-approval: therapist becomes eligible for directory (SPEC-001) and scheduling (SPEC-003)
- Out: availability editing (SPEC-003), payments, video, auto-verification APIs

## 2. Roles & Permissions
| Action | Patient | Therapist | Admin |
|---|---|---|---|
| Submit own onboarding/license | — | Y own | — |
| Approve/reject therapists | — | — | Y |

Policies: `TherapistProfilePolicy::approve` admin-only.

## 3. Domain Model
- `therapist_profiles{user_id PK/FK, headline, bio, specialties JSON, license_number, license_state, verification_status:pending|approved|rejected DEFAULT pending, rate_cents, currency, timezone}`.
- `therapist_verification_audits{id, therapist_id FK, actor_id FK, from, to, created_at}` append-only.

Invariant: status transitions only via `ApproveTherapist` / `RejectTherapist` Actions.

## 4. Requirements
| ID | Requirement | Intent trace |
|----|-------------|--------------|
| REQ-09 | Onboarding submits license_number + state + docs ref; admin approve/reject in Filament writes audit row with actor + timestamp | §2 L12 |

## 5. Acceptance Criteria (Gherkin)
```gherkin
Feature: REQ-09 approval audit
  Scenario: admin approves therapist
    Given pending therapist
    When admin approves in Filament action
    Then status is approved and audit row records actor and timestamp

Feature: REQ-09 authorization
  Scenario: non-admin cannot approve
    Given pending therapist
    When patient or therapist calls approve
    Then response is 403 and status unchanged
```

## 6. Non-Functional Requirements
- Validation: Form Requests; license_number required, rate_cents >0, timezone valid PHP tz.
- Audit append-only; no deletes.
- Standard-auth MVP, not HIPAA (intent §4 L21) — document risk.

## 7. Integrations
- None. Doc storage ref as string path only (no PHI scan in MVP).

## 8. Admin / Filament
- `TherapistProfileResource`: table filter by status, bulk approve/reject → delegates to `app/Actions/Therapists/ApproveTherapist.php` / `RejectTherapist.php`. Widgets: pending approvals count. Presentation only.

## 9. Edge Cases
- Re-submit after reject → back to pending with new audit row; double-approve idempotent.

## 10. Conflicts / Open Questions
- Open: manual check sufficient? intent L37 — assumed yes for MVP.

## 11. Test Plan
- `tests/Feature/TherapistApprovalTest.php`: admin approve/reject + audit, 403 for non-admin, resubmit flow.
