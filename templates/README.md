# <PROJECT-NAME>

> <One-sentence pitch — what this is, in plain language. The synthesizer fills this from `docs/foundation/PITCH.md`.>

## What it is

<2–3 sentences. Grounded in the pitch and the council's findings. No marketing claims that no seat raised.>

## Who it's for

<Concrete user description. Role, context, what they were doing five minutes before they encountered this. Grounded in the user-advocate's draft.>

## Why now

<Answer from the investor-advocate's "Why now" section. If they could not answer, write "Open — see DECISION-XXX in docs/foundation/OPEN-DECISIONS.md".>

## Architecture snapshot

<3–7 major components, plain prose, with the boundary between each named. Grounded in the architect's draft. Examples:>

- **<Component A>** — <responsibility>; owns <data shape / contract>; talks to <Component B> via <sync / async / batch>.
- **<Component B>** — <responsibility>; ...

<Optional: name the first scaling wall the architect identified, so future work knows what to watch.>

## Coding guidelines

<Short list — only what the council named as load-bearing for *this* project. No generic best-practice. Each line is one rule, one reason. Examples:>

- **<Rule>** — <reason, sourced from the seat that argued for it: e.g. "All model calls go through `lib/llm/` so the cost / latency budget can be enforced centrally (architect)">.
- **<Rule>** — <reason>.
- **<Rule>** — <reason>.

## Hard decisions

<Items where the council converged. One-line each, sourced.>

- **<Decision>** — <one line>. *(seat: architect / investor-advocate / ...)*
- **<Decision>** — <one line>. *(seat: ...)*

## Open risks

<Top 3 from security-auditor + skeptic. Each with the trigger that would escalate it.>

- **<Risk>** — <attacker capability or failure condition>. *Trigger to escalate:* <what observable change makes this urgent>. *(seat: security-auditor / skeptic)*
- **<Risk>** — ...
- **<Risk>** — ...

## Kill criteria

<From the skeptic. If the project should be abandoned or pivoted under condition X, name X. If none apply, write "None named by the council — see DECISION-XXX in docs/foundation/OPEN-DECISIONS.md".>

- <Condition under which this project should stop or change direction>.

## How agents work here

This project follows the [`agentic-ai-features`](https://github.com/moinsen-dev/agentic-ai-features) plugin conventions.

Before any feature work, an agent reads:

1. `CLAUDE.md` — the project spine (hard rules, human gates, task format).
2. This `README.md` — the project anchor (what / who / why / hard decisions).
3. `docs/foundation/PERSPECTIVES.md` — the council's reasoning, if more context is needed.
4. `docs/foundation/OPEN-DECISIONS.md` — items still to be resolved; **the implementation skills refuse to start while any item here is unchecked.**

For a single task, run `/agentic-ai-features:implement-task`. For a multi-task plan, run `/agentic-ai-features:task-loop`.

## Foundation outputs

This README and the documents under `docs/foundation/` were authored by `/agentic-ai-features:foundation`:

- `docs/foundation/PITCH.md` — the user's pitch, captured verbatim.
- `docs/foundation/perspectives/` — five independent council drafts.
- `docs/foundation/PERSPECTIVES.md` — consolidated council protocol (convergence, genuine disagreement, top concerns per seat).
- `docs/foundation/OPEN-DECISIONS.md` — gating checklist.

If the council got something wrong, edit the README directly and add a note in `PERSPECTIVES.md` so future agents see the correction.
