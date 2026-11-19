# Recipe Import Benchmark + Reference Agent — Build Spec for Claude Code

Nov 19, 2026 · @Nikhil

## Goal

Build a public benchmark of how reliably AI agents turn hard cooking reels (no caption, Hindi/Hinglish narration, recipe only on screen) into complete, validated recipes, plus a reference agent built to beat the baselines on it. General agents such as Meta's Muse already import recipe reels, so the product is not the pitch: the measured results are, and the agent is the system under test.

- **Why:** free general agents now import recipe reels, so a polished app no longer differentiates; what is missing is evidence of how they behave on hard reels (blank cards, invented quantities, Hinglish measures). I found no public comparison of this, but check again before claiming it in the README.
- **Who it's for now:** the Razorpay AI Builders application, which asks for "fresh work" that solves "a real, tough problem" and "prioritizes core logic over surface polish". Later, the import engine for a recipe-saver app.
- **What "done" means for v1:**
  - A deployed demo where you paste a reel link or upload a video and get a recipe card with per-field confidence.
  - A public GitHub repo with daily commits from day one.
  - A benchmark report on 50 hard reels: success rate, invented quantities, cost and time per import, for the reference agent with and without memory, against caption-only, Meta Muse, ReciMe and Deglaze. Publish the reel links, reference recipes and scoring code (not the videos).
  - A README that explains the agent loop, context design and memory design, plus a "what I prioritised and why" note.

## Scope

Build in this order and stop when time runs out; everything above the line is required for the submission.

**v1 must-haves (priority order)**

1. Benchmark: 50-reel set, reference recipes, scorer and runner (built first and the headline deliverable, so every later change is measured).
2. Media ingest: video upload, plus link fetch behind one interface; audio and keyframe extraction with ffmpeg.
3. Agent loop with four tools: caption, transcribe, read on-screen text, read comments.
4. Merge step with structured output (Zod), per-field source and confidence.
5. Validation, unit normalisation in code, and model fallback on failure.
6. Cache by video ID.
7. Memory: per-user preferences and corrections, applied to later imports.
8. Tracing: every step, model call, token count and cost.
9. Minimal web UI: paste link or upload → job status → recipe card with confidence and "estimated" badges.
10. Deploy (API + worker + UI) and the README.

**Nice to have (only after the above)**

- Sarvam speech-to-text for Hinglish audio, routed only if it wins on the eval set.
- Grocery list built from recipes.
- Razorpay UPI payment to unlock unlimited imports (test mode).

**Out of scope for v1:** mobile app, social features, meal plans, nutrition, public recipe feed, AI recipe generation.

## Architecture

&#91;embedded content: agent loop · cache, signals, merge, validate\]

Easy reels finish after the caption; hard reels loop through transcription, on-screen text and finally a stronger model, and every import ends with a validated card or explicit warnings, never a blank one.

## Agent loop and tools

The agent gathers only the signals a reel needs: cheap ones first, expensive ones only while the draft recipe still has gaps, and it stops as soon as validation passes or the budget runs out.

**Tools**

| Tool | Input | Output | Relative cost |
| --- | --- | --- | --- |
| `getCaption` | video ID / URL | caption text, description, hashtags | very low |
| `getComments` | video ID / URL | top and pinned comments (creators often post the recipe there) | low |
| `transcribe` | audio file, language hint | transcript with timestamps and detected language | medium |
| `readOnScreenText` | keyframes (scene changes, max 12) | text per frame with timestamps | medium–high |
| `mergeRecipe` | all signals so far + context (see Context engineering) | draft `Recipe` (schema below) | LLM call |
| `validateRecipe` | draft recipe | list of failed checks and missing fields | code only |

**Decision loop (orchestrator)**

1. Check the cache by video ID; if hit, return the stored recipe.
2. Call `getCaption` and `getComments`; run `mergeRecipe`; run `validateRecipe`.
3. If it passes, stop. If not, pick the next tool by what's missing:
   - quantities or steps missing → `transcribe` (if not yet done)
   - ingredient names or quantities still missing after audio → `readOnScreenText`
4. Re-merge with all signals, re-validate.
5. If still failing after all signals → one retry of `mergeRecipe` with the stronger model.
6. Stop. Fields that are still unknown are marked "estimated" or "not stated" with a warning; never left blank silently.

**Budgets and limits:** max 6 tool calls per import; max 2 merge calls; time limit 90 seconds; cost cap per import set by an environment variable. Every decision is logged with its reason ("transcribe: 3 ingredients missing quantities").

**Implementation note:** the orchestrator is plain TypeScript code with the rules above; the LLM decides only inside `mergeRecipe`. A version where the LLM chooses the next tool can be added later and compared on the eval set.

## Context engineering

Each `mergeRecipe` call gets a context assembled in code from seven labelled parts, so every part can be tested and its effect measured on the eval set.

| Part | What goes in | Why |
| --- | --- | --- |
| System prompt | Role, output must match the schema, never invent quantities, mark guesses as `inferred` | Sets the hard rules |
| Business rules | Unit list and what each Indian measure means, valid steps reference known ingredients, "a pinch" stays text | Keeps output consistent with validation |
| Examples | 2–3 short reel → recipe pairs, including one Hinglish and one no-caption case | Shows the target format on hard cases |
| Signals | Caption, comments, transcript, on-screen text, each tagged with its source and timestamps | Lets the model cite where each field came from |
| User memory | This user's preferences and past corrections (see Memory) | Personalises without retraining |
| Prior draft + failures | The last draft and the exact validation errors (on re-merge only) | Turns a retry into a targeted fix |
| Language hint | Detected language (e.g. `hi-en`) | Better Hinglish handling |

**Rules for Claude Code:** build the context in one function (`buildMergeContext`) with each part as a separate, unit-tested builder; trim long transcripts to the parts that mention ingredients or quantities; log the final token count per part in the trace.

## Memory

Memory turns a user's edits into rules for their next imports, and the eval report shows whether it actually helps.

**What is stored per user**

- **Preferences:** units (metric, cups, katori), diet (vegetarian, no egg), servings default, language.
- **Corrections:** each time the user edits a field on a recipe card, store the before/after pair and the source text it came from (e.g. "1 katori" → 150 ml).
- **Learned rules:** corrections that repeat (2+ times) are promoted to a short rule ("this user's katori = 150 ml").

**How it is used**

- Learned rules and preferences go into the merge context (only those relevant to the current reel's ingredients and units).
- Unit normalisation in code also reads the user's unit rules.

**How it stays safe and small**

- Memory is scoped strictly per user; one user's rules never reach another user's context (tested).
- Max 30 active rules per user; older, unused rules expire.
- The user can see and delete their memory in the UI.

**How it is measured:** a memory eval of 10 reels re-imported after a simulated user's corrections on earlier reels; report field accuracy with memory on vs off.

## Recipe schema, validation and units

The LLM returns this schema; code validates it and normalises units, so quantities are never left to the model alone.

```ts
const Ingredient = z.object({
  name: z.string(),
  quantity: z.number().nullable(),
  unit: z.string().nullable(),       // raw unit; normalised later in code
  note: z.string().nullable(),       // "finely chopped"
  source: z.enum(["caption", "audio", "ocr", "comment", "inferred"]),
  confidence: z.number().min(0).max(1),
});

const Recipe = z.object({
  title: z.string(),
  servings: z.number().nullable(),
  totalTimeMin: z.number().nullable(),
  ingredients: z.array(Ingredient).min(1),
  steps: z.array(z.object({ text: z.string(), ingredientRefs: z.array(z.number()) })),
  language: z.string(),              // "hi-en" for Hinglish
  sourceUrl: z.string().url(),
  warnings: z.array(z.string()),     // "quantity for salt not stated"
});
```

**Validation checks (code, not LLM)**

- At least 2 ingredients and 2 steps.
- Every ingredient is referenced by at least one step; every step reference points to a real ingredient.
- Quantities are plausible for the unit (no "40 tbsp salt"); ranges defined per unit.
- Any quantity with `source: inferred` gets a warning and confidence ≤ 0.5.
- No field blank without a warning.

**Unit normalisation (code)**

- Map raw units to a canonical set: g, kg, ml, l, tsp, tbsp, cup, piece, pinch, to taste.
- Indian measures: katori, glass, mutthi (handful) map to ml/g with a default, overridable by user memory.
- Hindi/Hinglish words: chammach → tbsp, chhota chammach → tsp, chutki → pinch, aadha → 0.5, etc. (table in `units/hi.ts`, unit-tested).

## Data model and API

Imports run as background jobs; the API creates a job and the UI polls its status.

**Tables (Postgres)**

| Table | Key columns |
| --- | --- |
| `users` | id, created\_at (simple anonymous user ID or magic-link email for v1) |
| `imports` | id, user\_id, source\_url, video\_id, status (queued / running / done / failed), recipe\_id, cost\_usd, duration\_ms, created\_at |
| `recipes` | id, video\_id (unique → the cache), recipe\_json, model\_used, created\_at |
| `import_steps` | id, import\_id, step\_no, tool, reason, input\_summary, output\_summary, tokens, cost\_usd, duration\_ms |
| `memory_items` | id, user\_id, kind (preference / correction / rule), key, value, uses, last\_used\_at |
| `recipe_edits` | id, user\_id, recipe\_id, field\_path, before, after, created\_at |
| `eval_runs` | id, git\_sha, config, metrics\_json, created\_at |

**Endpoints (Fastify)**

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/imports` | Body: `url` or uploaded file → returns `importId` |
| GET | `/imports/:id` | Status, recipe when done, and the step trace |
| PATCH | `/recipes/:id` | User edits a field → stored in `recipe_edits`, feeds memory |
| GET | `/me/memory` | List memory items |
| DELETE | `/me/memory/:id` | Delete a memory item |
| GET | `/health` | Liveness |

Media files are stored temporarily (local disk or Cloudflare R2) and deleted after the import finishes; only the structured recipe and source link are kept.

## Evals

The eval set is built first and run on every meaningful change; its numbers are the headline of the README and the submission.

**Test set (50 reels, stored as local video files so results are repeatable)**

- 20 with no caption or a caption without the recipe.
- 15 Hindi/Hinglish narration, with Indian measures.
- 10 fast cuts, recipe shown only as on-screen text.
- 5 easy ones (full recipe in caption), as a control.
- For each: a hand-written reference recipe in `evals/reference/<id>.json`.
- Plus the 10-reel memory set (see Memory).

**Metrics**

| Metric | Definition | Target |
| --- | --- | --- |
| Import success | Validation passes and ingredient recall ≥ 80% | ≥ 85% |
| Ingredient recall | Share of reference ingredients found (fuzzy name match) | ≥ 90% |
| Quantity accuracy | Share of quantities within ±10% of reference after normalisation | ≥ 80% |
| Invented quantities | Quantities with no supporting signal and not marked `inferred` | 0 |
| Blank imports | Imports with no ingredients | 0 |
| Cost per import | Total model + transcription cost | report it |
| Time per import | Wall-clock seconds | report median and p90 |
| Memory lift | Field accuracy on the memory set, memory on vs off | report it |

**Baselines to report:** caption-only pipeline, full agent, full agent + memory, plus apps and agents run by hand on the same reels: Meta Muse, ReciMe and Deglaze (a blank or incomplete import counts as a miss). Record the app version and the date of each hand run.

**How to run:** `pnpm eval` runs the set, writes results to `eval_runs` and prints a Markdown table; `pnpm eval --only hinglish` runs one slice.

## Tech stack and repo layout

TypeScript end to end in one pnpm monorepo; every external service sits behind an interface so it can be swapped after evals.

| Layer | Choice |
| --- | --- |
| Runtime | Node.js 22 LTS, TypeScript (strict), pnpm workspaces |
| API | Fastify + Zod |
| Jobs | BullMQ + Redis |
| Database | PostgreSQL with Drizzle ORM (Supabase or Neon is fine) |
| Media | ffmpeg (audio extraction, scene-change keyframes) |
| Speech-to-text | Whisper-style API behind `transcribe(audio, langHint)`; Sarvam Saaras as an option for Hindi/Hinglish |
| On-screen text | Vision-capable LLM on keyframes behind `readOnScreenText(frames)` |
| LLM | Vercel AI SDK with structured output (Zod); a cheap default model and a stronger fallback, both set by env vars |
| Tracing | Langfuse, or the `import_steps` table first |
| UI | Next.js (App Router), minimal |
| Tests | Vitest |
| Deploy | API + worker on Railway or Fly.io (needs ffmpeg in the image); UI on Vercel |

**Repo layout**

```text
recipe-import-agent/
  apps/
    api/          # Fastify routes, job enqueue
    worker/       # BullMQ worker running the orchestrator
    web/          # Next.js UI
  packages/
    agent/        # orchestrator, tools, buildMergeContext, validateRecipe
    media/        # ffmpeg helpers, fetchers (upload, link)
    units/        # normalisation tables (en, hi) + tests
    memory/       # memory store, rule promotion, retrieval
    db/           # Drizzle schema, migrations
    shared/       # Zod schemas, types, config
  evals/
    videos/       # test reels (gitignored or LFS)
    reference/    # hand-written reference recipes (JSON)
    run.ts        # eval runner
  docs/
    PRIORITIES.md # what was built first, what was cut, and why
  README.md
```

## Three-week build plan

Each milestone ends with a passing acceptance check and a commit; evals start in week 1 so every later step is measured.

**Week 1: foundations and baseline**

- [ ] Repo, monorepo setup, CI (lint, typecheck, Vitest), public on GitHub on day 1.
- [ ] Zod schemas, unit normalisation tables (en + hi) with tests.
- [ ] Media package: upload ingest, ffmpeg audio extraction, scene-change keyframes (max 12).
- [ ] Collect 50 test reels and write the reference recipes (start with 20, finish in week 2).
- [ ] Eval runner and scorer with metrics from the Evals section.
- [ ] Caption-only pipeline + `mergeRecipe` + `validateRecipe`.

* **Accept:** `pnpm eval` runs end to end and prints a baseline table for the caption-only pipeline.

**Week 2: the agent**

- [ ] Tools: `getComments`, `transcribe`, `readOnScreenText`, each behind an interface with tests on fixtures.
- [ ] Orchestrator with decision rules, budgets, stop conditions and logged reasons.
- [ ] `buildMergeContext` with the seven parts; re-merge with validation errors; stronger-model fallback.
- [ ] Cache by video ID.
- [ ] Jobs: BullMQ worker, `POST /imports`, `GET /imports/:id` with step trace.
- [ ] Finish all 50 reference recipes; run Muse, ReciMe and Deglaze by hand on the same reels and log the results.

* **Accept:** full agent beats caption-only on import success and recall; 0 blank imports; results saved in `eval_runs`.

**Week 3: memory, UI, ship**

- [ ] Memory: preferences, corrections from `PATCH /recipes/:id`, rule promotion, retrieval into context, per-user isolation test.
- [ ] Memory eval (10 reels), memory on vs off.
- [ ] Minimal UI: paste or upload, job status, recipe card with confidence and "estimated" badges, edit fields, view/delete memory.
- [ ] Link fetch for public reel URLs behind the same interface (after the terms-of-service check in Risks).
- [ ] Deploy API, worker and UI; record a 2-minute demo video.
- [ ] README, eval report and `docs/PRIORITIES.md`.

* **Accept:** live demo works on 3 fresh reels; README shows final eval table; submission checklist complete.

## Working rules for Claude Code

Nikhil reviews and commits every change; Claude Code works in small, tested steps and keeps the docs honest.

- **Small commits, every day:** one feature or fix per commit, a clear message (`feat(agent): add transcribe tool`). Razorpay will look at the GitHub activity graph.
- **Never push:** Claude Code does not run `git push`, create or change remotes, open pull requests or touch GitHub. Nikhil pushes from his own terminal after reviewing the commits.
- **Tests with every module:** unit tests for units, validation, context builders and memory isolation; fixture-based tests for tools (no live API calls in unit tests).
- **Measure before claiming:** after any change to prompts, tools or the orchestrator, run `pnpm eval` and record the result; never write a number in the README that the eval didn't produce.
- **Secrets:** API keys only in `.env` (never committed); provide `.env.example`.
- **Interfaces first:** every external service (speech-to-text, vision, LLM, fetchers) behind a typed interface with a fake for tests.
- **Keep `docs/PRIORITIES.md` updated:** each week, note what was built, what was cut and why. This is part of the submission.
- **Core logic over polish:** don't spend time on UI styling until week 3's acceptance check passes.
- **Ask before:** adding a paid service, changing the schema, or downloading content from a platform in a way the Risks section hasn't cleared.

## Submission checklist

The [AI Builders form](https://razorpay.typeform.com/to/Aj64eENJ) asks for a live link (or a 2-minute recording) and a public source-code link; the [AI Builders page](https://razorpay.com/ai-builders/) adds that they look at real projects and GitHub links.

- [ ] Live demo URL that works on a fresh reel (or a 2-minute screen recording).
- [ ] Public GitHub repo with daily commits from day one.
- [ ] README, top to bottom: one-line pitch, demo GIF, eval table, agent loop diagram, context design, memory design, how to run.
- [ ] Benchmark report: success rate, recall, quantity accuracy, invented quantities, cost and time, memory lift, versus caption-only, Muse and ReciMe; plus the published reference set (reel links and JSON, not the videos) and scoring code.
- [ ] `docs/PRIORITIES.md`: what was built first, what was cut, and why.
- [ ] A short note in the form that leads with the benchmark finding, then links the work to what the page says: an agent loop, context engineering, and memory.
- [ ] Read the remaining form questions (3–6) before submitting and prepare answers.

## Risks and open questions

The biggest open question is how to fetch reels legally; v1 works on uploaded videos so nothing blocks the build.

| Risk | Why it matters | Mitigation |
| --- | --- | --- |
| Platform terms of service | Downloading Instagram/TikTok video may break their terms | v1 and evals use uploaded files; check what's allowed before enabling link fetch; delete media after processing |
| Cost per import | Transcription, vision and LLM calls add up | Cache by video ID; cheap signals first; stronger model only on failure; cost cap per import |
| Hinglish accuracy | The core differentiator | Dedicated eval slice; test Sarvam vs Whisper-style model and route by result |
| Not better than Muse or ReciMe | Muse is free with huge distribution, so the agent alone is not the pitch | The benchmark is the pitch; report honestly where the reference agent wins and loses |
| Scope creep | Three weeks slip | Build in the scope order; stop at the line |
| Muse access | Muse launched in the US first, so it may not be usable from India | Check availability early; if unavailable, run ChatGPT and Gemini by hand instead and state that Muse was not tested |

**Open questions**

- [ ] Which reel-fetching method is allowed for public links?
- [ ] Which default and fallback models give the best accuracy for the cost?
- [ ] Does a video-capable model beat separate transcript + on-screen text on the eval set?
- [ ] Is Sarvam pay-as-you-go pricing workable for the demo?
