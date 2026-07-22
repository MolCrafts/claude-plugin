---
name: spec
description: Requirement → structured spec + acceptance under `.claude/specs/`, then auto grill (spec-audit) → clean auto impl-all. Tier C: 落盘/写 spec/spec this (slash optional); never silent from discuss/grill.
argument-hint: "<feature description>"
---

> **Codex:** Read `../CODEX.md` before executing this shared workflow. Claude Code follows the workflow directly.

# /mol:spec — Specification Generator

Read CLAUDE.md → parse `mol_project:` (`$META`); else emit adoption hint and stop. Resolve `$META.specs_path` (default `.claude/specs/`); create dir if missing.

Produces `<slug>.md` (design) + `<slug>.acceptance.md` (binding "done" contract per `plugins/mol/rules/evaluator-protocol.md`). After persist, **always** stress-tests the written design via `/mol:grilling` (spec-audit). `/mol:impl` refuses without both files; deletes both when done. Specs live under `.claude/specs/` — never `docs/` or `.claude/notes/`.

## Procedure

### 1. Parse & research

Derive kebab-case slug. State in one sentence.

**Conflict check** — read every existing spec under `$META.specs_path`:
- **Duplicate** → tell user, stop.
- **Supersede/refine** → update old spec in place. Pass old body to `spec-writer` as `conflict_decision: supersede:<slug>`.
- **Independent** → safe to create.

**Domain & placement** — run in parallel where possible:
- Physics + `$META.science.required: true` → delegate to `scientist` for equations/references. Capture verbatim.
- Glob for relevant files; flag new public API, cross-layer deps.
- Consult `librarian` (reads `.claude/notes/architecture.md` + runs a targeted source scan). **Mandatory for every spec — drafting never starts without this codebase scan.** Returns reuse candidates tagged `reuse` / `generalize` / `pattern` + recommended placement; `spec-writer` must resolve each candidate in the Design's Reuse decision — a spec that reimplements an existing capability instead of reusing or generalizing it is invalid. If `librarian` returns `stale: true`: invoke `architect` (inventory mode) → `/mol:map` (user-confirmed) → re-consult `librarian`. If user defers `/mol:map`, note "blueprint refresh deferred" and proceed.

### 2. Delegate drafting to `spec-writer`

Invoke `spec-writer` with: `request`, `slug`, `scope_layer`, `scientist_output`, `conflict_decision`, `interaction_points`, `librarian_report`.

`spec-writer` drafts spec body (Summary / Domain basis / Design / Files / Tasks / Testing / Out of scope) + acceptance criteria, self-validates, returns markdown **without writing to disk**.

Branch on `Status:`:
- `ok` → proceed to Step 3.
- `blocked` → surface failed items. User relaxes or refines; re-invoke from Step 2.
- `split-needed` → large-spec split rule fired (`plugins/mol/rules/large-spec-split.md`). **Don't prompt.** Re-invoke `spec-writer` once per sub-slug in chain order with `slug: <base>-NN-<phase>`, `request: sub-scope`, `conflict_decision: independent`. Collect full chain; proceed to Step 3.

### 3. Persist & show

Persist immediately — **no approval prompt, no waiting**:
1. Write `{$META.specs_path}{slug}.md` with `status: approved` (overwrite for supersede; bump `revised`). Chain → one per sub-spec; no parent file.
2. Write `{$META.specs_path}{slug}.acceptance.md`. Chain → one per sub-spec.
3. Update `{$META.specs_path}INDEX.md`:
   ```
   - [{slug}]({slug}.md) — <one-line summary> [approved]
   ```
   Chain → one entry per sub-spec. Supersede → update in place.

Then show spec body + acceptance exactly as written. Call out: librarian reuse candidates and how the Design's Reuse decision resolved each (first), criteria from Testing strategy, UI checks recorded in the spec body's **UI verification** section (never acceptance criteria), items deliberately not turned into criteria, supersede diff if any.

Tell the user: *"persisted — now entering `/mol:grilling` (spec-audit) before ready-for-impl."* Spec is on disk as `approved`, but Step 3.5 may supersede it.

Post-persist tweaks the user requests **before** grilling finishes → apply in place. Material design changes mid-flight → fold into the audit / supersede path.

### 3.5 Audit grill → supersede if needed

**Mandatory.** Auto-invoke `/mol:grilling` in **spec-audit** mode with: `slug`, paths to the written spec + acceptance, and a short pointer that the Design/Tasks/acceptance surface is under test. Use the Skill tool (Claude) or read-and-execute `../grilling/SKILL.md` (Codex). Never call user-only `/mol:grill`.

Chain of sub-specs → grill each sub-spec in chain order (or the first incomplete one if resuming); do not skip.

When `/mol:grilling` returns:

| `audit_result` | Action |
|---|---|
| `clean` | Optional: set non-binding frontmatter `grilled: true` on the spec file(s). Proceed to Step 4. |
| `supersede_needed` | Re-invoke `spec-writer` with `conflict_decision: supersede:<slug>`, `request` = original request + Decisions log + supersede payload. On `Status: ok`, overwrite both files + INDEX (same as Step 3). Set `grilled: true` after successful supersede. If `blocked`, surface items and stop with files left as last good persist. |
| redirected / under-formed | Leave last persisted files as `approved`. Surface Open list + reason. Do **not** delete. User may re-run `/mol:grilling mode:spec-audit <slug>` or re-spec. |

**Grilling remains read-only** — this skill owns all supersede writes.

### 4. Report

Only after Step 3.5 settles (clean, supersede applied, or explicit park on under-formed):

Spec path(s), task count, criteria count by `type`, grill outcome (`clean` / `superseded` / `parked`).

When outcome is `clean` or `superseded`: **auto-invoke `/mol:impl-all`** with the slug or chain base (no prompt). When `parked` / redirected: report only — do not implement.

End with one-line summary after impl-all returns (or after park).

## Guardrails

- **Chinese input** → `spec-writer` produces body in Chinese; frontmatter keys, INDEX entry, and Tasks verb-prefixes stay English for downstream tooling.
- **Drafting is delegated** to `spec-writer` to keep parent context free for conversation. Triage, persistence, INDEX upkeep, and post-persist supersede stay here; first persist is automatic — never wait for approval. See `plugins/mol/rules/agent-design.md`.
- **UI-runtime checks never become acceptance criteria.** They live in the spec body's **UI verification** section (non-binding); `/mol:web` verifies them ad hoc and they never park a spec at `code-complete`.
- **Always auto-invoke `/mol:grilling` after persist.** Do not hand the user a "ready for impl" F2 before the audit settles.
- **After grill settles clean (or supersede applied clean):** auto-invoke `/mol:impl-all <slug-or-prefix>` so implementation runs end-to-end without a second human kick. Parked grill redirect → do not impl.
- **Spec lifecycle** (`draft` → `approved` → `in-progress` → `code-complete` → `done`) is defined in `plugins/mol/rules/evaluator-protocol.md`. `grilled: true` is advisory metadata, not a lifecycle state.
