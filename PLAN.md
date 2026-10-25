# PLAN.md — Study Buddy

Rebuild of ["We Need a Name"](https://github.com/Tony04305/uva-hackathon-we-need-a-name) (WNAM) as a
document-to-practice quiz app. This plan follows the build document's working method: read the
reference, decide what to keep/change, define the data model, API routes and pipeline, then stop
for review before any code is written.

## 0. Decisions (D1–D9)

Idil left all nine decisions blank, so every default from the build document applies. Listed here
so they're easy to override:

| # | Decision | Applied default |
|---|----------|------------------|
| D1 | Gamification | Keep levels (1–7) + streaks. Drop plants, collection, leaderboard for v1. |
| D2 | Question modes | `multiple_choice`, `open`, `mixed`. |
| D3 | Accounts | Guest profile (name + 4-digit tag), quiz sets tied to the guest token. |
| D4 | Sharing | No sharing in v1; data model allows adding it later. |
| D5 | Review before practice | Yes — see §0.1 for what this screen actually does. |
| D6 | File types | PDF now; DOCX/TXT/MD in phase 2 — see §0.1. |
| D7 | Max upload | 10 MB / ~60 pages of text. |
| D8 | AI tutor | Yes, grounded only in the source chunk of the current question. |
| D9 | LLM provider | One interface, swappable; recommendation below — see §7. |

## 0.1 Phase 1 (build now) vs. phase 2 (later)

Idil's feedback narrows the scope and adds one item (sharing) earlier than the build document's
default. Nothing here changes the data model in §4 — it's designed so phase 2 is additive, not a
rebuild.

**Phase 1 — build now:**
- PDF upload only (DOCX/TXT/MD come later; D6 narrowed).
- Multiple-choice questions only. No question-type selector in the UI yet — it would just offer one
  option. The `questions`/`question_options`/`rubric_criteria` tables already support `open`
  underneath, so adding it later is additive, not a rebuild.
- Levels + streaks (unchanged from §0 default).
- Guest profile only (name + 4-digit tag). No login/password yet.
- Review/edit/delete screen before practicing (D5 — see explanation below).
- Floating AI tutor hint chat (D8).
- **Sharing a quiz set by link** — moved up from "later" to phase 1, see §4/§6.
- Wrong-answer feedback: always shows *why* it's wrong and what the correct answer is (not just
  a checkmark) — this was already in the plan (§8), confirming it explicitly here since it was
  asked about directly.

**Phase 2 — design for it now, build later:**
- Open questions + rubric grading, then the MC/open/mixed selector (D2) once both exist.
- DOCX/TXT/MD upload (D6).
- Plants/collection (D1).
- Leaderboard (D1) — per quiz set, not global: once a quiz set is shared, the people who practiced *that* set can see how they rank against each other on it. A global, cross-document leaderboard wouldn't compare anything meaningful (different documents, different difficulty), so this replaces WNAM's single shared leaderboard design, not just re-adds it.
- Real accounts: a login that lets someone come back and see study sessions made as a guest.
  Future-proofed now: a guest profile is just a row with a token; "upgrading" it to a real account
  later means attaching an email/password or OAuth identity to that *same* row, not migrating
  `quiz_sets`/`questions`/`attempts` to a new owner. No schema change needed today for this to stay
  possible.

### What D5 actually means (you asked)

Before you start practicing, you get a screen listing every question the AI generated for your
document, each with its full answer key (correct option, explanations). You can read through them,
fix a question that's worded badly, or delete one that's wrong or pointless, *before* it ever
becomes something you get quizzed on. Once you're happy and hit "start practicing", that screen
locks — from that point on the app follows the same rule as the reference app: you never see the
key before you answer.

## 1. What we keep from WNAM

- **The client never sees the answer key before submitting.** The server resolves the question by
  ID; the client sends only the attempt. This is the single most important property to preserve,
  and it gets *harder* here because the answer key itself is LLM-written, not curated — see §5.
- **LLM output is never trusted raw.** JSON-parsed, schema-checked field by field, length-limited,
  plain-text-only, rejected on deviation (`economics-grader.js`, `tutor.js` are the templates).
- **Untrusted student/document text in prompts.** WNAM already treats student answers as untrusted
  and instructs the model accordingly. We extend the same treatment to uploaded document text,
  which is a new, larger untrusted-input surface (see §5, "untrusted documents").
- **AI failure never corrupts progress.** Grading failure leaves the attempt and streak unchanged.
- **Progression rules.** 7 levels, 3-in-a-row to level up, wrong/timeout resets streak not level,
  timed levels at 2/4/6 with a fixed time budget. This logic (`progress.js`) is close to reusable
  as-is; the state stays client-side (localStorage), the server stays authoritative only for
  question content and grading — same split as WNAM.
- **Real tests on core logic**, `node --test`/Vitest, including hostile/malformed model output.
- **Rate limiting per guest** (D1), extended to cover generation jobs per day, not only AI
  requests per minute.

## 2. What we deliberately don't copy

- `main.js`'s mixed rendering/state/logic → split into `api/`, `state/`, `features/`, `ui/`,
  `i18n/` (per the build document's architecture section).
- No types → TypeScript strict, both frontend and worker.
- Schema duplicated in `schema.sql` and inline in `worker.js` → one source of truth, numbered
  migrations (`migrations/0001_init.sql`, …), loaded by both the worker and the test harness.
- Hard-coded question banks → there are no banks; everything comes from the uploaded document.
  This also means WNAM's **hint-selection** pattern (pick one of ~3 pre-written, human-reviewed
  hints per question) cannot be reused as-is: there is no pre-written hint bank for an arbitrary
  document. The tutor must *generate* a grounded hint from the source chunk instead of *selecting*
  one from a fixed list. The safety shape stays identical (schema-checked output, give-up/off-topic
  redirects, never reveal the key pre-submission) — only "select" becomes "generate under the same
  constraints". Flagged explicitly because it's the main place WNAM's safety pattern needs new
  logic rather than reuse.
- Plants/collection/leaderboard code and tables (`garden.js`, `rarities.js`, `plants.js`,
  leaderboard SQL/routes) are dropped per D1, not ported and stubbed.

## 3. Architecture

- **Frontend**: Vite + TypeScript (strict), vanilla TS. Reasoning (per the build doc's "justify in
  one paragraph" requirement): the UI surface here is forms, lists, quiz cards, a status poller and
  a chat panel — no large interdependent reactive state tree that would justify Preact/Svelte's
  overhead. WNAM already proved vanilla JS works for a harder version of this UI (math keyboard,
  voice input); the only real complaint about it was organization, not the lack of a framework.
  We fix the organization problem directly (`api/`, `state/`, `features/`, `ui/`, `i18n/`) instead
  of pulling in a framework to fix it indirectly. Revisit if slice 2's practice UI turns out to need
  real reactivity.
- **Backend**: Cloudflare Pages Functions / Worker, TypeScript, routed by resource (see §6).
- **Storage**: D1 for relational data, R2 for raw uploads (deleted after extraction, see §8).
- **Async generation**: Cloudflare **Workflows**, not Queues. Reasoning: the pipeline
  (extract → chunk → plan coverage → generate → verify → persist) is a linear sequence of steps
  that each need independent retry and a durable "progress" state — which is exactly what
  Workflows gives for free (step-level retries, `step.do()` checkpointing). Queues would need a
  second consumer deployment and our own progress bookkeeping on top. Workflows run inside the
  same Pages/Workers project. Revisit only if a step turns out to need true fan-out concurrency
  Workflows can't express cleanly.

## 4. Data model

Mirrors the build document's minimum list, fleshed out. Numbered migrations, one source of truth
used by both the worker and tests (as WNAM's `createWorker()` test-injection pattern already shows
is possible).

```
guest_profiles
  id, name, name_key, tag, token_hash, ui_language, created_at, updated_at
  -- no plants/score/leaderboard columns; identity + rate limiting only

documents
  id, guest_profile_id, original_filename, mime_type, byte_size,
  storage_key, page_count, status (uploaded|extracting|extracted|failed),
  error_message, created_at, deleted_at

chunks
  id, document_id, chunk_index, page_start, page_end, heading,
  text, token_count, created_at

generation_jobs
  id, document_id, quiz_set_id, question_type, question_language,
  requested_count, status (queued|extracting|generating|verifying|ready|failed),
  generated_count, verified_count, accepted_count, error_message,
  started_at, finished_at

quiz_sets
  id, guest_profile_id, document_id, title, question_type, question_language,
  requested_count, accepted_count, status (draft|ready|published|failed),
  share_token (nullable, unique), created_at

questions
  id, quiz_set_id, chunk_id, type (mc|open), prompt, source_quote, page_reference,
  difficulty (1-3), cognitive_level (recall|understanding|application), level_assigned (1-7),
  model_answer (open only), empirical_correct, empirical_attempts,
  status (active|reported_hidden|deleted), created_at

question_options   -- mc only
  id, question_id, label, is_correct, position, explanation

rubric_criteria     -- open only
  id, question_id, criterion_key, label, max_score (2), description

attempts
  id, question_id, quiz_set_id, guest_profile_id, submitted_answer (json),
  is_correct, score, ai_grade (json, nullable), timed_out, created_at

question_reports
  id, question_id, guest_profile_id, reason (wrong_key|ambiguous|off_topic|other),
  note, created_at, resolved

rate_limits
  profile_id, window_started_at, request_count     -- AI requests/min, per guest
  -- plus a generation_jobs_per_day counter, same upsert pattern as WNAM's tutor_rate_limits
```

Design note on D5 (review before practice): the review/edit screen shows the full question
*including* the answer key, because at that point the viewer is the document owner checking their
own generated content, not a student being quizzed. Once a quiz set is `published`, every practice
endpoint (`/practice/next`, grading) drops key-bearing fields from the response — same boundary
WNAM draws, just moved to a `published` flag instead of being permanent. Flagging this because it's
a deliberate interpretation of D5, not explicit in the build document; happy to change it.

Progression (level/streak/timer) itself is **not** a table: it stays client-side per quiz set,
exactly like WNAM's `progress.js`/localStorage, because the build document keeps "everything else
(levels, streaks, timed levels, tutor, feedback)" working the same way, and WNAM's own split
(server owns content + grading, client owns progression state) is one of the properties worth
keeping. `attempts` is the server-side log used for grading, empirical difficulty, and reports.

**Design note on sharing (now, not later):** a published quiz set can get a `share_token` (random,
generated on request). Anyone with the link can start practicing that quiz set as their own guest
profile — their own level/streak/attempts, same questions. The owner's edit rights and the
answer-key boundary are unaffected: a visitor following a share link goes straight into the
published, key-free practice flow, never the review screen. No login needed for this, which is why
it fits before real accounts exist.

## 5. Generation pipeline (server-side, Workflow)

1. **Upload** (`POST /api/upload`) → store in R2, create `documents` row (`uploaded`), return
   document ID immediately. Never blocks on generation.
2. **Extract**: PDF via a Workers-compatible PDF text extractor, DOCX via a Workers-compatible
   DOCX-to-text library. Both compatibility claims get verified with a real file in slice 1 — this
   is a spike, not an assumption (candidates to try first: `unpdf` for PDF, `mammoth` for DOCX;
   confirmed or replaced once actually run in a Worker). Scanned/image-only PDFs (no extractable
   text) are rejected with "This PDF contains no selectable text."
3. **Chunk** by heading/paragraph into ~500–1,000-token pieces, stable IDs, page references.
4. **Plan coverage**: distribute the requested question count across chunks proportional to chunk
   length, so a 60-page document isn't front-loaded onto page 1.
5. **Generate** per chunk, structured output, one call per chunk (or small batch of chunks) against
   the LLM interface's `generateQuestions`. Every question carries `source_chunk_id`,
   `source_quote` (≤300 chars, verbatim), and type-specific fields (MC: 4 options/1 correct/
   per-option explanation; open: model answer + 2–4 rubric criteria), plus `difficulty` and
   `cognitive_level`.
6. **Verify**, both deterministic and a second LLM call:
   - Quote-verification: `source_quote` must appear verbatim (whitespace-normalised) in the chunk.
     Deterministic, reject on failure — no LLM call needed for this one.
   - Independent verifier call: answers the question from the chunk alone, never sees the key;
     reject on disagreement or flagged ambiguity.
   - MC sanity: no duplicate options, no "all/none of the above", correct option not systematically
     longest, correct position randomised server-side before storage.
   - Deduplicate near-identical questions (embedding or string-similarity threshold — picked in
     slice 2 against real generated output, not assumed now).
   - Reject metadata questions ("according to chapter 3…", page numbers, authors) unless the
     content itself requires it.
7. **Persist** only accepted questions; store rejection counts on `generation_jobs` for monitoring.
   If fewer questions survive than requested, the quiz set says so honestly (no padding).
8. **Status**: `generation_jobs` row updated at each Workflow step; `GET /api/jobs/:id` exposes
   `extracting → generating N/M → verifying → ready/failed`. Frontend polls (SSE is a possible
   upgrade, not needed for a job that finishes in well under a minute for 60 pages).

### Untrusted documents

Document text is delimited clearly in every generation/verification prompt, the model is told
it's being handed untrusted data, and all output is still schema-checked regardless of what the
document contains — same belt-and-suspenders approach WNAM already uses for student text.

## 6. API routes

```
POST   /api/profile                  create guest profile (name + 4-digit tag, cookie token)
GET    /api/profile                  current profile

POST   /api/upload                   upload file → documents row (uploaded)
DELETE /api/documents/:id            delete document + cascaded chunks/quiz sets/questions/attempts

POST   /api/documents/:id/quiz-sets  create quiz set + generation job (type, language, count, pages)
GET    /api/jobs/:id                 poll job status/progress
GET    /api/quiz-sets                list guest's quiz sets
GET    /api/quiz-sets/:id            quiz set detail (full content incl. key — review screen, pre-publish)
POST   /api/quiz-sets/:id/publish    lock editing, mark ready for practice
POST   /api/quiz-sets/:id/share      generate/return the share_token for a published quiz set
GET    /api/share/:token             public, no-login: start practicing a shared quiz set

PATCH  /api/questions/:id            edit during review (pre-publish only)
DELETE /api/questions/:id            delete during review (pre-publish only)
POST   /api/questions/:id/report     "report this question" (wrong key/ambiguous/off-topic)

GET    /api/quiz-sets/:id/practice/next   next question for current level (no key fields)
POST   /api/attempts                      submit one attempt → grade, explanation, source quote + page

POST   /api/tutor                    hint/review chat, grounded in the question's source chunk
```

## 7. LLM provider (D9) — free option to start

**Yes, you need an LLM provider account for this** — the app itself has no built-in intelligence;
it sends your extracted PDF text to a language model over an API and asks it to write questions
back in a fixed format. That's the part that needs an account.

**Free recommendation to start building: Google Gemini (via Google AI Studio).**
- Genuinely free tier, no credit card needed to get a key: https://aistudio.google.com/apikey
- Rate-limited (fine for one person building/testing a few quiz sets; not something you'd run a
  public app on at scale), but good enough for all of phase 1.
- Supports strict JSON-schema output (`responseSchema`), which fits the "never trust raw LLM
  output" rule just as well as OpenAI's Structured Outputs.
- Handles Dutch well.

**One real caveat, directly relevant to this app's own privacy note (§8):** Google's free tier
terms allow them to use what you send it to improve their products; the *paid* tier does not. For
a study app where people upload their own documents (possibly lecture notes, theses, personal
material), that's worth knowing before real use — not a reason to avoid it while building, but a
reason to decide deliberately before this goes further than you and test documents. Cloudflare
Workers AI (already free in your Cloudflare account, no new signup) is another zero-cost option,
but its default model is the one the build document already flags as too weak for reliable Dutch
question generation — untested here, so I wouldn't assume it works either way.

**Plan**: build phase 1 against the Gemini free tier now (fast to start, zero cost), and let the
build document's own "generation eval" step (§9 — 3 fixture documents, acceptance rate, Dutch
quality) produce real evidence on whether it's good enough, rather than deciding on pricing-page
text. If it's good enough and the data-use terms are acceptable to you, stay on it; if not, moving
to a paid tier (OpenAI, Anthropic, or Gemini's paid tier) is the contained change described in §3,
not a rewrite.

**What I need from you:** a Gemini API key from the link above (or tell me if you'd rather start
somewhere else). It goes into a local `.dev.vars` file (never committed) and, once deployed, into
a Cloudflare Worker secret — never into source code or the repo.

## 8. Difficulty, grading, privacy, limits

- **Difficulty/levels**: LLM-assigned `difficulty`/`cognitive_level` map onto levels 1–2 (recall),
  3–4 (understanding), 5–7 (application), timed at 2/4/6 — same timing structure as WNAM. Empirical
  correctness (`empirical_correct`/`empirical_attempts` on `questions`) adjusts difficulty over
  time; if a level's bucket is empty, practice falls back to the nearest level and says so.
- **Grading**: MC is deterministic server-side. Open answers use LLM grading against the stored
  rubric + source chunk, same shape as `economics-grader.js` (strict JSON schema, untrusted student
  text, failure leaves state unchanged). Graded in the question's language; answers accepted in
  either language. After answering: explanation + source quote + page number, always.
- **Privacy**: raw upload deleted from R2 after extraction (or after N configurable days); deleting
  a document cascades to everything derived from it; document text only goes to the chosen LLM
  provider, nowhere else. A short privacy note ships in the UI (EU users).
- **Limits**: 10 MB / ~60 pages per upload, questions per set, generation jobs per guest per day, AI
  requests per minute per guest (reusing WNAM's atomic upsert rate-limit pattern).

## 9. Quality bar / testing

Same bar as the build document's "no AI slop" section: no speculative code, no restating comments,
small named functions, no `utils.ts` dumping ground, actionable UI-language errors, accessible
controls. Tests (Vitest or `node --test`):

- Progression rules (ported from WNAM's `progress.test.js` style).
- Request validation and LLM-output validation, including malformed/hostile model output
  (mirrors `economics-grading-api.test.js`).
- Quote-verification and MC sanity checks.
- Grading/practice endpoints never leak the answer key before submission.
- Integration test of the generation pipeline with a stubbed LLM.
- **Generation eval**: 3 fixture documents (EN, NL, mixed/technical) + a script reporting
  acceptance rate, quote-verification failures, verifier disagreements, duplicate rate, language
  correctness. Run before calling generation "done"; numbers go in the README.

## 10. Vertical slices (build order)

**Phase 1:**

1. Upload + extraction + chunking, no AI. PDF only. Prove PDF extraction actually works in a
   Worker (spike, not an assumption).
2. MC generation + verification + practice loop with levels/streaks + review/edit screen + report
   button + share-by-link.
3. Floating AI tutor hint chat, grounded in the current question's source chunk.

**Phase 2 (after phase 1 is working and reviewed with Idil):**

4. Open questions + rubric grading, then the MC/open/mixed selector.
5. DOCX/TXT/MD upload.
6. Question-language choice + UI i18n (EN/NL) — the build document's reason for this (questions in
   a different language than the UI or source document) only matters once more than one question
   language is actually supported end to end.
7. Plants, leaderboard, real accounts.

After each slice: run tests, run the app locally, report what works, what doesn't, and what's
uncertain — no claiming something works without having run it.

## 11. Repository

Connected to https://github.com/idilyilmz/study-buddy — local `git init` done, `origin` set to that
URL, nothing committed yet. I will not run `git commit` or `git push` myself, now or later, even if
asked — that stays with you, same as your other repos. I'll stage files and tell you what's ready;
you run the commit/push (or we do it together, never unattended).

## Open questions for Idil

1. **Gemini API key**: get one free at https://aistudio.google.com/apikey and paste it here when
   ready — that unblocks slice 2 (MC generation). Everything else in slice 1 (upload/extraction/
   chunking) doesn't need it, so I can start now.
2. Anything else you want different from the phase 1/2 split in §0.1 (now including: PDF-only,
   MC-only, sharing by link, per-quiz-set leaderboard moved to phase 2)?

Working directory confirmed: `C:\Users\iyilm\Desktop\personal projecten\study-buddy`.
Repository confirmed: https://github.com/idilyilmz/study-buddy (connected locally, see §11).
