# SPEC-004: Payments with Stripe
- Status: approved
- Source intent: `intent/telehealth-therapy-app.md` (Status: approved)
- Created: 2026-10-07
- Supersedes: monolith `SPEC-001-telehealth-marketplace-mvp.md` §REQ-05 (+ REQ-08 refund half)

## 1. Scope
- In (intent §5 L25):
  - Stripe Checkout per booking, webhooks, idempotency, manual payout log (no Connect auto-payout in MVP)
  - Listens to SPEC-003 `BookingConfirmed` / `BookingCancelled`
- Out: Connect auto-payouts, coupons, insurance (Phase 2)

## 2. Roles & Permissions
| Action | Patient | Therapist | Admin |
|---|---|---|---|
| Pay own booking | Y own | — | — |
| View payments | Y own | Y assigned | Y all read |

## 3. Domain Model
- `payments{id, booking_id FK UNIQUE, stripe_session_id UNIQUE, stripe_payment_intent UNIQUE NULL, amount_cents, status:pending|paid|refunded|failed}`.
- Manual `payout_logs{id, therapist_id, period, amount_cents, status}` (CSV export only).

Invariant: amount = booking `price_cents` snapshot.

## 4. Requirements
| ID | Requirement | Intent trace |
|----|-------------|--------------|
| REQ-05 | Checkout Session per booking; webhooks `checkout.session.completed` / `payment_intent.*` idempotent on `stripe_session_id`; bad signature → 400 no state change | §5 L25 |
| REQ-08P | Cancel pre-cutoff with paid payment → flag `refund-pending`; Stripe refund executed once via idempotency key | Open Q L39 |

## 5. Acceptance Criteria (Gherkin)
```gherkin
Feature: REQ-05 Stripe idempotency
  Scenario: duplicate webhook delivery
    Given booking pending with session sess_123
    When checkout.session.completed sess_123 is delivered twice
    Then booking becomes confirmed once
    And no duplicate payment row is created

Feature: REQ-05 invalid webhook
  Scenario: bad signature rejected
    When webhook with invalid signature arrives
    Then response is 400 and no state changes

Feature: REQ-08P refund once
  Scenario: pre-cutoff cancel refunds once
    Given paid booking cancelled pre-cutoff
    When refund webhook delivered twice
    Then payment is refunded once
```

## 6. Non-Functional Requirements
- Webhook: signature verify, store raw event id for dedupe, return 200 fast + queue heavy work (Redis).
- Secrets via `.env` (`STRIPE_*`), never logged. Amounts in cents (unsigned).

## 7. Integrations
- Stripe: Checkout Session create in `app/Actions/Payments/CreateCheckout.php`; webhook handler delegates to `ConfirmBooking` / `MarkPaymentFailed`; manual payout CSV.

## 8. Admin / Filament
- `PaymentResource`: read-only + refund button → `RefundPayment` Action; filter status/date.

## 9. Edge Cases
- Success-after-cancel → auto-refund flag; out-of-order events guarded by status machine; partial refunds out of scope.

## 10. Conflicts / Open Questions
- Open: region affects Stripe availability — intent L36. Assumed Stripe-supported region.

## 11. Test Plan
- `tests/Feature/StripeWebhookTest.php`: double delivery idempotent, bad sig 400, refund-once. Use fake webhook payloads, `log` driver.
