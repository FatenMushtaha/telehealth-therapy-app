# PLAN-001: Telehealth Therapy Marketplace — MVP Build
- Status: draft
- Source intent: `intent/telehealth-therapy-app.md` (approved)
- Specs: `docs/ai-sdlc/specs/SPEC-001-directory-discovery.md` through `SPEC-005-video-notifications.md`
- Created: 2026-10-07
- Stack: Laravel v12 PHP 8.2, MySQL `telehealth_therapy_app`, Redis queue/cache/session, Filament v5+, Vite + Tailwind v4

Current skeleton: `routes/web.php` single `/`, `app/Models/User.php` only, `bootstrap/app.php` default routing.

## Architecture (mandatory, AGENTS.md)
- Fat models / thin controllers (<50 lines, no SQL/business rules inline).
- One use-case per `app/Actions/<Domain>/::execute()`; orchestration in `app/Services/`.
- Filament presentation only — delegates to Actions/Services.
- MySQL-compatible migrations; tests use sqlite `:memory:` per `phpunit.xml`; Pint style.

## Phase 0 — Foundations (blocks all)
1. Align `.env` to `.env.example` (MySQL 127.0.0.1:3306, Redis 6379, `QUEUE_CONNECTION=redis`, `CACHE_STORE=redis`, `SESSION_DRIVER=redis`). Require MySQL + Redis up before `composer setup`.
2. `composer require filament/filament` + `php artisan filament:install --panels`; add Stripe SDK (`stripe/stripe-php` or Cashier — record decision); choose SMS driver (TBD, SPEC-005 open).
3. Auth + roles: migration `add_role_to_users`, `role` middleware alias in `bootstrap/app.php`, Policies `TherapistProfilePolicy`, `BookingPolicy`, `PaymentPolicy`.
4. Scaffold `app/Actions/`, `app/Services/`, `app/Policies/`, shared Form Requests, `booking_status_changes` + `therapist_verification_audits` audit tables.
- Verify: `php artisan migrate`, `php artisan test --compact`, `./vendor/bin/pint --test`.

## Phase 1 — SPEC-002 Onboarding (unblocks directory)
- Migration `therapist_profiles` + audits; Model `TherapistProfile` (casts via `casts()`, scope `approved()`); factories/seeders.
- Actions `RegisterTherapist`, `ApproveTherapist`, `RejectTherapist`.
- Filament `TherapistProfileResource` (approve/reject bulk → Actions) + pending-count widget.
- Tests: `TherapistApprovalTest` (approve+audit, 403 non-admin, resubmit).

## Phase 2 — SPEC-001 Directory
- Actions `SearchTherapists`, `ViewTherapistProfile`; Redis 60s cache + event invalidation.
- Blade `directory/index`, `directory/show` (Tailwind, next-10 slots, no license docs).
- Tests: `DirectoryTest` (pending hidden, filters, cache, no docs leak, next-10).

## Phase 3 — SPEC-003 Availability + Booking (core risk)
- Migrations `availability_slots` (`UNIQUE(therapist_id,starts_at)`, `INDEX(status,starts_at)`), `bookings` (`slot_id UNIQUE`, price snapshot), `booking_status_changes`.
- Actions `CreateAvailability`, `CreateBooking` (transaction + `FOR UPDATE` + 409 on conflict), `CancelBooking`, `GenerateIcs`; job `ExpireHeldSlots` (15m).
- Thin controllers + Form Requests + `BookingPolicy`; Filament `BookingResource` (cancel → Action).
- Tests: `AvailabilityTest`, `ConcurrentBookingTest`, `CancellationTest`.

## Phase 4 — SPEC-004 Stripe (depends P3)
- Migrations `payments` (`booking_id UNIQUE`, `stripe_session_id UNIQUE`), `payout_logs` (CSV only).
- Actions `CreateCheckout`, `HandleStripeWebhook` (sig verify, dedupe), `RefundPayment`; webhook route; manual payout export.
- Filament `PaymentResource` read-only + refund button.
- Tests: `StripeWebhookTest` (double delivery idempotent, bad sig 400, refund-once).

## Phase 5 — SPEC-005 Video + Reminders (depends P3/P4)
- `app/Services/VideoLinkService::generate()` provider-agnostic (Zoom/Daily.co TBD); store `bookings.video_url`.
- Event `BookingConfirmed` → `SendBookingReminders` (T-24h/T-1h email, T-1h SMS, dedupe key `booking_id+type`); `log` driver in tests.
- Join route gated T-15m→end + Policy 403 otherwise.
- Tests: `VideoJoinTest`, `ReminderTest`.

## Phase 6 — Hardening
- Eager loads, indexes, p95 <300ms check, audit completeness, ICS asserts, `composer test`, `./vendor/bin/pint` fix.
- Document PHI risk: standard-auth MVP, no HIPAA/BAA/recording — needs sign-off (intent §4 L21).

## Order
`P0 → P1 → P2 → P3 → P4+P5 (parallel) → P6`. Do not start P4/P5 before REQ-04 atomic booking passes.

## Open decisions
1. Region (Stripe support). 2. Video provider + recording. 3. SMS provider. 4. 24h cutoff/fees. 5. PHI sign-off.
