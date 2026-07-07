# Proposal: Grafting Hermes-Agent's Skill Self-Authoring onto Myco Vault

**Status:** Research proposal — for Shane's review. No production code changes. Nothing here is implemented.
**Court:** CC (research/proposal only; Codex gate not yet engaged — see §4 for what *would* engage it)
**Source studied:** `NousResearch/hermes-agent` @ `8301654` (2026-07-06), read-only clone in session scratch.

> **Environment caveat.** This session ran in a remote container scoped to
> `mycolove-mission-control` only, so `vault_writer.py`, `myco-vault`,
> `exec-approvals.json`, and `AGENTS.md` were **not readable here**. The
> Hermes side of this document is grounded in the actual source (file:line
> citations throughout). The Myco side maps onto the write path as described
> in the task brief; every load-bearing assumption about `vault_writer.py`
> is tagged **[ASSUMPTION]** and should be verified against the real code
> before any implementation ticket is cut. The task asked for the deliverable
> at `/Users/myco/scratch/`; that path doesn't exist in this container, so
> it's delivered on the `claude/hermes-skill-graft-proposal-h9x3kh` review
> branch instead — still nowhere near production.

---

## 0. How Hermes' mechanism actually works (grounded summary)

Hermes treats skills as **procedural memory**: markdown packages the agent
itself authors, distinct from declarative memory (MEMORY.md/USER.md). Three
subsystems cooperate:

### 0.1 The write tool — `skill_manage` (`tools/skill_manager_tool.py`)

One tool, six actions: `create`, `edit` (full SKILL.md rewrite), `patch`
(fuzzy find-and-replace), `delete`, `write_file`, `remove_file`. A skill is a
directory:

```
~/.hermes/skills/<category?>/<name>/
├── SKILL.md            # YAML frontmatter + markdown body (required)
├── references/         # session detail, condensed knowledge banks
├── templates/          # copy-and-modify starter files
├── scripts/            # re-runnable verification/probe scripts
└── assets/
```

Validation is strict and mechanical (no model judgment involved):

- **Name**: `^[a-z0-9][a-z0-9._-]*$`, ≤64 chars (`skill_manager_tool.py:459`).
- **Frontmatter**: must open with `---`, parse as a YAML mapping, contain
  `name` + `description` (≤1024 chars), and have a non-empty body
  (`_validate_frontmatter`, `:508`).
- **Size caps**: 100k chars for SKILL.md, 1 MiB per supporting file (`:455`).
- **Path discipline**: supporting files only under the four allowed subdirs;
  traversal (`..`) rejected before any allow-listing; resolved paths must
  stay inside the skill dir (`_validate_file_path`, `_resolve_skill_target`).
- **Atomic writes**: tempfile + `os.replace`, with content backup and
  rollback if a post-write scan blocks (`_atomic_write_text`, `:740`).
- **Delete defense-in-depth**: never `rmtree` anything that isn't strictly
  inside a known skills root, never a root itself, never through a
  symlink/junction (`_validate_delete_target`, `:193` — a direct port of a
  real Kilo Code data-loss incident).

### 0.2 The autonomous author — the background review fork (`agent/background_review.py`)

After a turn, the agent may spawn a **daemon-thread fork of itself** that
replays the conversation snapshot and asks "should any skill be saved or
updated?" The fork is aggressively sandboxed:

- **Tool whitelist**: only memory + skills toolsets; everything else denied
  at dispatch (`background_review.py:791-804`).
- **Persistence isolation**: the fork shares the parent's session id (for
  prompt-cache warmth) but has DB writes hard-disabled, so its harness
  prompt never leaks into the real session and gets re-read as a user
  instruction later (`:723` — they hit this bug; it's documented inline).
- **No compression, capped at 16 iterations, auto-denies any dangerous
  command approval** rather than blocking on an interactive prompt.
- **The review prompt is the policy** (`_SKILL_REVIEW_PROMPT`, `:171`):
  patch a skill that was actually loaded this session first; then patch an
  existing umbrella; then add a support file; only create a *new class-level*
  skill as a last resort. It carries an explicit **do-not-capture list**:
  environment-dependent failures, negative tool claims ("X is broken" —
  these harden into self-imposed refusals), transient errors that resolved,
  and one-off task narratives.

### 0.3 The guards — provenance + grounding (`tools/skill_provenance.py` and guards in the write tool)

This is the part the task specifically asked about (item 3):

1. **Origin tagging.** A `ContextVar` (`skill_write_origin`) defaults to
   `"foreground"`; the review fork sets it to `"background_review"` before
   its tool loop (`skill_provenance.py:37-78`). Every guard keys off
   `is_background_review()` — the *same tool* behaves differently depending
   on who's calling.

2. **Read-before-write (the grounding guard).** When `skill_view` returns
   file content to the model, it calls
   `mark_background_review_skill_read(path)` which records the resolved path
   in a per-context set (`skill_manager_tool.py:56-91`). Every mutating
   action (`edit`/`patch`/`write_file`/`remove_file`) then refuses — with an
   instructive error telling the model to go read the file and retry —
   unless the exact target path was read *in this review turn*
   (`_background_review_read_before_write_guard`, `:366`). The fork
   physically cannot rewrite content it only inferred from the transcript.

3. **Ownership restriction.** The autonomous fork may only write
   curator-owned sediment: pinned, bundled, hub-installed, and
   external-directory skills are all refused
   (`_background_review_write_guard`, `:281`). Foreground user-directed
   edits to those same skills are allowed — autonomy is what narrows the
   write surface, not the tool.

4. **Fail-closed deletes.** During autonomous curation a delete is only
   legal as a *verified consolidation*: `absorbed_into=<umbrella>` where the
   umbrella already exists on disk; a bare prune is refused
   (`_curator_consolidation_delete_guard`, `:405`). Even then the delete
   routes to a recoverable archive (`.archive/` + `hermes curator restore`),
   never `rmtree` (`_delete_skill`, `:1079`).

5. **Provenance marking for later lifecycle.** Only skills *created by the
   background fork* get `mark_agent_created` (`skill_manage`, `:1387`);
   the curator may only auto-consolidate/prune those. Skills a user asked a
   foreground agent to write belong to the user, forever.

6. **Optional human approval gate** (`tools/write_approval.py`). A
   per-subsystem `write_approval: true` makes every skill write **stage** to
   `~/.hermes/pending/skills/<id>.json` with a one-line gist instead of
   committing; the user approves/rejects out-of-band (`/skills pending`) and
   approval replays the exact payload through a bypass ContextVar. Staging
   is mandatory for background-origin writes (a daemon thread can't block on
   a prompt). Off by default in Hermes; **I propose it be ON and
   non-optional for Myco v1** — see §3.

7. **Lifecycle telemetry lives in a sidecar**, not frontmatter
   (`tools/skill_usage.py`): `.usage.json` keyed by skill name holds usage
   counters, `patch_count`, state (`active`/`stale`/`archived`), `pinned`,
   and the `agent_created` provenance bit. Deliberate design note in their
   docstring: keeps operational churn out of user-authored content.

### 0.4 Honcho (reference only, out of scope for v1)

`plugins/memory/honcho/` is a cross-session *user-modeling* memory provider
(dialectic reasoning over who the user is), injected into the user message at
API-call time to preserve prompt caching. Notably, the review fork runs with
`skip_memory=True` specifically so its harness prompt never contaminates the
Honcho namespace. Nothing in the skill graft depends on it; the only lesson
worth importing is that isolation pattern: **autonomous forks must have zero
side effects on stores they weren't spawned to write.**

---

## 1. Mapping `skill_manage` onto `vault_writer.py`'s write path

**[ASSUMPTION]** `vault_writer.py` (myco-scripts) is the single choke point
through which agent-originated writes to myco-vault flow, and it already:
(a) knows the vault root and refuses paths outside it, (b) understands the
entity/type model via frontmatter (`type:` key), and (c) validates
frontmatter per type before writing. If any of those aren't true, they are
prerequisites, not part of this graft.

The graft is **not** "port skill_manager_tool.py." It's: add one note type
and a small action vocabulary to the existing write path, and copy Hermes'
guard *semantics* (not its code) into it.

| Hermes `skill_manage` | Myco Vault equivalent | Notes |
|---|---|---|
| `create` | `vault_writer` create-note with `type: skill`, under a fixed subtree (e.g. `Skills/`) | Reuse existing type-schema validation; add the Skill schema (§2). Name-collision check across the subtree, like `_find_skill`. |
| `edit` (full rewrite) | Existing full-note write | Must re-validate frontmatter post-write, same as create. |
| `patch` (fuzzy find/replace) | **New, and worth having.** A targeted old→new replace on note body | Hermes found full rewrites are how agents silently drop content; `patch` is their "preferred for fixes" action. Even an exact-match (non-fuzzy) v1 is fine. Refuse a patch that breaks frontmatter (`skill_manager_tool.py:981`). |
| `delete` | **Do not expose in v1.** Status demotion instead: set `status: archived` in frontmatter | Hermes needed three layers of delete defense plus a recoverable archive. The vault is git-backed and flat markdown — archival-by-status costs nothing and deletes are exactly one of Shane's hard gates (§4). |
| `write_file` / `remove_file` | Probably **skip for v1** | Hermes needs support-file dirs because skills carry scripts/templates. A Vault Skill note can start as a single note; wikilinks to ordinary vault notes cover the `references/` use case idiomatically. Revisit if skill bodies bloat. |

Mechanics worth copying verbatim into whatever `vault_writer.py` grows:

- **Atomic write** (tempfile + `os.replace` in the target dir) if it doesn't
  already do this.
- **Instructive refusals.** Every Hermes guard returns an error that tells
  the model exactly how to proceed legally ("call skill_view(name) … then
  retry"). This is what makes guards compatible with autonomous loops
  instead of just killing them.
- **Size caps** per note (Hermes: 100k chars) so a runaway author can't
  write a megabyte of "lessons."

## 2. The `Skill` note type

Proposed frontmatter (durable identity in frontmatter, volatile telemetry
elsewhere — Hermes' sidecar lesson, adapted to a git-backed vault):

```yaml
---
type: skill
name: klaviyo-flow-audit          # same charset rule: ^[a-z0-9][a-z0-9._-]*$
description: How to audit a Klaviyo flow end-to-end (triggers, filters, revenue attribution)
status: candidate                  # candidate | active | stale | archived
version: 3                         # integer, bumped on every edit/patch
origin: background-review          # foreground | background-review
source-sessions: ["2026-07-06-cc-klaviyo-audit"]   # appended, never rewritten
evidence: ["[[Session Log 2026-07-06]]"]           # links to artifacts of actions actually performed
gated-refs: []                     # names of hard-gated capabilities the body references (§4)
---
```

Body conventions (from Hermes' schema description, which works well): trigger
conditions, numbered steps with exact commands, a pitfalls section, and
verification steps.

**Versioning.** The vault is in git **[ASSUMPTION]**, so full history is
free; the `version` int exists so a skill can be referenced at a version
("did step 4 change since v2?") without archaeology. Hermes' `patch_count`
telemetry equivalent: derivable from git, don't store it.

**Where telemetry goes.** Hermes uses a sidecar JSON to keep churn out of
authored content. In an Obsidian vault, a hidden sidecar is anti-idiomatic
and invisible to Shane. Recommendation: keep `status` in frontmatter (it's
load-bearing and human-reviewed), and put usage counts/last-used timestamps
— if we ever want them — in a single machine-owned log note or JSON under
`myco-scripts`, not per-skill frontmatter. Don't build the curator
(auto-stale/auto-archive) in v1 at all; the library will be small enough for
Shane to prune by hand.

**Promotion criteria** (the `status` state machine — this replaces Hermes'
optional approval gate with a mandatory one):

- `candidate` — the only status an agent may *create* a skill in.
  Candidates are inert: not loaded into agent context, not citable as
  standing instructions.
- `candidate → active` — **human-only transition** (Shane edits the
  frontmatter or runs a small promote script). Suggested bar: Shane has
  either watched the procedure work, or the note's `evidence` links check
  out, AND `gated-refs` is empty or explicitly signed off (§4).
- `active → stale` — either party may set it, with a body note saying what
  changed; stale skills still load but carry a warning banner.
- `archived` — human-only in v1 (this is the delete-equivalent).
- Agents may `patch` **candidate** notes freely and **active** notes only
  with the grounding guard satisfied (§3); every agent patch bumps
  `version` and appends to `source-sessions`.

## 3. Where the grounding guard lives in Myco's stack

Hermes' central trick is that grounding is enforced **in the write path,
keyed on origin** — not by trusting the prompt. Three enforcement points,
in order of importance:

1. **Origin is a required parameter of the write, not an inference.**
   `vault_writer.py` should take an explicit `origin` on every skill write
   (`foreground` when Shane is driving a session; `background-review` for
   any autonomous/scheduled author, if that ever exists). Hermes uses a
   ContextVar because one process hosts both actors; in Myco's
   script-per-invocation world **[ASSUMPTION]**, a CLI flag / function
   argument is simpler and harder to forget. The writer stamps it into
   frontmatter itself — the model never writes its own `origin`,
   `source-sessions`, or `evidence` fields; the tool does.

2. **Evidence-or-refuse (the "did you actually do this?" guard).** Hermes'
   read-before-write guard proves the fork loaded what it's editing. Myco
   needs the *creation-side* analogue too, which Hermes gets implicitly from
   replaying a real transcript. Concretely:
   - **For edits/patches:** same rule as Hermes — the write path refuses to
     modify a skill note whose current content wasn't read through the tool
     in this session. Cheapest v1 implementation: writer requires the caller
     to pass the current `version` (or a content hash) of the note it thinks
     it's editing; mismatch → refuse with "read the note, then retry."
     That's a compare-and-swap, gets optimistic concurrency for free, and
     needs no session state.
   - **For creates:** a skill note is only accepted with at least one
     `evidence` link into an artifact of *performed* work (session log, tool
     output note, commit) **[ASSUMPTION: such artifacts exist in the vault
     or can be linked]**. The writer verifies the link target exists. This
     doesn't prove the evidence supports the claim — that's what the
     `candidate` quarantine plus Shane's promotion review is for — but it
     makes "I inferred this from vibes" structurally impossible to submit
     without fabricating a link, which review then catches.
   - **The do-not-capture list** (env-dependent failures, "tool X is
     broken", resolved transients, one-off narratives) belongs verbatim in
     whatever prompt drives skill extraction. It's the highest-value
     prompt-side content in the whole Hermes system and costs nothing.

3. **Quarantine as the backstop.** Hermes' staging gate (`write_approval` +
   pending store + replay-on-approve) maps directly onto the
   `candidate` status in §2 — but *stronger*: instead of a separate pending
   store, the vault itself holds candidates, visible in Obsidian, diffable
   in git, and inert until promoted. Autonomous writes never becoming live
   instructions without a human transition is the property that makes
   everything above safe to get slightly wrong.

The layering, stated once: **prompt** (do-not-capture list, prefer-patch
ordering) shapes behavior; **write path** (origin stamping, CAS
read-before-write, evidence-link verification, schema/size validation)
enforces mechanics; **status lifecycle** (candidate quarantine, human
promotion) contains judgment errors that survive both.

## 4. Hard-gate touchpoints — flag for Codex, not designed around

Explicit list of everything in this proposal that touches or approaches a
hard gate (money / send / delete / config-secret-self-edit). Per the brief,
none of these are designed around here; each needs Codex review before the
corresponding capability exists.

1. **Delete.** Any true deletion of a vault note by an agent. This proposal
   avoids it (archival-by-status, §1), but the moment anyone proposes
   `vault_writer` delete or a Hermes-style curator prune, that's the delete
   gate. Also latent: `patch` with an empty replacement can destroy a note
   body without "deleting" it — the writer should refuse patches that empty
   a body, and that rule itself is gate-adjacent policy for Codex to bless.
2. **Skills whose *content* drives gated actions.** A Skill note is
   executable-by-proxy: once `active`, it's a standing instruction a future
   agent will follow. A skill encoding "how to issue refunds in Stripe" or
   "how to send the weekly email" smuggles money/send behavior through a
   markdown write that itself trips no gate. Mitigation surface (needs
   Codex sign-off on the mechanism, not just the list): the `gated-refs`
   frontmatter field (§2), populated by a writer-side keyword/tool-name scan,
   and a promotion rule that a skill with non-empty `gated-refs` requires
   explicit Codex/Shane review — promotion of such a skill is treated *as*
   a gated action.
3. **Config-secret-self-edit.** Three sub-cases: (a) a Skill note that
   instructs agents how to modify `exec-approvals.json`, `AGENTS.md`, or the
   skill system's own guards — self-modifying policy through the side door;
   (b) the extraction prompt and the writer's validation rules themselves
   becoming agent-editable (they must live in myco-scripts under normal code
   review, never in the vault where the skill system could patch them);
   (c) secrets leaking *into* skill bodies from transcripts (API keys in
   example commands) — writer-side secret scan before accepting content,
   analogous to Hermes' optional `skills_guard` scan.
4. **Autonomous background writes to myco-vault at all.** The Hermes
   background-review fork has no Myco equivalent today, and the brief
   explicitly excludes autonomous vault writes. Standing one up (cron,
   post-session hook, anything) requires an `exec-approvals.json` change,
   which is itself gate-territory. v1 of this graft can ship as a purely
   **foreground, Shane-invoked** extraction step ("/extract-skills" at end
   of session) and still capture most of the value.

## Out-of-scope confirmations

Per the brief: no `exec-approvals.json` changes, no `AGENTS.md` changes, no
hermes-agent runtime dependency (it was cloned read-only into session
scratch for study), no writes to myco-vault. Honcho reviewed for reference
only (§0.4).

## Open questions for Shane

1. Verify the **[ASSUMPTION]** tags — especially what `vault_writer.py`
   already validates and whether session logs/artifacts exist in the vault
   to serve as `evidence` link targets.
2. Is `Skills/` the right subtree, and should candidates live in
   `Skills/candidates/` (Obsidian-visible quarantine) vs. a frontmatter-only
   distinction? Frontmatter-only is proposed here to keep links stable across
   promotion, but a folder makes the quarantine visually obvious.
3. Does v1 extraction run foreground-only (recommended, §4.4), and if so,
   invoked how — end-of-session slash command, or a manual script pass over
   session logs?
