# Project architecture scan — output template

The **explore subagent** must return its findings using this template. The parent agent uses it to plan Payunit file placement and patterns.

```markdown
# Project architecture scan — [project name]

## Stack summary
- **Type:** mobile | web | backend | fullstack | monorepo
- **Languages / frameworks:** …
- **Package manager:** …

## Repository layout
- **Root structure:** (top-level dirs and their role)
- **Architecture style:** feature-first | layered | clean/hexagonal | MVC | modular monolith | microservices | other
- **Source root:** e.g. `lib/`, `src/`, `apps/api/`

## Client (if applicable)
- **Framework:** Flutter / React / Vue / …
- **State management:** …
- **Routing:** …
- **HTTP client:** …
- **DI / service locator:** …
- **Reference feature to mirror:** (path of a similar feature, e.g. auth or orders)

## Server (if applicable)
- **Framework:** Express / Nest / FastAPI / Laravel / …
- **API style:** REST | GraphQL
- **Route registration pattern:** …
- **ORM / DB:** …
- **Reference module to mirror:** (path of similar module)

## Configuration & secrets
- **Env loading:** .env / dart-define / vault / …
- **Existing API base URL config:** (file path)
- **Never commit:** (patterns already in .gitignore)

## Existing payment / order domain
- **Payment code exists:** yes | no
- **Paths:** …
- **Order / transaction models:** …
- **Webhook routes:** …
- **PaymentGateway port exists:** yes | no
- **Port / adapter paths:** …
- **Active PAYMENT_PROVIDER:** …

## Recommended Payunit placement
| Layer | Suggested path(s) | Mirror existing |
|-------|-------------------|-----------------|
| Port (`PaymentGateway`) | … (reuse if exists) | … |
| Config | … | … |
| PayunitAdapter | … | … |
| Use-case (`applyPaymentEvent`) | … (reuse if exists) | … |
| Webhook `POST /webhooks/payments/payunit` | … | … |
| UI / controller | … (must stay PSP-agnostic) | … |

If port exists: **adapter only** — do not rewrite checkout or fulfillment.

## Integration constraints
- Client-only or backend present in repo: …
- Must use backend proxy: yes | no (justify)
- Test setup: …

## Open questions for user
- (only if blocking — e.g. no backend in repo but production needed)
```

Rules for the subagent:
- Cite **real file paths** from the repo, not guesses.
- Pick one **reference feature** whose structure Payunit code should copy.
- If monorepo, scan the app/package the user is working on.
- Do **not** implement Payunit — scan and recommend only.
