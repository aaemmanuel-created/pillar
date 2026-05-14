# Next Steps — Picking Up in Claude Code

This file is a handoff note from the Cowork chat to Claude Code.

## Where to Start

1. Open a terminal in the `PILLAR by PVA/` folder.
2. Run `claude` to start Claude Code (it will pick up `CLAUDE.md` automatically).
3. Ask Claude Code to summarise the state and propose next changes — it will have full context from `CLAUDE.md`.

## Open Threads from the Cowork Conversation

These were the main themes by the end of the chat — pick up any of them in Claude Code:

- **Onboarding animations** — refinements to timings, stagger, and fade curves on step 0 and the "Time is a currency" splash.
- **Focus mode polish** — breathing ring, entry transitions, swipe-to-complete feel.
- **Hook-ordering correctness** — the `!profile` check must stay AFTER all hook declarations in `PPPPApp`. Regression risk.
- **Service worker cache discipline** — remember to bump `pva-cache-v<N>` in `sw.js` on every `index.html` change.
- **Icons** — `icon-192.png` and `icon-512.png` are canonical. The `icon-a-*`, `icon-b-*`, `icon-c-*`, `icon-new-*` PNGs in `pva-deploy/` are experiments that can be pruned whenever you want.
- **Login bonus popup** — rendering is disabled but XP logic still runs. Decide whether to re-enable, redesign, or remove entirely.

## Long-term direction (not yet built)

- **Small-group / "Club" feature** (Strava-Clubs-inspired). The accountability circle is currently capped at 3 chosen people (Strava-Beacon-inspired) — a tight, real circle. The natural next step is a *shared room*: a small group like a church cell or a discipleship triad where members can see each other's streaks, freeze stockpiles, season-of-life tags, and "Steady in the Secret Place" awards, and where prayer requests can flow in one place rather than via N⨯N messaging. This is a real long-term direction but a much bigger build (shared Firestore docs, group membership/invite flow, group-level moderation, privacy controls, and a separate UI surface). Flagged here so it doesn't get lost — do NOT build inline with smaller accountability tweaks.

## Suggested First Prompts for Claude Code

- "Read CLAUDE.md, then give me a one-page summary of the architecture and any technical debt you notice."
- "Run a local server on `pva-deploy/` and walk me through the user flow as if I were a new user."
- "Audit `index.html` for any remaining hooks-after-early-return patterns or other React rule violations."
- "Propose a cleanup of the experimental icon PNGs — list which are safe to delete."

## Things Claude Code Should NOT Do

- Do not introduce a build step, npm, or a bundler. This app runs directly from static files.
- Do not split `index.html` into separate modules/files.
- Do not change domain, DNS, or Firebase project settings without explicit confirmation.
- Do not commit or push without being asked.
