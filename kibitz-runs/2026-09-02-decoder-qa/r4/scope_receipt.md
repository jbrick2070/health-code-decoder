# Scope receipt -- explicit partial campaign

Requested by the user on 2026-09-02: "ask kibitz codex to QA, and cursor and sonnet for good
measure." This is a single QA round on built code, not a four-round plan-hardening arc.

- Rounds run: r4 only (residual defects / convergence prompt, the closest fit for QA of a build).
- Rounds NOT run: r1, r2, r3. No earlier Kibitz artifacts exist for this repo.
- Input: docs/qa-brief.md (copied to input.md by the script; SHA-256 recorded below after copy).
- Driver: claude (Claude Fable 5.1 in Claude Code). The `claude` CLI lane (Sonnet) was included
  at the user's explicit request even though it shares the driver's model family; its review is
  weighted accordingly during grounding.
- Reviewer lanes: codex, cursor, claude.
- Anchor review: driver_anchor.md, written before fan-out.

- input.md SHA-256: b95db4620a5db70d499b14e739c563bd79a2b39d8c3220f9a3982c1029a7ec6a
