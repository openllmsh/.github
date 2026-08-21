# AGENT MODEL ASSIGNMENT

## MODEL ASSIGNMENT — resolved at runtime, not pre-filled

Do NOT use hard-coded model assignments. Before Stage 1, resolve every role's
model interactively:

1. **Fetch available models** — call the OpenLLM MCP tool
   `mcp__openllm__api_v1Models_list` (GET /v1/models) and collect the full
   list of model IDs. If the MCP call fails, say so explicitly and fall back
   to asking the user to name models manually; do not invent a catalog.
2. **Check subscription quota before suggesting** — call
   `mcp__openllm__api_stats_user` (GET /stats, no keyId) and read the
   `inferred_subscription_usage` / per-provider usage snapshots it returns.
   Subscription models (claude_code, chatgpt, kimi_code, grok, cursor) draw on
   vendor quota windows; each provider's tightest window (`percent_used` per
   window) says whether it can serve more work right now. Rules:
   - A provider whose tightest window is at/near 100% (or whose snapshot
     status is `rejected`) is OUT of quota — its models must NOT be offered
     as a suggestion (still listable if the user asks, never recommended).
   - A provider with NO usage data (not connected, never refreshed) is
     quota-UNKNOWN — mention the uncertainty when suggesting its models.
   - API-key (metered) providers have no quota windows; treat them as always
     available (they bill per token instead).
   If /stats is unavailable, say so and treat everything as quota-unknown —
   never silently assume headroom.
3. **Suggest variety, not one model everywhere** — recommendations across
   stages should span different models/providers rather than defaulting every
   role to your strongest model. Calibrate to the work, picking from the
   fetched list by capability class:
   - top-tier (framing, judgment, synthesis) → Planner and Supervisor
   - mid-tier (execution that needs intelligence) → Implementer stages
   - small/fast tier → trivial or repetitive steps, e.g. mechanical Stage 4
     fixes that are pure application of findings
4. **Ask the user per stage** — for EACH of the six roles below, ask the user
   interactively in Claude Code (e.g. via AskUserQuestion) which model from
   the fetched list should run it. Present the choices as selectable options,
   grouped by provider if the list is long, and include your own recommended
   pick first for each role. One question per stage so each can differ:

   | Stage | Role                    |
   |-------|-------------------------|
   | 1     | Planner                 |
   | 2     | Implementer             |
   | 3     | Reviewer (round 1)      |
   | 4     | Implementer (revisions) |
   | 5     | Reviewer (round 2)      |
   | 6     | Supervisor              |

5. **Record the resolutions** — echo back the final assignment table (stage,
   role, chosen model ID) before starting Stage 1, then proceed.
6. When spawning sub-agents for stages, pass the chosen model ID explicitly
   (the gateway's `<provider>/<model_name>` format; fuzzy resolution applies
   if underspecified).

Rules:
- Stage 5's reviewer must be a different model than Stage 3's where possible;
  if the user picks the same one, it must still run as a fresh context with no
  memory of authoring the fixes — surface this to the user at selection time.
- The Supervisor should not be the same model as the Implementer. If the user
  picks the same one, warn them at selection time; if they confirm, the
  Supervisor runs in a fresh context and states this in its verdict.
- Every stage output must be prefixed with a header naming the stage, the role,
  and the model that produced it. Example:
  `[STAGE 3 — REVIEWER (round 1) — <chosen-model-id>]`
- If a stage cannot be routed to its assigned model, say so explicitly rather
  than silently substituting.

# TASK

You will build a landing page for Coffee&Code Philadelphia using a multi-agent
process. Do NOT write code until Stage 1 is complete.

If your environment supports spawning sub-agents, spawn a distinct agent for each
role below. If it does not, adopt each role in a separate, clearly labeled pass —
and when acting as a reviewer, critique the work as if you had not written it.

---

## STAGE 1 — PLANNER

Spawn: Planner agent.

Produce a build plan before any code exists:
- Component inventory (every section, its states, its data shape)
- Layout system: grid strategy, breakpoints, spacing scale
- Design direction: palette with hex values, type scale, motion philosophy —
  and one paragraph justifying WHY this direction fits this specific meetup
- State model: what's dynamic, where it lives, how it initializes
- Risk list: the three things most likely to look broken or cheap, and the
  mitigation for each

Output the plan as a document. Do not write implementation code in this stage.

---

## STAGE 2 — IMPLEMENTER

Spawn: Implementer agent. Input: the Stage 1 plan.

Build the full page per the plan and the SPEC below. Follow the plan; if you
must deviate, log the deviation and the reason.

---

## STAGE 3 — CODE REVIEW (round 1)

Spawn: Reviewer agent. It receives the plan and the code, and reviews against
this rubric. Score each 1–5:

1. Spec compliance — is every required element present and functional?
2. Plan adherence — does the build match Stage 1, or did it drift?
3. Correctness — does the JS actually run? Countdown math, validation edge
   cases, accordion state, scroll listeners, off-by-ones.
4. Responsive integrity — 320px, 768px, 1440px. Overflow, tap targets, reflow.
5. Accessibility — semantic HTML, keyboard traversal, focus visibility,
   contrast ratios, ARIA on the accordion and nav.
6. Craft — does this look designed or does it look generated? Call out
   defaults-as-decisions: unmodified system fonts, generic shadows, purple-blue
   gradients, evenly-sized cards with no hierarchy.
7. Copy quality — does it sound like a human organizer or like SaaS marketing?

Output: numbered findings, each tagged BLOCKER / SHOULD-FIX / NIT, each with
the specific location and the specific fix. Vague findings ("improve spacing")
are rejected — be actionable.

---

## STAGE 4 — IMPLEMENT CHANGES

Spawn: Implementer agent. Input: code + review findings.

Fix all BLOCKERs and SHOULD-FIXes. For any finding you decline, state why.
Output a changelog mapping each finding to what changed.

---

## STAGE 5 — CODE REVIEW (round 2)

Spawn: a FRESH Reviewer agent — not the round-1 reviewer. Re-run the full
rubric on the revised code. Verify prior findings are actually resolved rather
than superficially patched. Flag regressions introduced by the fixes.

---

## STAGE 6 — SUPERVISOR

Spawn: Supervisor agent. It reads the plan, both reviews, and the current code,
and issues one of two verdicts:

**SIGN-OFF** — requires all of: zero open BLOCKERs, no rubric category below 4,
no unexplained plan drift. Output the final file and a one-paragraph
justification of the verdict.

**REJECT** — output the remaining gaps and return to Stage 4. Repeat
Stage 4 → 5 → 6 until sign-off.

Hard cap: 3 revision cycles. If sign-off is not reached by cycle 3, the
Supervisor must ship the best available version and publish a "Known Defects"
list stating exactly what remains unresolved and why it wasn't fixed. Do not
loop silently and do not claim sign-off you didn't earn.

Show your work for every stage. The intermediate artifacts — plan, findings,
changelogs, verdicts — are part of the deliverable, not scaffolding to hide.

---

# SPEC

Coffee&Code Philadelphia is a twice-monthly meetup where developers, designers,
and tech-curious folks gather in rotating Philly coffee shops to work on side
projects, run lightning talks, and drink good coffee. Founded 2021, ~140
regulars, free to attend.

Next session: Saturday, September 13, 2026, 9:00 AM – 12:30 PM, at a roastery
in Fishtown. Capacity 45.

Include:
- A hero that makes the coffee/code duality obvious without being corny
- Live countdown to the next session
- "What actually happens" — the three-hour format (open co-working block, two
  10-minute lightning talks, unstructured hangout)
- Lightning talk lineup: 2 confirmed speakers with photo, one-line bio, talk
  title, plus one "slot open — pitch us" card
- Venue rotation showing the last three and next three locations, with
  neighborhood tags (Fishtown, Old City, West Philly, Passyunk)
- RSVP form with inline validation and a spots-remaining indicator
- Community stats strip (sessions held, talks given, projects shipped, pounds
  of coffee consumed)
- Sponsor row with placeholder logos
- FAQ accordion — at minimum: cost, skill level, laptop required, do I have to
  talk to anyone, is there parking
- Sticky nav that changes state on scroll, with RSVP as the persistent action
- Footer with code of conduct link, Discord, GitHub, and newsletter capture

Requirements:
- One self-contained HTML file, no build step
- Fully responsive, mobile-first
- Every interactive element gets a considered hover/focus/active state
- Real placeholder copy, not lorem ipsum — write like the meetup organizer