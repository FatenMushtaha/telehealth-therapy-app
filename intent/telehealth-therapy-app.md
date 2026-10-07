# Intent: Telehealth Therapy Marketplace — MVP

Author voice: Two-sided marketplace for adults seeking individual therapy and licensed therapists. Patients search directory to session. Therapists onboard to booking. Responsive web first. Standard auth for MVP. Stripe + video integration. Core MVP first, matching/messaging/admin in next phase.

## 1. Core Value Prop & Target Audience
- Problem: Adults struggle to find, book, and attend therapy online; therapists lack simple scheduling/client tools.
- Solution: Integrated marketplace with directory, booking, payment, video.
- Initial users: Adults seeking individual therapy; licensed psychologists/counselors/social workers.

## 2. Core Workflows
- Patient: Search directory -> view profiles -> book/pay -> join video -> follow-up.
- Therapist MVP: Onboard, license verify (manual for MVP), set availability/rates, accept/manage bookings.
- Deferred to Phase 2: guided matching quiz, in-app messaging, homework, admin review.

## 3. Platforms & Tech Stack
- Responsive web only for MVP.
- Stack per AGENTS.md: Laravel v12 (PHP 8.2), MySQL, Redis (queue/cache/session), Filament v5+ back-office, Vite + Tailwind v4.

## 4. Non-Functional Requirements
- MVP: Standard Laravel auth (Breeze/Fortify pattern), roles (patient/therapist/admin), HTTPS, validation/authorization.
- Explicitly out of MVP: formal HIPAA attestation. Risk noted — must add encryption at rest, audit logs, consent, retention before handling real PHI at scale.
- Performance: <300ms server p95 for directory/booking pages (Redis cache), queue video-link + reminder jobs.

## 5. Key Integrations
- Stripe (payments + payouts via Connect or manual payout log for MVP).
- Video: Zoom or Daily.co embedded links.
- Notifications: email + SMS reminders (queue via Redis).
- No native calendar sync in MVP (ICS download only).

## 6. Constraints & Scope
- MVP must-have: directory, profiles, availability, booking, Stripe pay, video join, therapist onboarding, Filament admin.
- Phase 2: guided matching, messaging, notes templates, ratings, payouts automation, calendar sync.
- Timeline model: Core MVP fast (6-8 weeks), phased extras after.

## Open questions
- Launch country/region? Affects payment (Stripe availability) and health-data law.
- License verification: manual admin check in Filament sufficient for MVP?
- Video provider choice: Zoom vs Daily.co? Recording needed? (impacts compliance)
- Refund/cancellation policy and no-show handling?
- Confirm acceptance of PHI risk with standard-auth-only MVP, or add HIPAA-minimum (encryption + audit + BAA)?

Status: approved
