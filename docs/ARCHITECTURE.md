# LEVACY LEGAL AI — Architecture Package (Spec §73 First Response)

Status: **PROPOSAL — awaiting approval. No application code has been written.**
Scope: sections A–J of the Build Master Specification. Phase 1 will not start until this is approved.

Core principle (governs every design choice below): *AI identifies, extracts, searches, compares, summarizes, calculates, drafts, flags. The attorney decides.*

---

## A. PRODUCT ARCHITECTURE

### Module map

| Layer | Modules |
|---|---|
| **Shell** | Collapsible sidebar, Home prompt, universal search (⌘K), notifications, role-aware nav |
| **Matter core** | Matters, Clients/Contacts/Entities, Matter Workspaces (11 tabs), Matter Memory, Conflict Checks, Tasks, Calendar, Deadlines |
| **Documents** | Repository, versions, folders, tags, processing pipeline, viewer with citation deep-links, Suggested Matter Updates |
| **AI** | Gateway (provider-agnostic), Chat, RAG retrieval, document actions (§8), drafting, Explainability panel, AI audit |
| **Research** | Florida Research Mode, authority store, citation verification, law-version tracking |
| **Practice modules** | Real Estate (closing dashboard, title, deed QC, recording, tax calc, land trust), Probate, Estate Planning, Business/LLC + ownership charts, Contract/Lease review, Litigation |
| **Governance** | Attorney review/approval workflow, Florida Compliance dashboard, AI provider registry, client AI consent, security incidents, retention |
| **Admin** | Users, roles, permissions, matter types, categories, templates, firm knowledge, integrations, audit viewer, settings |
| **Integrations** | Adapter framework; every adapter reports `NOT_CONFIGURED` until real credentials exist |

### Architectural style

- **Modular monolith** (single Next.js deployment + a separate worker process). Not microservices — the team is small and transactional integrity across matter/doc/audit matters more than independent scaling.
- **Domain modules** under `src/modules/<domain>` each owning: schema, repository (DB access), service (business rules + authorization), API handlers, UI components. UI never touches the DB directly.
- **Everything that matters is enforced server-side and again in the database.** The browser is untrusted.
- **Human-in-the-loop is a first-class data model**, not UI decoration: `approvals`, `suggested_updates`, and `verification_status` columns gate state transitions.

### Stack (per spec §56)

- Next.js (App Router) + React + TypeScript (strict) + Tailwind CSS; own design tokens (no borrowed branding)
- PostgreSQL via **Supabase** (Auth, Postgres, Storage, RLS, pgvector) — see risk R-3 for the BAA/enterprise-terms question
- Background jobs: **pg-boss** (Postgres-backed queue, no extra infra) running in a worker container
- Validation: Zod at every boundary; ORM/query: Drizzle or Kysely (typed SQL, migrations in repo)
- Tests: Vitest (unit/integration), Playwright (e2e), pgTAP (RLS policy tests)

---

## B. DATABASE SCHEMA

All tables: `id uuid pk`, `firm_id uuid not null` (multi-tenant-ready), `created_at`, `updated_at`, soft-delete `deleted_at` where retention rules apply. Every matter-scoped table carries `matter_id not null`.

### Identity & access
- `users` (links to auth.users; name, status, mfa_enrolled)
- `roles` (Administrator, Managing Attorney, Attorney, Paralegal, Legal Assistant, Accounting, Read Only)
- `permissions` (key e.g. `document.download`, `matter.close`, `billing.view`) ; `role_permissions`
- `user_roles` (firm-level role assignment)
- `matter_users` (user ↔ matter; `access_level`: none/view/edit/manage; `is_responsible_attorney`; `ethical_wall boolean`)
- `sessions_meta` (device, last_seen, revoked_at)

### Parties
- `clients`, `contacts`, `entities` (name, type, state, doc number, status), `entity_relationships` (parent, child, role, ownership_pct, effective_from/to), `matter_contacts`, `matter_entities`

### Matters
- `matters` (number, client_id, name, description, type, status, responsible_attorney_id, billing_type, property_address, county, court, case_number, opened_at, closed_at)
- `matter_types`, `matter_statuses` (admin-configurable lookup, seeded from spec)
- `related_matters`, `workspaces` (1:1 with matter), `notes` (matter_id, author, `attorney_approved`)
- `matter_memory_items` (matter_id, key, value, source_document_id, source_locator, status: suggested/approved/rejected, approved_by)
- `suggested_updates` (matter_id, field, current_value, proposed_value, source citation, status, decided_by) — **document intelligence never writes directly to matters**

### Documents
- `documents` (matter_id nullable only for firm-level docs, category, status, confidentiality, current_version_id)
- `document_versions` (document_id, version_no, storage_key, sha256, mime, size, uploaded_by, status, supersedes_version_id, is_approved_version)
- `document_processing` (version_id, state: uploading/processing/indexed/ready/failed, error, ocr_used)
- `document_pages` (version_id, page_no, text, bbox_json) — enables page-level citations
- `document_chunks` (**matter_id**, version_id, page_start/end, section_label, text, tsvector, embedding vector)
- `document_metadata` (extracted key/value + confidence + locator)
- `document_tags`, `folders`, `document_access_log`

### Work management
- `tasks`, `task_documents`, `deadlines` (event, trigger_date, calculated_date, rule_authority, method, day_type, holiday_adjust, verified_by, verified_at, `ai_generated`), `calendar_events`

### AI / research
- `chats` (matter_id nullable, owner, title, archived, mode), `messages` (role, content, citations_json)
- `ai_runs` (full audit per §47: prompt, user, matter, provider_id, model, document_ids, knowledge_ids, external_source_ids, response, citations, approval_status, tokens, latency, transmission_decision)
- `research_memos`, `authorities` (citation, type, jurisdiction, court, decision_date, official_url, text_hash, effective_date, amendment_date, retrieved_at, version_label), `authority_verifications` (authority_id, checks_json, status: VERIFIED/REQUIRES_VERIFICATION/SUPERSEDED/UNAVAILABLE, law_checked_as_of, verifier), `citations` (message_or_memo → authority or document+page+locator)

### Governance
- `approvals` (subject_type, subject_id, from_status, to_status, requested_by, decided_by, decision, notes) — generic attorney-review workflow
- `conflict_checks` (matter_id, status, search_terms, results_json, reviewer, attorney_decision, notes)
- `ai_providers` (name, model, api_type, retention/sharing/training policy, security terms, contract notes, storage location, deletion procedure, approval_status, last_reviewed, enabled)
- `client_ai_consents` (client_id, matter_id, status, date, method, scope, restrictions, notes); `firm_ai_policy` (requires_consent_before_external_transmission boolean, by matter type)
- `compliance_sources` (type: statute/constitution/rule/bar rule/admin code/county/policy; citation; status: current/needs_review/superseded/pending_attorney_review; last_verified_at; verified_by)
- `security_incidents` (+ `incident_events`), `retention_policies`, `legal_holds`
- `audit_logs` (append-only, hash-chained: `prev_hash`, `row_hash`)

### Knowledge & templates
- `templates` (category, version, status: draft/attorney_approved/archived, approved_by, approved_at), `template_versions`
- `firm_knowledge` (type, version, status, approved_by, `supersedes_id`), `firm_knowledge_chunks` (firm-level, embedding) — **separate table from `document_chunks` by design** (two knowledge layers, §5)

### Florida / real estate / practice modules
- `county_requirements` (67 rows seeded with county names only; all other fields NULL until verified — never guessed), `rate_tables` (kind, rate, basis, authority, effective_from/to, county nullable, verified_at, verified_by) — **versioned, never hard-coded**
- `calculations` (inputs, rate_table_id, result, shown_calculation, run_by)
- `closing_items` (matter_id, item_type, status, responsible, due, completed_at, document_id)
- `title_reviews` / `deed_reviews` (extracted fields + comparison results + flags)
- `land_trusts`, `land_trust_instruments`
- `probate_cases` (+ `probate_assets`, `probate_creditors`), `estate_plans` (+ `execution_checklists`)
- `business_filings` (status: draft/signed/ready_to_file/filed/accepted/active), `ownership_edges` (see entity_relationships)
- `litigation_cases` (+ `pleadings`, `discovery_items`, `hearings`, `evidence_items`)
- `integrations` (type, state: NOT_CONFIGURED/CONNECTED/ERROR, encrypted credential ref), `integration_events`

### Key relationships
```
clients 1─* matters 1─1 workspaces
matters *─* users (matter_users)        ← the permission pivot for RLS
matters 1─* documents 1─* document_versions 1─* document_chunks(matter_id)
matters 1─* tasks / deadlines / notes / chats / closing_items / conflict_checks
ai_runs → ai_providers ; ai_runs → documents/knowledge/authorities (junction tables)
authorities 1─* authority_verifications
```

---

## C. SECURITY MODEL

### Authentication
- Supabase Auth, email+password with **mandatory MFA (TOTP/WebAuthn)** for attorneys/admins; configurable for others. No self-signup — invite only.
- Short-lived access tokens, rotating refresh tokens, idle timeout (default 30 min) and absolute timeout configurable; session list + remote revoke.

### Authorization — defense in depth (three gates)
1. **Route/middleware gate**: authenticated + role has the permission key.
2. **Service gate**: every service method calls `authorize(user, action, resource)`; matter access requires a `matter_users` row at sufficient level. Role grants *capability*; `matter_users` grants *access to a specific matter*. Both are required (least privilege). Ethical-wall flag overrides everything including Administrator for matter content.
3. **Database gate (Postgres RLS)**: policies on **every** matter-scoped table, including `document_chunks`, `messages`, `ai_runs`, `deadlines`, and storage objects:
   ```
   USING ( firm_id = auth.firm_id()
     AND EXISTS (SELECT 1 FROM matter_users mu
                 WHERE mu.matter_id = <table>.matter_id
                   AND mu.user_id = auth.uid() AND mu.access_level <> 'none') )
   ```
   The app DB role is non-superuser, `BYPASSRLS` is **never** granted to anything request-serving. The worker uses a separate role and sets `app.matter_id` / `app.job_user` per job; it cannot read across matters in one query.

### Matter isolation (spec §5, §58, §65)
- `matter_id` is `NOT NULL` + FK on every chunk, embedding, note, task, chat, generated doc, AI run.
- Vector search is **always** a filtered query: `WHERE matter_id = ANY($authorized_ids)` executed under RLS — filtering is not optional in the retrieval function signature (`retrieve(ctx: AuthorizedContext, …)`; `AuthorizedContext` can only be constructed by the authorization module).
- Cross-matter search is an explicit, separate action requiring a user choice **and** permission on every matter touched; logged as such.
- Direct URL/API/storage access: signed, short-TTL URLs generated only after an authz check; storage paths include `matter_id` and bucket policies mirror RLS.
- Search index (Postgres FTS) is the same tables → same RLS. No external search engine in v1, avoiding a second isolation boundary.

### Documents
- Encrypted at rest (platform + application-level envelope encryption for storage objects as option), TLS 1.2+ everywhere.
- Upload: size limit, MIME sniffing (not extension), allow-list, **malware scan (ClamAV) in the worker before a version becomes `Ready`**; files quarantined on failure.
- Download requires `document.download` permission; every view/download writes `document_access_log` + `audit_logs`.
- Deletion = soft delete → retention workflow → authorized secure purge (never silent hard delete; legal holds block purge).

### Secrets
- Env vars only via host secret manager; never in repo; `.env.example` has names only. `gitleaks` pre-commit + CI. AI provider keys live **server-side only**; there is no code path that sends a provider key or provider call from the browser. Integration credentials stored encrypted (KMS-wrapped), never returned by any API.

### Audit
- `audit_logs` is append-only (no UPDATE/DELETE grants; trigger blocks it), hash-chained for tamper evidence, with periodic anchor of the latest hash to an external store. Honest note: "tamper-*resistant*," not tamper-proof — a DB superuser could still rewrite; the anchor makes that detectable.

### Other controls
Rate limiting (per user/IP/route, stricter on AI + auth), CSRF protection, strict CSP, secure cookies, input validation (Zod), output encoding, dependency scanning, structured logs with **no document text or client PII in application logs**, backups (PITR + encrypted, tested restores), documented DR plan (RPO/RTO targets set with the firm).

---

## D. AI ARCHITECTURE

### Provider-agnostic gateway
`AIGateway.complete(ctx, request)` is the *only* way any module calls a model. Before any call it runs the **Transmission Policy Check**:

1. Is the provider `enabled` **and** `approved` in `ai_providers`? If not → blocked, "AI provider unavailable / not approved."
2. Does the matter's client require consent per `firm_ai_policy`? If yes and no `client_ai_consents` record with sufficient scope → **blocked** (not warned), message shown, attempt logged.
3. Is the user's role permitted AI access (`ai.use`) and matter access?
4. Log `external_data_transmission` to audit with what categories of content were sent.

Adapters (Anthropic, OpenAI, Azure OpenAI, Bedrock, local) implement one interface. Day-one: only providers an admin has approved in the registry; none are pre-approved.

### Matter-aware RAG pipeline (spec §57)
```
request → resolve active matter (from route, not from model text)
        → authorize (user × matter × AI permission)           [fail closed]
        → transmission policy check
        → query understanding (intent: factual / extract / draft / research)
        → retrieval (all filtered by matter_id under RLS):
             a. matter metadata + approved memory items
             b. hybrid search over document_chunks (FTS + vector, RRF merge)
             c. version filter: prefer latest approved/executed version; superseded excluded unless asked
             d. firm_knowledge_chunks (approved + current only) when relevant
             e. external authority only if Research Mode (→ section E)
        → context assembly with source hierarchy (§49), each chunk tagged with doc/page/section ids
        → conflict pre-pass: same field, different values across docs → surfaced as CONFLICT DETECTED (§50), never silently resolved
        → generate (structured output: answer + claims[] each pointing to chunk ids)
        → citation resolver: every claim's chunk id must exist in the *retrieved set*; claims without a valid source are rendered as unsourced or dropped
        → ai_runs audit record
        → render with Explainability panel (ANSWER / WHY / SOURCES USED / ISSUES / MISSING / ATTORNEY REVIEW)
```
Citations are generated by the **system from retrieval metadata**, not typed by the model, so a document citation cannot be invented. If retrieval returns nothing: "Information not available."

### Prompt-injection posture
Document text and web content are treated as **data, never instructions**: wrapped in delimited, labeled blocks; the model has **no tools that write** (no send-email, no update-matter, no file/URL fetch) in chat; side effects only via explicit user-clicked actions. Retrieval output can't widen scope (matter ids come from the server context, not from model output). Adversarial test corpus in Phase 9.

### Two knowledge layers
`document_chunks`/`matter_memory_items` (matter-scoped, RLS by matter) vs `firm_knowledge_chunks` (firm-scoped, approved-only, versioned). They are never in one index and never merged before the context assembler labels them.

---

## E. LEGAL RESEARCH ARCHITECTURE

### Principle
The model **never is the source of an authority**. Authorities enter the system only by retrieval from a source adapter, and are stored in `authorities` with provenance. The model may *discuss* retrieved text; it may not *introduce* a citation.

### Source adapters (priority order per §11)
| Priority | Source | Candidate access path | Notes |
|---|---|---|---|
| 1–2 | Fla. Constitution, Fla. Statutes | Official Florida Legislature site (leg.state.fl.us / Online Sunshine) | Statute year/version handling is essential (§14) |
| 3–4 | Fla. Supreme Court, DCAs | Official court opinion pages; CourtListener API (free, not official) | Official PDFs preferred as the "verified" text |
| 5 | Court rules | Florida Supreme Court / Florida Bar rule publications | |
| 6 | Fla. Admin. Code | Florida Dept. of State (FAC/FAR) | |
| 7–9 | Agency, federal, county/municipal | Agency sites, govinfo/eCFR, Municode (varies) | Per-county; no statewide assumption |
| 10 | Secondary | Clearly tagged SECONDARY, never "verified authority" | |

**Important limitation (see R-6):** reliable "is this still good law?" (KeyCite/Shepard's-style) is a commercial product. Free sources can confirm *existence, text, and quoted language*, but not subsequent negative treatment. Without a licensed citator, status is shown as **"Subsequent history not checked"** — never "good law."

### Verification engine (spec §13) — deterministic code, not LLM judgment
For each authority a candidate response cites:
1. **Existence** — resolved from a source adapter by citation (exact lookup).
2. **Citation match** — reporter/court/year/party names agree with the retrieved record.
3. **Jurisdiction** — recorded and checked against the question.
4. **Quote check** — every quoted string must appear verbatim (normalized whitespace) in retrieved text; otherwise the quote is stripped/flagged.
5. **Proposition support** — LLM-assisted *entailment check against the retrieved passage*, marked as **advisory**; an authority is only `VERIFIED` for the proposition if the passage supports it and an attorney has not overridden. Failure → `AUTHORITY REQUIRES VERIFICATION`.
6. **Status** — checked via a licensed citator/API if configured; otherwise "not checked."
7. **Statute versioning** — selects the version in force on the *transaction/event date* (matter field), records effective/amendment dates; displays `LAW CHECKED AS OF: <date>` only when retrieved from an official source on that date.

State machine: `RETRIEVED → VERIFIED | REQUIRES_VERIFICATION | UNAVAILABLE | SUPERSEDED`. Default for anything not passing all required checks is `REQUIRES_VERIFICATION`. UI renders verified (solid) vs. unverified (hatched, warning label) distinctly; unverified authority cannot be inserted into a draft work-product without an attorney acknowledging it.

### Output
Research memos use the §12 format (Question Presented → … → Attorney Review). Each authority row shows Citation / Court / Date / Proposition / Official link / Current status / Date verified / Binding-Persuasive-Secondary label. Finalizing a memo requires an attorney approval record (§67).

---

## F. APPLICATION ROUTES

```
/login  /mfa  /invite/[token]
/home
/matters                        list, filter, new
/matters/new                    (runs conflict-check gate)
/matters/[id]                   overview
  /documents  /documents/[docId]
  /chat       /chat/[chatId]
  /research   /tasks   /deadlines   /contacts   /notes
  /generated  /financial   /activity
  /closing  /title  /deeds  /land-trust       (real estate, shown by matter type)
  /probate  /estate-plan  /entity  /litigation  /contract-review
  /conflicts
/workspaces                     (cross-matter workspace index)
/assistant                      (AI Assistant; matter optional)
/documents                      (firm-wide repository, permission-filtered)
/knowledge
/research                       Florida Research Mode (+ /research/[memoId])
/templates  /templates/[id]
/tasks  /calendar  /history
/search?q=
/tools/recording-calculator
/admin
  /admin/users  /roles  /permissions  /matter-types  /document-categories
  /admin/templates  /knowledge  /compliance  /ai-providers  /ai-consent
  /admin/integrations  /security  /security/incidents  /audit  /retention  /settings
/api/*   (REST route handlers; see below)
```
API conventions: `/api/v1/<module>/…`; every handler = `authn → authz(permission + matter) → zod parse → service → audit`. Direct URL to an unauthorized matter returns a generic 403/404 page ("You do not have permission to access this matter") with no existence leak.

---

## G. UI COMPONENT STRUCTURE

```
src/
  app/                         routes (thin; compose modules)
  modules/<domain>/            schema · repository · service · api · components
  ui/                          LEVACY design system (original tokens: color, type, spacing, radius, motion)
     primitives/  Button Input Select Dialog Drawer Tabs Tooltip Badge Table Toast …
     patterns/    DataTable (paginated, virtualized) · EmptyState · StatusPill · FormField
  components/
     shell/       AppShell · Sidebar(collapsible) · TopBar · GlobalSearch · NotificationMenu
     ai/          PromptBox · ModeSelector(Research/Review/Drafting/FL Law) · ContextChips(matter, docs)
                  ChatThread · MessageBubble · CitationChip → SourceViewer · ExplainabilityPanel
                  SuggestedUpdatesTray · ConflictBanner
     docs/        UploadDropzone · ProcessingStatus · DocumentViewer(PDF.js, page jump) · VersionHistory · TagEditor
     matter/      MatterHeader · WorkspaceTabs · MatterPicker · MemoryPanel
     legal/       AuthorityCard · VerificationBadge · LawCheckedAsOf · AttorneyReviewBanner
                  ApprovalStepper · DeadlineCard(VERIFY DEADLINE) · RequirementTable · OwnershipChart
     states/      ComingSoon · NotConfigured · RequiresSetup · PermissionDenied · ErrorPanel
```
Rules: any feature not implemented renders `ComingSoon`/`NotConfigured`/`RequiresSetup` — **no dead buttons** (§69). Disclaimer text (§59) lives in one `<AIDisclaimer>` component placed at: AI output footers, research memos, generated documents, calculators — not on every screen.

---

## H. IMPLEMENTATION ROADMAP

Follows §72, with security/testing running continuously rather than only in Phase 9.

| Phase | Deliverable | Exit criteria |
|---|---|---|
| **1 Foundation** | Auth+MFA, users/roles/permissions, matter CRUD, workspaces shell, basic documents (upload/download/list, no AI), RLS, audit foundation, Home | Matter isolation test suite green (Search/API/URL/doc retrieval); audit chain verified |
| **2 Doc intelligence** | Worker queue, text extraction, OCR, page-level storage, FTS, versions, metadata, Suggested Updates, viewer with citation jump | Processing states visible; extraction never overwrites matter data |
| **3 AI** | Gateway, provider registry, consent gate, RAG, memory, firm knowledge, chat, citations, ai_runs | Citation-accuracy tests; no cross-matter retrieval in adversarial tests |
| **4 FL Research** | Source adapters, verification engine, law versioning, memo format | Fabricated-citation red-team passes; unverified never labeled verified |
| **5 Real estate** | Closing dashboard, title/deed review, 67-county table, versioned rate tables + calculators, land trust | Calculations reproduce attorney-supplied worked examples |
| **6 Other practice areas** | Probate, estate planning, LLC/ownership charts, contract/lease, litigation | Per-module attorney review of checklists |
| **7 Operations** | Tasks, calendar, templates, conflict workflow, approvals, compliance dashboard | Approval gates enforced server-side |
| **8 Integrations** | One adapter at a time, only with real credentials | Each shows NOT CONFIGURED until proven |
| **9 Security review** | Pen test, prompt-injection/AI-leak tests, backup restore drill | Findings closed |
| **10 Production** | Deployment, monitoring, IR runbook, training, docs | Firm sign-off |

Each phase ends with a demo + test report and **waits for your approval** (§73).

---

## I. RISKS / QUESTIONS

### Technical
- **R-1 Scanned/poor-quality PDFs**: OCR accuracy limits extraction (legal descriptions, amounts). Mitigation: confidence scores, side-by-side verification, never auto-apply.
- **R-2 Page-level citation precision**: requires keeping page+bbox from extraction; feasible for text PDFs/DOCX (DOCX has no fixed pages—cite section/paragraph).
- **R-3 Supabase suitability**: confirm available plan/terms (BAA/DPA, region, encryption options). Alternative: self-managed Postgres + own auth (Keycloak/Auth.js). Needs your decision.
- **R-4 Large-file performance**: chunked upload, background processing, pagination — designed in, but needs load testing with real volumes.
- **R-5 RLS complexity/perf**: policies on vector queries need index design; test at scale.

### Security
- Prompt injection via uploaded documents/emails (Phase 9 test corpus).
- Insider threat / over-broad admin: ethical walls, no admin bypass of matter content, audit review.
- Hash-chained audit is tamper-*resistant*, not immutable; needs external anchoring.
- Provider-side data handling: depends on contract terms we must record per provider — I cannot assert any provider's policy; admin must enter it from the actual agreement.

### Legal-tech / professional responsibility
- **R-6 No free "good law" check.** Reliable subsequent-history checking needs a licensed citator (Westlaw/Lexis/vLex or similar). Without it the system can verify existence/text/quotes only and will say "status not checked."
- Florida Bar guidance on generative AI (e.g., Rule 4-1.1 competence, 4-1.6 confidentiality, 4-5.3 supervision, and the Bar's published AI ethics opinion) should be reviewed by **the firm's ethics counsel** — I will implement configurable policy controls but will not assert what the rules require. Same for Fla. Stat. §501.171 notification analysis: the incident module tracks and flags; counsel decides.
- Tax/recording rates and county requirements must be **entered and verified by a person from official sources**; seed data will be empty/“Unverified,” not guessed.
- Statute-version logic depends on obtaining historical statute text (official site provides by-year versions; coverage must be confirmed).
- Unauthorized-practice/representation language: product copy reviewed by attorney.

### Missing inputs / third-party services needed
- AI provider account(s) with enterprise/API terms (and the written terms to record)
- Supabase (or alternative) project + region decision
- Email/SMS for invites & MFA recovery (SMTP provider)
- OCR engine choice (self-hosted Tesseract vs cloud OCR — cloud OCR is also an external transmission and goes through the same consent gate)
- Malware scanning (ClamAV self-hosted is fine)
- Licensed citator (optional but strongly recommended), CourtListener key (free)
- Per integration (Clio, Qualia, Simplifile, DocuSign, Google, Microsoft, Notion): developer app registration + customer authorization — **none built until credentials exist**
- Hosting target (Vercel + managed worker, or AWS/Azure) and domain

### Questions for you
1. **Firm size and user count** at launch? (affects infra choices)
2. **Hosting/data residency** preference; is Supabase acceptable?
3. **Which AI provider(s)** has the firm vetted, if any?
4. **Does the firm already own a legal research subscription** (Westlaw/Lexis/Fastcase via Florida Bar membership, etc.)? This changes the research architecture materially.
5. **Existing case-management system** (Clio?) — do you want it as source of truth or this app?
6. **Single firm or multi-firm SaaS?** I've included `firm_id` everywhere; confirming avoids rework.
7. **Does the firm's policy require client AI consent** for all matters or by matter type (sets the default of the transmission gate)?
8. Priority practice area to validate first (I'd suggest residential/commercial real estate).

---

## J. PHASE 1 PLAN (exact)

**Goal:** a secure, empty-but-real foundation: sign in with MFA, see only matters you are assigned, create matters, upload/download documents, everything audited, isolation proven by tests. **No AI in Phase 1.**

### Work packages
1. **Repo scaffold** — Next.js + TS strict + Tailwind, ESLint/Prettier, Vitest, Playwright, Drizzle migrations, `.env.example`, gitleaks, CI (lint, typecheck, unit, RLS tests). Docker compose for local Postgres.
2. **Design tokens + shell** — LEVACY design tokens, collapsible sidebar with all 12 nav items; unbuilt ones route to `ComingSoon`.
3. **Database migrations** — firms, users, roles, permissions, role_permissions, user_roles, clients, contacts, matters, matter_types/statuses, matter_users, workspaces, notes, documents, document_versions, tasks (basic), audit_logs, retention_policies (table only). Seed: roles, permissions, matter types, statuses, document categories (from spec lists).
4. **Auth** — invite-only onboarding, login, MFA enroll/verify, session timeout, remote revoke, rate limiting.
5. **Authorization layer** — `authorize()`, `AuthorizedContext`, permission keys, RLS policies for every matter-scoped table, ethical-wall flag.
6. **Matters** — list (paginated, filter), create (all spec fields), edit, status transitions, assign users, Matter workspace shell with the 11 tabs (Overview, Documents, Notes, Tasks, Contacts, Activity working; AI Chat/Research/Deadlines/Generated/Financial → `ComingSoon`).
7. **Documents (basic)** — upload (type/size/MIME validation; malware-scan hook that is **REQUIRES SETUP** if ClamAV isn't configured, and uploads are held in "Quarantined – scan not configured" or admin explicitly allows), versions, metadata, tags, category, folder, rename, signed download, status display (Uploading → Ready/Failed).
8. **Home** — prompt box UI present but disabled with `COMING SOON` (AI is Phase 3); recent matters/documents, my tasks lists working.
9. **Audit foundation** — hash-chained append-only log; events: login, matter access, doc upload/download/delete, permission changes, admin actions; admin audit viewer (paginated, filterable).
10. **Admin (minimal)** — users, role assignment, matter-access assignment, document categories.
11. **Tests (must pass to exit Phase 1)** — auth + MFA flows; role/permission matrix; **Matter A/B isolation** via UI search, API, direct URL, document download URL, storage path guessing, and direct SQL under the app role (pgTAP); audit chain integrity; upload validation.

### Out of scope for Phase 1
AI, embeddings, OCR, research, any practice module, integrations, calendar.

### Definition of done
All Phase 1 tests green in CI; a written isolation test report; a demo walkthrough; sample data (if any) clearly labeled **SAMPLE DATA**.

---

## Approval requested

Please confirm or adjust: (1) the stack (esp. Supabase vs alternative), (2) answers to the questions in §I, (3) permission to begin Phase 1 as scoped in §J. No code will be written until you approve.
