# SPEC Template (docs/ai-sdlc/specs/_template.md)

Use for every `SPEC-###.md`. Every acceptance criterion must be verifiable by an automated test.

```markdown
# SPEC-###: <Title>
- Status: draft | approved | superseded
- Source intent: `intent/<slug>.md` (Status: <draft|approved>)
- Created: YYYY-MM-DD

## 1. Scope
- In:
- Out (explicit non-goals):

## 2. Roles & Permissions
| Action | Role... |

## 3. Domain Model
- Table/fields, invariants, indexes.

## 4. Requirements
| ID | Requirement (testable, unambiguous) | Intent trace (section/line) |
|----|-------------------------------------|------------------------------|

## 5. Acceptance Criteria (Gherkin, one per critical path)
```gherkin
Feature: ...
  Scenario: ...
    Given ...
    When ...
    Then ...
```

## 6. Non-Functional Requirements
- Auth, validation, performance, queue, audit, security.

## 7. Integrations
- Contract, idempotency, failure mode.

## 8. Admin / Filament
- Resources, actions (presentation only, delegate to Actions/Services).

## 9. Edge Cases

## 10. Conflicts / Open Questions
- Conflict: ... (surface, do not silently resolve)

## 11. Test Plan
- Feature/unit tests mapping REQ IDs -> test file/method, data setup, assertions.
```
```

Rules:
- Trace every requirement to intent line/section.
- AC must state Given/When/Then with observable outcome.
- No business logic in controllers/Filament — must reference `app/Actions/` + `app/Services/`.
- Migrations MySQL-compatible. Tests use sqlite `:memory:` per `phpunit.xml`.
