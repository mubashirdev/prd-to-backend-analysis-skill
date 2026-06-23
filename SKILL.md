---
name: prd-to-backend-analysis
description: "Use when the user gives a PRD Notion link and asks for a backend tech analysis, says 'PRD to backend analysis', or starts a flight's BE analysis. Trigger: /prd-to-backend-analysis <prd-link>. Backend only. Do NOT use for frontend analysis, finalizing/implementing an already-approved analysis, or a quick feasibility/effort opinion (use `assess`). Heavyweight: it mutates git state and creates a Notion page."
---

# PRD → Backend Tech Analysis

Produces the **backend tech analysis** a developer writes once a PRD is ready: the manager + FE review it, the team socializes the open decisions, then development proceeds against it. **Backend only** — you don't design the FE, but you DO specify the surfaces FE builds against (routes + socket payloads, when the feature has them).

**Calibrate altitude from a real sibling, not a hard-coded name.** In Phase 2, `notion-search` for an existing "… — Backend Tech Analysis" page and skim one for **altitude and leanness — not its section inventory** (the *AI Agents: Instruction Validator* analysis is a good one if it surfaces; if none surfaces, fall back to the template below). Include only the sections this PRD's backend actually needs, and **derive sync-vs-async from the code, never from the feature category** — e.g. a "search field" write may be a direct index write OR a queued reindex (SQS → streamer lambda → Redis → reindex); trace the actual path before deciding which sections exist.

## Prerequisites
- The Notion MCP (`mcp__claude_ai_Notion__*`) must be connected, with **write** tools allowlisted: `notion-create-pages`, `notion-update-page` (plus `notion-fetch`, `notion-search`). Without them the first write halts the autonomous run on a permission prompt.
- Fan-out uses the **`Workflow`** tool (this skill instructing it is valid opt-in). If `Workflow` isn't available, fall back to dispatching parallel **`Agent`** subagents with the same per-agent contract.

## Operating rules
- **Run autonomously.** Don't stop to ask clarifying questions. Unsure, or a product/business judgment → a **To be discussed** row (with a recommendation), never a blocker. (Legitimate hard stops: a relevant-dirty git tree, or a missing/unusable/inaccessible PRD link — see the phases.)
- **Backend only.** Don't design FE UX; do specify routes (method · path · ACL · request · response) and socket payloads when present — that's the FE contract.
- **Standardize on existing patterns.** Find the closest precedent in the repo and mirror it; call out what you're mirroring. Prefer reuse over invention.
- **Verify against code (`file:line`).** Never assert behavior you haven't read. A "reused" route/model/serializer is frequently NOT inert — hidden background LLMs, serializers that reshape the payload or read Redis, async double-writes. Read it before depending on it. **`@respond-io/*` packages may not be installed** — verify their behavior from in-repo **call sites** (e.g. `OrganizationQuotaUsage` usage in a lambda manager, `POLICY_TYPES` in a routes file), or `npm install` the owning module only if you genuinely need package internals.
- **Never fabricate numbers** (cost, latency, pricing). Frame cost as token-volume × calls, leave rates to fill in, and mirror sizing (e.g. timeouts, memory) from the closest existing resource.
- **Look up current docs for anything external — don't trust memory.** If the feature needs a NEW external/vendor API, SDK, or a version-sensitive surface (LLM models + params, payment/provider APIs, a third-party endpoint) there's no in-repo call site to verify against and your training is stale. `WebSearch` + `WebFetch` the **latest official docs**, confirm the endpoint/params/model-id still exist, and cite the source URL + the date checked in the analysis. For Claude/Anthropic specifically, use the **`claude-api`** skill as the source of model ids, pricing, and params.
- **Team standards:**
  - *Universal:* soft-delete via Sequelize **`paranoid: true`** (manages the `deletedAt` column); ACL via `@respond-io/access-control` (`MANAGER_ACCESS` etc.); tenancy scoped to `req.space` and verified per route.
  - *Conditional (only if the feature uses them):* SQS retry is the standard **`ReportBatchItemFailures` (`batchItemFailures`) + `VisibilityTimeout` + `MessageRetentionPeriod`, ~3 retries per job, no DLQ** — just apply it; credits via `OrganizationQuotaUsage` (atomic in-lambda, refund on failure); a Redis lock for user-triggered long-running jobs.
- **Show the artifact, don't describe it.** Every contract is a fenced code block of the REAL shape, not prose. That means: each route with a **Sample request** + **Sample response** JSON; each event/webhook with its full payload JSON (provider→platform notification, the enqueued SQS message, the unified internal message, and any event emitted); each new/changed DB row as `CREATE TABLE` + a sample row; each new constant/enum addition shown as the literal line added; each new file shown as a short skeleton of its key method(s) and the exact object it builds. "Mirror gmail" / "see X" is NOT acceptable for a core contract — paste the concrete shape the dev will write. The bar is the sample *Ai Agent analysis* doc: dense with real JSON and schemas.
- **Cover everything — the doc is build-ready.** A developer must be able to implement the whole in-scope feature from this doc WITHOUT re-reading the codebase. Every payload at every hop; every package (`@respond-io/*`) edit with the literal additions; every cross-service enum entry — shown concretely, with the touched file cited `file:line` inline where it lands. Gaps/uncertainties go to **To be discussed** — but a known contract is never left as a pointer.
- **Lean in prose, complete in artifacts.** Tables and code blocks over paragraphs; cut narration, hedging, and restated PRD. Omit a whole section only when the feature truly doesn't use it (`## Not needed` records deliberate absences) — but never omit or abbreviate a payload/schema/file-shape to "stay lean". Concrete contracts are the point, not bloat.
- **Plugin-portable.** No user-specific absolute paths. Find the repo from the working dir; reach Notion via the connected MCP.

## Checklist — create a TodoWrite todo per phase, complete in order
1. Pre-flight
2. Ingest PRD
3. Explore codebase (Workflow fan-out)
4. Draft into Notion
5. Review + validate-and-fill loop (Workflow fan-out, ≤ 5 rounds)
6. Present

## Phase 1 — Pre-flight
- **No PRD link?** If the skill was invoked without a PRD link (empty arg), ask the user for it and wait — the one permitted clarifying question. Do nothing else (no git checkout/pull, no Notion writes) until you have it.
- **Two roots — don't conflate them:** the **module-map dir** = the dir holding the module-map `CLAUDE.md` (often the cwd, e.g. `…/service`); the **git root** = `git rev-parse --show-toplevel`, which may be an **ancestor** of it. Run all git commands at the **git root**, and remember its `git status` paths are git-root-relative (so a module appears as e.g. `service/workflows/`).
- If not on `dev`, `git checkout dev`; then `git pull --ff-only`.
- **Dirty-tree rule:** hard-stop **only** if *tracked* files under the modules you'll analyze are uncommitted — `git -C <gitroot> status --porcelain -- <gitroot-relative module dirs>`. **Tolerate** untracked files and edits unrelated to the analysis, including a modified module-map `CLAUDE.md` (read its working-tree copy as-is). Already on `dev`, nothing relevant dirty → proceed.
- If `git pull` fails (diverged / conflict / no network / auth — possibly from sibling trees outside your modules): do **not** abort — proceed read-only against current `HEAD` and add a To-be-discussed row noting the analysis ran against possibly-stale local code.
- Only on a relevant-dirty tree: stop, name the blocking tracked files, ask the user to commit/stash. Never auto-stash.

## Phase 2 — Ingest PRD  (all Notion access stays in the main agent)
- Fetch the PRD link via `notion-fetch`. **If it returns access-denied / not-found / empty → STOP and report the exact link + error** (unusable input — a legitimate stop).
- Follow linked/child sub-pages that matter to the backend (design docs, prompt pages, decision-item DBs): extract child page ids from the fetch result and re-fetch them (cap depth ~1–2). A fetch too large to read inline is saved to a file by the tool — `Read`/slice it, or hand the path to a subagent; don't skip it.
- **Read the PRD structure, not just the prose.** Respond.io PRDs are usually: Overview → Problem Definition → Scope Summary (**Must Have** / **Future Ideas**) → numbered functional Sections → data-contract tables → callouts → Decision Items. When parsing:
  - The atomic requirement unit is **screen → UI element → {Text/Label, Location, Action, Behaviour}** nested in `<details>` blocks. Treat the **`Behaviour:`/`Behavior:`** bullets as the implicit acceptance criteria — there are rarely `As a user…` stories or explicit acceptance-criteria headings.
  - **Tables** carry the data contracts (events, properties, permission matrices) — extract them as contracts, not prose.
  - **Red callouts** = "not ready / don't build yet" or scope caveats; **yellow callouts** = engineering notes (URL routes for Pendo, strings needing translation). Surface both.
  - Anything under **Future Ideas and Scope** is out of scope — do not design for it (note it in `## Not needed` if it shaped a decision).
- **Calibrate:** `notion-search` for a prior "… — Backend Tech Analysis" and skim one for altitude/leanness (see the intro). Don't copy its sections — copy its terseness.
- **Find-or-create the output page** (idempotent — a re-run rewrites in place instead of spawning a duplicate, and a run that died in a later phase leaves no orphan page behind): `notion-search` for the exact title `"<PRD name> — Backend Tech Analysis"`; if an owner-owned page with that title already exists, **reuse its page id**; otherwise create it with **`notion-create-pages`**, `parent` **omitted** (→ a workspace-level, owner-only page) and that title. Capture the page id/URL — Phase 4 pushes with `replace_content`, so reusing an existing page cleanly overwrites any stale content.
- **Stage the PRD for the fan-outs:** `Write` the ingested PRD + relevant sub-pages, as a concise requirements digest, to **`prd-digest.md` in a run scratch dir _outside_ the working tree** (e.g. under `$TMPDIR` or `/tmp` — never the repo cwd, so it can't litter the user's tree or get swept into a later `git add`). Spawned explorer/reviewer agents can't reach Notion, so pass them this file's **absolute path** — it (not Notion) is the PRD source for Phases 3 and 5.
- Incomplete / "stakeholder review" PRD → proceed best-effort, note gaps in To be discussed. If a proven prototype/POC exists, treat it as the algorithm reference and frame the analysis as the **port** into the backend.

## Phase 3 — Explore the codebase (two-wave Workflow fan-out)
Orient from the module-map `CLAUDE.md` + the relevant module's `CLAUDE.md`. All explorers are **read-only**, work from the **repo + `prd-digest.md`** (staged in Phase 2; never the Notion MCP — spawned agents may not have it), and return a fixed shape: `{ area, summary, keyFiles:[{path, role}], facts:[<with file:line>], patterns:[…], gaps:[…] }`.

Run **two waves** — because which deep explorers you need *depends on facts you don't have yet*. Don't guess the fan-out up front; let wave 1 decide it.

**Wave 1 — orient & trace (always these four):**
- Owning service + data model + where config/state lives (and its limits, e.g. a `TEXT` cap); the closest CRUD/flow prior-art to mirror end-to-end.
- **Persistence & migrations** — first-class: nearly every feature persists, so **always** trace where/how this feature's data is stored (Sequelize/Dynamoose conventions), don't leave it to a conditional deep-dive. Surface **migration safety** when it alters a large existing table (additive-nullable-then-backfill vs a blocking `ALTER`); detail it in the doc.
- **The write path, traced.** Find the write call site and follow it: direct repository write (sync) **or** an enqueue (SQS → streamer → Redis → reindex, async)? This single fact gates wave 2 — **never infer async-ness from the feature category.**
- Existing routes + the exact ACL/permission pattern (which policy gates what).

**Wave 2 — deep dives, spawned from wave-1 findings (only the ones that apply):**
- **Serializer/resource layer** — does it reshape payloads, read Redis, generate URLs? (a classic non-inert "reuse").
- **Search** *(if wave 1 found an index touch)* — index mapping, analyzer, reindex/backfill, `create-index` script.
- *(If wave 1 traced async)* SQS/Lambda patterns + naming + the closest queue's config; **handler idempotency** (the ~3-retry standard means a job WILL re-run — what makes re-execution safe?); the real-time/socket delivery path + channel convention.
- *(If AI)* existing LLM infra (`@respond-io/ai` / `@respond-io/ai-agent`) — models, structured-output support, credit/tracing — from in-repo call sites.
- *(If a NEW external API / vendor / SDK with no in-repo precedent)* this explorer **looks up the latest official docs** (`WebSearch` + `WebFetch`) — confirm the real endpoints/params/auth/model-ids and return them with the source URL + date checked. Don't design against a remembered API shape; for Anthropic, defer to the `claude-api` skill.
- **Cross-service ripple** — a new actor/field/capability often touches services the PRD never names. Trace, for THIS feature: does it count toward billing/plan limits? appear in user/assignee dropdowns or pickers? show in reports, data export, or super-admin? get gated by a permission? emit/consume events other services read? Don't reuse a fixed checklist — derive it from what this feature actually introduces.

Synthesize both waves before drafting; the design must be grounded in what exists.

## Phase 4 — Draft into Notion
**The local file `draft.md` (same scratch dir as `prd-digest.md`, _outside_ the working tree) is the single source of truth; the Notion page is a render of it.** Write the full doc to `draft.md` first, then push it to the page (captured in Phase 2) with **one** `notion-update-page` `command: replace_content` (pass `new_str` = the whole doc; `allow_deleting_content: true`). Phase 5 edits `draft.md` and re-pushes the same way. **Never use `insert_content` to add sections in the loop** — appending across rounds duplicates content on the page; always rewrite from `draft.md`. Use the **adaptive template** below.
Conventions: when an entity is **reused as-is**, say so; when **reused but modified**, mark the modified parts; **`~~strikethrough~~`** any existing table/route/field the new design supersedes, and **✅** anything the PRD says already exists/landed.
Notion-flavored-markdown gotchas: toggle headings use `{toggle="true"}` and their children **must be tab-indented** (fence lines carry the indent; code content sits flush-left); tables use `<table>`/`<tr>`/`<td>`; keep every JSON payload **clean multi-line** (one field per line) so it doesn't overflow. If unsure of syntax, read the spec via `ReadMcpResourceTool(server="claude_ai_Notion", uri="notion://docs/enhanced-markdown-spec")` — do **not** route that URI through `notion-fetch`.

## Phase 5 — Review + validate-and-fill loop
Reviewers read `draft.md` + `prd-digest.md` AND verify against the repo (`file:line`). Run a **Workflow** of 6 **read-only** reviewer lenses, each returning `{ lens, gaps:[{title, severity, where, problem, recommendation}], verdict }`:
1. PRD coverage · 2. Concurrency, correctness & **idempotency** (retried jobs, double-writes, race on the credit/lock) · 3. API & data contract · 4. Security / ACL / tenancy / cost · 5. Ops / failure / rollout / **migration safety** · 6. **Completeness & concreteness critic** — hunt what the doc describes in prose but does NOT show as a concrete artifact (a missing payload JSON, a missing sample request/response, a missing package-change literal, a file in the flow named without its `file:line` or skeleton, a "mirror X"/"see X" left as a pointer). Every such omission is a finding; the bar is "a dev can build it all from this doc without re-reading the codebase."

**Verify before you fill.** A reviewer's gap is a claim, not a fact — plausible-but-wrong findings will corrupt the doc if applied blindly. For each high/medium finding, do a cheap adversarial check: does it actually reproduce at the cited `file:line`? If it doesn't hold, drop it. Reviewers **return findings only — the main agent applies all edits to `draft.md`, then re-pushes via `replace_content`** (subagents touch neither the file nor Notion). For each surviving finding: correctness/contract gap with a clear best answer → **fill the doc**; product/business-judgment or genuinely uncertain → **To-be-discussed row** (with a recommendation). Auto-fix high+medium; low → note.
**Loop:** track the round in TodoWrite and keep a seen-set of findings (dedup by `where`+claim). After filling, re-run the review; **stop when a round surfaces no new high/medium findings, hard cap 5 rounds** — at the cap, dump any remaining findings into To be discussed rather than looping.

## Phase 6 — Present
Give the user the Notion link + a tight summary of what landed and the open **To be discussed** items. Clean up the scratch dir (`prd-digest.md` + `draft.md`) — the final doc now lives in Notion. **Terminal state = a draft with `_pending_` decisions.** Do NOT proceed to implementation or invoke implementation skills.

---

## Output document template (adaptive — include only what the PRD needs)
H1 (`#`) top sections, H2 (`##`) groups, H3 (`###`) toggles for long code/JSON (each route, each socket payload). Start with `<table_of_contents/>`. Order:

### # Frontend API contract  *(if the feature exposes routes/sockets — put it FIRST)*
Intro: service + ACL policy + scoping + the per-route tenancy lookup (verify the entity belongs to the caller's space before any work) + plan gate.
- `## Routes` — `### <METHOD path> {toggle="true"}` each, with method · path · ACL · and **a `<details>` Sample request + a `<details>` Sample response (real JSON, success + every error)** — exactly like the sample analysis's `# API` section. Mark reused endpoints ✅ and note what's being extended.
- `## Socket` *(if any)* — an envelope code block (`channel`/`name`/`envelope`, one field per line), then `### <EventType> {toggle="true"}` each with clean multi-line JSON.

### # Payloads  *(REQUIRED whenever the feature has webhooks, queues, events, or provider calls)*
The contract at every hop, as real JSON. Mirror the sample analysis's `# Payload` section. Include each that applies, in `### <name> {toggle="true"}` blocks:
- **Inbound webhook** — the exact provider→platform notification body (e.g. the Graph change-notification JSON, validation-handshake request).
- **Provider request/response** — the outbound call to the external API (e.g. Graph `sendMail` body) and its response.
- **Queue messages** — the JSON enqueued to each SQS queue (inbound webhook queue, outbound send queue) — one field per line.
- **Unified internal shape** — how the message/event is normalized into the platform's internal format (the `type`, all fields).
- **Emitted events** — any chat-activity/Segment/domain event the flow fires, full payload (Segment: include userId + orgId).
Never describe a payload in prose ("subject, cc, bcc…") — paste the JSON.

### # Infrastructure  *(describe each NEW resource concretely — the manager's inventory)*
- `## Database` *(if persistence)* — table columns + **indexes sized to the real reads** + soft-delete (`paranoid: true`) + relationship (1-to-1 vs 1-to-many) and upsert/insert behavior; why this store. **Migration:** new table vs altering an existing one — if altering a large table, the safe path (additive nullable column → backfill → enforce) not a blocking `ALTER`.
- `## Search (OpenSearch)` *(if used)* — index/alias, mapping + analyzer for new fields, write path (sync vs queued reindex), backfill/reindex plan for existing docs.
- `## SQS` *(if async)* — queue names + the standard retry (`ReportBatchItemFailures`, ~3 retries, `VisibilityTimeout` + `MessageRetentionPeriod`, no DLQ) + BatchSize + **what makes the handler idempotent** (retries re-run the job — dedup key / conditional write / refund-on-failure).
- `## Lambda` *(if async)* — function names + sizing; reference the flow section for logic.
- `## Redis` *(if used)* — keys, values, TTL, lock semantics; any new cache primitive needed.
- `## External integration` *(if a new vendor/external API)* — endpoint(s) · auth · request/response shape · the exact model-id/version, each **citing the official doc URL + date checked** (per the lookup rule). Flag rate limits and where the secret/key lives.
- `## Access & cost` — tenancy + plan gate (universal); credit gating (where it decrements + refunds) and lock-concurrency *(only if applicable)*.
- `## Cross-service impact` *(if it ripples)* — per affected service, what must change there and why (billing/plan limits, auth/login, dropdowns & pickers, reports, data export, super-admin, permissions, events other services consume). Derived fresh per PRD — see Phase 3.
- `## Package changes (@respond-io/*)` *(if a shared package changes)* — these are sibling source repos compiled into `node_modules`; the edit lands in the source repo and is **republished + version-bumped**, never edited in `node_modules`. Show the **literal additions** (the exact `SOURCE`/enum lines, the new registry-map entry) and a short skeleton of any new package class. Note the publish-before-deploy sequencing.
- `## Rollout & observability` — feature-flag/dark-launch lever, structured job logs + a CloudWatch metric/alarm.
- `## Not needed` — explicitly record deliberate absences (OpenSearch none? new vendor none? rate limit none? Future-Scope items deferred?).

### # Business logic / flow
Diagram convention: render any flow diagram as a **`mermaid` fenced code block** (Notion renders it natively); skip the diagram on a single-hop flow.
- *async:* `## <Flow> — <lambda> lambda` — flow diagram + `###` steps, each step showing the **real payload it produces/consumes** (link the shapes in `# Payloads`; structured-output schema / scoring only **if AI**).
- *sync:* `## <Operation> — <service> service` — the route → controller → repository → serializer → search chain with the real object at each hop; a flow diagram only when there's >1 hop or a fan-out.

Name new/changed files **inline at the step that touches them** — `path` + the key method/object a new file builds, or the exact edit (`file:line`) for a modified one. This is where file-level detail lives; there is **no separate changeset table** (see Cut-list).

### # Testing
- `## Integration tests {toggle="true"}` — mock the heavy/expensive infra; call cheap real services at the boundary; **state which is which for THIS feature** (AI: mock DB/SQS/socket/Redis, call the LLM for real but self-skip when the API key is absent; search: test-index vs mocked search repo). Describe *what* to test, not the test framework — the runner is implementation, out of scope for the analysis.
- `## QA testing {toggle="true"}` — manual scenarios.

### # To be discussed
Intro: "Fill the Decision column once agreed — the development agent reads decisions from here." A `<table>`: `# | Topic | Recommendation | Decision`, each Decision `_pending_`. Only genuinely-open items; move ambiguous/non-user-facing design choices here.

## Cut-list — never include
No "Goal" preamble. No standalone "Security/Scalability/Reliability" section (fold specific, non-default items into Infrastructure). No "Effects on other areas / scope" chatter. No "Implementation notes". No redundant "LLM models" section (fold model choice into a code block + a To-be-discussed row). No exhaustive `# Files & changes` changeset table — that's an implementation-plan artifact, not analysis; surface touched files as inline `file:line` within the Infrastructure/Flow sections instead.
