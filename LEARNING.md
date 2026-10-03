# Learning log — what changed, the concept behind it, one interview question.

## Session 1 — Writing the spec
- Changed: wrote docs/SPEC.md; no code yet.
- Concepts: soft vs hard constraints (soft = allowed but penalised, so the solver always returns a roster); Erlang C turns calls/hour + handle time into agents needed; a constraint must be written as math before it becomes code.
- Concept 2: shifts wrap around the week (mod 168) so Sunday night connects to Monday morning.
- Interview question: "Why make coverage a soft constraint with a penalty instead of a hard one, and how do you choose the penalty size?"
