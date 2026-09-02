# AI Coach — Build Plan on `mastra-supabase-starter`

**Status:** Draft v1.0 — implementation plan, confirmed decisions locked in
**Scope:** AI Coach only. AI SOS Buddy is referenced only as a stubbed deep-link handoff — no second agent, no Buddy persona (none has been spec'd yet).
**Source:** OverU AI Coach Specification v1.0, Parts 1–4 (Persona & Behaviour, Knowledge & Memory, Evaluation Scenarios, Tools)
**Built on:** this repo as-is (`@mastra/core` 1.54.0, `@mastra/memory` 1.24.0, `@mastra/pg` 1.18.0, `@mastra/rag` 2.4.2, Supabase Auth + Postgres + pgvector)

---

## 0. Decisions this plan is built on

| Decision | Answer |
| --- | --- |
| Guardrail depth | Full pipeline (tiers 1–3), explicitly flagged provisional — the trigger matrix is Scenario 10 / §6.5 as written, not a client/clinically signed-off matrix |
| Deep-link contract | Proposed here (§4), to be corrected once real app routes exist |
| SOS Buddy handoff | Stub only — a deep link, no live second agent |
| Default demo model | `gpt-4o-mini` (the starter's existing default) |

Two things I'm stating as assumptions rather than asking about, because they're cheap to reverse:

- **Message allowance (5 free / 100 premium) is out of scope this round.** It's entitlement/tier logic, not agent behaviour, and the starter has no subscription concept at all yet. Flag it as a known gap, don't build it.
- **This prototype runs on seeded fixture data, not a real backend.** OverU's actual data model doesn't exist yet (confirmed against the wider engagement record). Every field the spec calls "deterministic platform data" — OverU Day, active emotional state, today's Power Move, behavioural insights — is seeded per fixture user in this repo's own Postgres, not fetched from a real OverU service.

---

## 1. Architecture: what the spec's 10 tool sections actually become

The spec describes 11 numbered items under "Tools." Read against Mastra's primitives, they collapse into two very different things, and conflating them would mean the agent burning tool calls on data it needs on literally every turn.

**Injected context (assembled deterministically, every turn, never a tool call):**
§10.1 Today's Power Move, §10.2 Why It Works, §10.4 OverU Day, §10.5 Current Emotional State, §10.6 Behavioural Insights, §10.11 OverU Journey Context — the spec itself says these come from "deterministic platform logic," not agent inference (§7.11, §10.11). Bundling them as one context block also matches the "fewer, higher-level tools" principle already established across the wider engagement.

**Three real tools, agent-invoked:**
1. `viewTheScience` (§10.3) — RAG lookup over evidence sources.
2. `recommendAction` (§10.7 + §10.8 merged — they're the same action: suggest a feature, with a reason, as a deep link).
3. `handoffToSosBuddy` (§10.10) — stubbed per §0.

§10.9 (Coach Memory) isn't a tool at all — it's the memory configuration in §3 below.

---

## 2. Context assembly (not a tool)

A single function, `getCoachContext(userId)`, called before every agent turn (in the request handler / a Mastra step ahead of `agent.generate`/`agent.stream`, not inside the agent's own reasoning loop):

```ts
type CoachContext = {
  firstName: string;
  overUDay: number;
  activeEmotionalState: "ES1" | "ES2" | "ES3" | "ES4" | "ES5" | "ES6";
  currentMood: "Calm" | "Not Calm";
  todaysPowerMove: {
    id: string;
    instruction: string;
    whyItWorks: string;
    evidenceSource: string;
    publicationYear: number;
  };
  yesterdaysPowerMove?: {
    id: string;
    status: "completed" | "skipped" | "partial" | "none";
    userStatedReason?: string; // only if voluntarily shared, per §8.4
  };
  behaviouralInsights: string[]; // e.g. "Frequently completes Power Moves"
};
```

Serialised into the system/context message ahead of the user's turn — not left for the model to request. This is what makes §8 ("the Coach forgets between days, never judges") actually reliable rather than aspirational.

**Data source (this repo, seeded — not real OverU):** a new `user_context` table, one row per fixture user, plus a `power_moves` table (see §5). Naming deliberately echoes the `user_context` view pattern already used elsewhere in the wider OverU/Ubiquitous work, so this isn't a one-off convention.

---

## 3. Memory configuration

Per §8, deliberately thinner than the starter's current default:

```ts
memory: new Memory({
  options: {
    lastMessages: 20,       // matches §8.2/8.4's "last 10" plus headroom; no semantic recall
    // no workingMemory config — the Coach must not accumulate long-term facts (§8.5)
  },
}),
```

No `semanticRecall`. No persistent working memory. Coaching continuity (yesterday's Power Move outcome) comes from `getCoachContext`, not from Mastra searching memory — keeping the "fresh every day" rule (§8.1) a property of the architecture, not the prompt's good behaviour.

---

## 4. Tools and the deep-link contract

Proposed schema — confirm/correct once real app routes exist:

```ts
const OverURoute = z.enum([
  "power_move.today",
  "journal.new",
  "community.home",
  "community.share_win",
  "meetups.list",
  "anti_loop.track",
  "ai_sos_buddy.start",
  "view_the_science",
]);

const DeepLinkAction = z.object({
  type: z.literal("deep_link"),
  route: OverURoute,
  params: z.record(z.string()).optional(),
  label: z.string(),        // user-facing button text
  rationale: z.string(),    // why the Coach suggested it — logged, not necessarily shown
});
```

### `recommendAction`
Input: `{ actionType: enum, reason: string }`. Output: `DeepLinkAction`. Merges §10.7 and §10.8 — there is no meaningful difference between "recommend a feature" and "deep-link to a feature" once the output is structured.

**Tier-1 guardrail on this tool (see §6):** before the action is returned to the client, a deterministic post-processor checks the route is valid and, where tier logic exists later, reachable by this user — closing the exact gap the wider OverU documentation calls out about unvalidated structured actions. For this prototype, "reachable" just means "route is in the enum"; entitlement-aware validation is out of scope per §0.

### `viewTheScience`
Input: `{ topic: string }`. Output: `{ evidence: Array<{ claim: string; source: string; year: number }>; relatedPowerMoveId?: string }`. Backed by RAG over the Power Move evidence corpus (§7).

### `handoffToSosBuddy`
Input: `{ reason: string }`. Output: `DeepLinkAction` with `route: "ai_sos_buddy.start"`. No second agent runs. This is also the tool the safety-mode guardrail invokes automatically on a high-severity signal (§6) — the LLM doesn't have to "decide" to escalate correctly under distress; the pipeline forces the call.

---

## 5. Data & RAG

**`power_moves` table (Postgres, not vector)** — seeded from the client's spreadsheet (~1,400 rows already confirmed ready): `id, es, instruction, why_it_works, evidence_source, publication_year, variant_index`. This is what `getCoachContext` queries deterministically by active ES — never a semantic search.

**RAG index (`coach_knowledge`)** — embeds the *evidence* side of the same data: one chunk per Power Move combining `why_it_works` + `evidence_source`, metadata `{ powerMoveId, es, sourceType: "power_move_evidence" }`. This is what `viewTheScience` queries. When the client's separate ~15–20 source Knowledge Framework document lands, ingest it into the same index with `sourceType: "knowledge_framework"` — additive, not a rebuild.

This means the corpus for `viewTheScience` is real and available *today*, without waiting on the Knowledge Framework doc — the spreadsheet's `Evidence Source` / `Publication Year` columns are already citation-grade content.

**Ingestion path:** extend `src/scripts/ingest.ts` (currently markdown-only via `MDocument.fromMarkdown`) with a second ingestion function for structured rows — chunk-per-row rather than recursive markdown chunking, since each row is already the right unit.

---

## 6. Guardrail pipeline (full, explicitly provisional)

Every response passes through this chain. **The category list and severity bands are Scenario 10 and §6.5 as written — not a client- or clinically-approved trigger matrix.** Every place this surfaces (code comments, the eval report, any demo) says so.

**Tier 1 — deterministic:**
- Pre: basic input sanity (length, empty-message handling).
- Post: `recommendAction` / `handoffToSosBuddy` route validation (§4).

**Tier 2 — agent-as-processor (the provisional part):**
- Pre: a lightweight classifier scoring input against self-harm intent, suicidal ideation, harm-to-others (§6.5, Scenario 10, Scenario 4's revenge-language edge), and severe distress — graded, not binary: venting passes, a specific plan (method/timing/means) escalates.
- Post: a boundary-compliance check against §6 — no diagnosis, no medication/legal advice, no revenge validation, no "I am your therapist/friend" framing, no fabricated evidence. Implementation choice (one combined call vs. separate pre/post calls) is left to build time — test against the §9 fixtures either way.

**Tier 3 — Mastra built-ins:** `PromptInjectionDetector`, `ModerationProcessor`, `PIIDetector`, wired per-agent. **Verify exact instantiation against the installed `@mastra/core` 1.54.0 embedded docs once `npm install` has run** — the bundled Mastra skill in this repo (`.agents/skills/mastra/`) is explicit that remembered API shapes are unreliable and embedded docs in `node_modules` are the source of truth.

**On a high-severity tier-2 result:** deterministic override — the model is not trusted to freelance a safety response. Return grounding/de-escalation language plus UK crisis resources, force a `handoffToSosBuddy` call, and write a row to a new `escalations` table (`user_id, category, score, source_surface, created_at, reviewed`). Nothing reads that table automatically yet — there's no moderator console in this prototype — but the emission exists, which is the one part of the wider "a queue is not an alarm" problem this repo can actually fix today.

---

## 7. The instructions prompt

§1–§6 of the spec become the `instructions` string. Structure it in the same order as the spec (Mission → Hierarchy of Truth → Tenets → Response Architecture → Style & Tone → Forbidden Behaviours) rather than compressing it into a shorter paraphrase — the eval scenarios test specific §6 sub-clauses (e.g. "never say 'you must be devastated'"), so specificity in the prompt is worth the token cost here. Level 1 (Safety) gets a one-line pointer to the guardrail pipeline rather than trying to re-derive safety behaviour from prose alone — the pipeline is the backstop, the prompt is the first line.

---

## 8. Build order

1. **Migrations** — `power_moves`, `user_context`, `escalations` tables (`supabase/migrations/0003…0005`).
2. **Seed scripts** — convert the Power Move spreadsheet export into the `power_moves` table; seed 10 fixture users, one per §9 persona (Emily, Daniel, Sarah, James, Rachel, Michael, Olivia, Emma, Alex, Chris), with their stated day/ES/mood/Power Move.
3. **Context assembly** — `getCoachContext`.
4. **Agent + prompt** — rewrite `support-agent.ts` as `coach-agent.ts`: new instructions (§7), thinner memory (§3), context injected ahead of the call.
5. **Tools** — `recommendAction`, `viewTheScience`, `handoffToSosBuddy` (§4), replacing `search-knowledge.ts`.
6. **Guardrails** — the tier 1–3 pipeline (§6), clearly commented as provisional.
7. **RAG ingestion** — extend `ingest.ts` for structured Power Move rows (§5).
8. **Evals** — `tests/coach-scenarios.test.ts`, one test per §9 persona, asserting the "Must Not Do" and success-criteria lines against a real `agent.generate` call, same pattern as the existing `rag.test.ts`.
9. **Demo** — extend `chat.ts` with `--persona <name>` to sign in as a fixture user and load their context, so a walkthrough of all 10 scenarios is one flag away.

---

## 9. Carried-forward open items (not resolved by this plan)

- Real app deep-link routes (§4) — placeholder until confirmed.
- The guardrail trigger matrix is provisional (§6) — needs the client/clinical sign-off session that's outstanding elsewhere in the engagement before any real user is exposed to it.
- Message allowance / entitlement enforcement — out of scope (§0).
- AI SOS Buddy — no persona spec exists; this repo only ever produces a deep link to it.
