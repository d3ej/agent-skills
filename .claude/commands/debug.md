---
description: Systematic root-cause debugging — reproduce, localize, fix, guard, verify
---

Invoke the agent-skills:debugging-and-error-recovery skill.

Debug the current issue systematically:

1. **Reproduce** — Confirm the failure with a minimal, reliable reproduction case
2. **Localize** — Narrow the failure to the smallest possible scope (file, function, line)
3. **Reduce** — Strip away everything not involved in the failure
4. **Hypothesize** — State one hypothesis at a time; don't fix multiple things at once
5. **Fix** — Apply the targeted fix; explain why this is the root cause, not a symptom
6. **Guard** — Write a regression test so this exact failure can't silently recur
7. **Verify** — Run the full test suite; confirm no regressions

If the failure involves external systems (network, database, third-party APIs), check those boundaries first — most production bugs live at integration points.

Stop and ask for more context if the reproduction case is unclear.
