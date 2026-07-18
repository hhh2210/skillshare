# Feature Proposal: Install Local Path as Symlink (`--link`)

## Problem

`skillshare install <local-path>` copies the skill directory into the skillshare source tree. This works for standalone skills but breaks the expected workflow when a skill is **owned and updated by another application** that ships it as part of its own bundle.

**Concrete example — Surge for macOS:**

Surge (a macOS proxy app) ships an authoritative skill at:

```
/Applications/Surge.app/Contents/Resources/Skills/surge
```

After `skillshare install /Applications/Surge.app/Contents/Resources/Skills/surge`, skillshare writes a snapshot copy to `~/.config/skillshare/skills/surge/`. When Surge updates (e.g. new commands added to `command-reference.md`), the copy silently falls behind until the user explicitly runs `skillshare update surge` or reinstalls with `--update`/`--force`. The recorded local source makes a manual refresh possible, but nothing ties that refresh to the app update, so users are unlikely to know when it is needed.

The current workaround — manually deleting the copy and symlinking the source directory — bypasses skillshare's metadata and is not reliably discovered by the current source walker.

## Proposed Solution

Add a `--link` flag to `skillshare install` for local path sources:

```bash
skillshare install --link /Applications/Surge.app/Contents/Resources/Skills/surge
```

Instead of copying the directory, Skillshare places a managed **directory link** in the source tree (a symlink on Unix and a directory junction on Windows):

```
~/.config/skillshare/skills/surge  →  /Applications/Surge.app/Contents/Resources/Skills/surge
```

The source-tree entry always reflects the live path. Merge- and symlink-mode targets see those changes through their existing indirection; copy-mode targets receive the latest files the next time `skillshare sync` runs. No re-install is needed after app updates.

### CLI surface

```
skillshare install --link <local-path>
skillshare install -L <local-path>     # short form
```

The first version accepts only a local filesystem source that resolves to exactly one selected skill. Git and other remote sources, tracked installs, agent installs, and multi-skill selections are rejected with a clear error. The external link target and per-machine metadata `source` are the normalized absolute directory selected by `SkillInfo.Path`, not necessarily the directory passed on the command line. A single nested selection links that nested skill directory. A root skill (`SkillInfo.Path == "."`) may link the input directory only when discovery found no descendant skills; when descendants exist, the current installer treats the root as an orchestrator and copies only its `SKILL.md`, so `--link` rejects that selection rather than exposing the whole repository and its nested skills through one managed entry.

`--name` and `--into` change only the destination in Skillshare's source tree. Before changing that destination, the install must verify the selected directory and its `SKILL.md`, run the normal install audit, and reject source/destination overlap that could create a link cycle. That comparison is based on filesystem identity, not just cleaned path strings: existing paths are resolved through symlink/junction and case aliases, and a not-yet-created destination is compared by resolving its deepest existing ancestor. If the implementation cannot establish that the source and destination are disjoint, it fails closed. `--force` authorizes replacing the destination only after those checks; it never authorizes writing through to or deleting the external target.

### Commit and rollback

A linked install is successful only when the directory link and every required persistence record agree. Global mode requires the link plus `.metadata.json`; project mode additionally requires the corresponding `.skillshare/config.yaml` entry. A metadata or project-config write failure is therefore fatal, not a warning.

The operation takes an interprocess install lock covering the destination, metadata store, and project config before it validates the final preconditions or stages any change. Lock acquisition has a bounded wait and a diagnostic naming the contended operation; ordinary installs that touch the same records use the same lock so a linked install cannot race a copy install or bare-config replay.

While holding that lock, the operation stages the new link and candidate records before discarding prior state. When replacing an entry, the old destination and its records remain recoverable until the new link and all required records are durable. If link creation or any persistence step fails, Skillshare removes the staged/new link, restores the previous destination and records, and reports the install as failed. Staging artifacts are recognizable so a later install or doctor run reports and can restore an operation interrupted before cleanup. The exact staging primitive may differ by platform, but rollback must never traverse or modify the external target; if rollback itself cannot complete, the recovery location is retained and reported.

### Metadata

Skillshare does not have a live global `registry.yaml`. Installed state and provenance live in a `.metadata.json` store inside each effective skills source directory. Global bare `skillshare install` replays that store. Project bare `skillshare install` reads the `skills` list in `.skillshare/config.yaml`, while the project skills source has its own `.metadata.json` for installed state.

Each managed link's per-source-directory metadata entry should store `link: true` alongside the normalized absolute selected-skill `source` path so `list`, `check`, `update`, and `uninstall` can distinguish it from an untracked symlink or a copied local skill:

```json
{
  "version": 1,
  "entries": {
    "surge": {
      "source": "/Applications/Surge.app/Contents/Resources/Skills/surge",
      "type": "local",
      "link": true
    }
  }
}
```

Both global and project installs add `link: true` and the resolved absolute target to the effective skills source's machine-local `.metadata.json` `MetadataEntry`. Project mode additionally carries the flag through its declarative `.skillshare/config.yaml` skill entry and the install DTO used to replay that entry. The project entry preserves the original source expression, resolving paths relative to the project root; a source inside the project should therefore be stored project-relative, while an application-owned absolute path remains explicitly machine-local:

```yaml
skills:
  - name: surge
    source: /Applications/Surge.app/Contents/Resources/Skills/surge
    link: true
```

Persisting the flag is part of the first version: a later bare install resolves the declarative project source (when present), refreshes the machine-local absolute metadata target, and recreates the link instead of silently converting it into a snapshot copy. If an absolute machine-local path is unavailable on another machine, that entry is reported and skipped while the remaining entries continue. Global `config.yaml` does not gain a `skills` list, and legacy `registry.yaml` remains migration-only input rather than gaining a new live schema.

Metadata reconciliation uses link-aware inspection. A `link: true` record is retained until an explicit uninstall or a successful conversion to an ordinary copied install, even when the link or its target is absent or invalid. Reconciliation may update observed state, but it must not silently erase the intent needed to diagnose, repair, or uninstall the entry.

### Managed-link states

State is determined from the persisted record, the source-tree entry inspected without following it, the recorded target identity, and the target's current skill boundary:

| State | Definition | Mutation policy |
| --- | --- | --- |
| **Healthy** | A symlink/junction exists, points to the recorded target, and the target is a readable directory with `SKILL.md` | Discovery, sync, and audit may read through it; update does not rewrite it |
| **Broken** | The correct link object exists, but its recorded target is missing | Preserve records; skip update and sync with a diagnostic; uninstall removes only the link and records |
| **Absent** | A managed record exists, but no source-tree entry exists | Preserve until bare install rehydrates it or uninstall removes the records; update and sync do not create it |
| **Replaced/wrong-target** | The destination is a regular file/directory or a link to a different target | Preserve records and refuse update, sync, and uninstall without mutating the replacement; require the user to resolve the conflict or run a new audited `install --link --force` transaction |
| **Invalid target** | The correct link is reachable, but the target is not a readable skill directory with `SKILL.md` | Preserve records; skip update, sync, and audit-as-skill with a diagnostic; uninstall may remove only the link and records |

These states are management states, not integrity guarantees. Content underneath a healthy external target can still change after inspection.

### Behavior of adjacent commands

| Command | Linked skill behavior |
| --- | --- |
| `skillshare list` | Include every metadata-backed managed link; show `→ /path/to/target` when healthy and visible `[broken]`, `[absent]`, `[conflict]`, or `[invalid]` state when not |
| `skillshare check` | Report a healthy entry as `externally managed — link is live`; return the recorded target and structured state for every non-healthy entry without claiming content immutability |
| `skillshare update` | Skip a healthy entry as `externally managed`; report and preserve every non-healthy state without replacing the link or falling back to a copy |
| `skillshare uninstall` | For healthy, broken, or invalid entries, remove only the source-tree link and records; for absent entries, remove only the records; refuse and preserve replaced/wrong-target entries |
| `skillshare doctor` | Report the managed state, recorded target, and recovery action, including link-target mismatch and invalid skill boundaries |
| `skillshare collect` | Reserve and skip every managed-link destination, regardless of state, rather than replacing it; users can explicitly uninstall and reinstall without `--link` to opt into a copy |

### Sync behavior

Discovery should add only metadata-backed managed links; it must not start following arbitrary user-created directory symlinks. Only healthy managed links participate like ordinary skill directories under their logical source-tree paths. Every non-healthy state is skipped without failing the remaining discovery or sync work.

Generated links preserve Skillshare's source-of-truth indirection: merge-mode per-skill links point to the logical entry in the Skillshare source tree, while a symlink-mode target continues to point to the Skillshare source directory. Neither mode flattens directly to the ultimate application path. Copy mode dereferences each healthy managed link and copies its current files when sync runs. This keeps target behavior consistent if the external path or source-tree entry is later replaced.

The current source walker does not descend into a directory symlink merely because it contains `SKILL.md`, so this behavior requires an explicit linked-skill discovery path rather than relying on the manual-symlink behavior today.

Managed-link diagnostics are returned as structured data alongside discovery results; the discovery layer does not print them or write through a global diagnostic stream. Human CLI commands render warnings on stderr, JSON commands keep stdout valid and include a structured warnings field, and server handlers return diagnostics in their response contract. Background consumers such as hub indexing may skip non-healthy entries through a caller-owned logging policy. This lets each consumer preserve its output contract while sync continues past a bad link.

### Audit and integrity policy

`install --link` runs the normal install audit against the current target before creating the managed link and honors the existing audit flags and policy. The install output explains that this is an explicit trust relationship: later application updates become visible immediately and are not automatically re-audited.

Linked entries should omit the copied-skill `file_hashes` snapshot. Expected external changes would otherwise look like perpetual local tampering, while the link cannot provide snapshot integrity by design. `skillshare audit` must follow a healthy managed link and scan its current content; `check` and `doctor` validate the recorded target identity and link health rather than claiming content immutability. Uninstall and dirty-file protections must never traverse into or modify the external target.

## Alternatives Considered

**Re-run `install --force` after each app update.** Fragile and easy to forget. Breaks the "install once" contract and produces silent drift.

**Manual symlink in source dir.** Bypasses skillshare's tracking and is not reliably discovered by the current source walker: there is no metadata, `list` indicator, or purpose-built broken-link diagnostic. The feature request is to give this pattern first-class support.

**`mode: link` in config.** Rejected for the first version because `mode` already describes source-to-target distribution (`copy`, `merge`, or `symlink`), while this feature describes how one source entry is owned. Persisting a per-entry `link: true` flag keeps those two concepts separate.

**Watch-based auto-sync.** Heavier than needed. A symlink at install time is sufficient and adds zero runtime overhead.

## Scope

- [ ] Small (1-3 files, < 200 lines)
- [ ] Medium (3-10 files, 200-500 lines)
- [x] Large (10+ files, 500+ lines)

Expected touch points:

- `cmd/skillshare/install.go` and `internal/install/` — wire `--link` / `-L`, resolve the selected `SkillInfo.Path`, enforce single-skill local-only use, and own the staged commit/rollback across destination and persistence records
- `internal/install/metadata.go` and the `.metadata.json` schema (currently `schemas/registry.schema.json`) — persist link intent in each skills source's metadata store
- `internal/config/`, the project schema, install DTOs, and reconciliation — persist/replay link intent for declarative project skills and retain every managed-link state until explicit removal or conversion
- A leaf filesystem-link package (`internal/utils/` or a dedicated leaf package with no config/install/sync dependency) — own platform-specific create, inspect, resolve, and remove primitives used by both install and sync
- `internal/sync/` — merge metadata-backed links into discovery, return structured diagnostics, preserve source indirection for merge/symlink modes, and dereference only healthy links for copy mode
- CLI discovery consumers — global/project `sync`, `list`, `check`, `audit`, `uninstall`, `doctor`, `status`, `diff`, and `init`, including their human, JSON, and TUI renderers
- Server and background discovery consumers — every `DiscoverSourceSkills*` caller, including `internal/server/` overview, target, skill/content/batch/toggle, analyze, audit, uninstall, sync/matrix/diff, git, and skill-ignore handlers, plus `internal/hub/` indexing
- `.github/workflows/test.yaml` — execute Windows-specific link tests on a Windows runner rather than merely cross-compiling build-tagged code

Tests: unit tests for flag compatibility, selected nested-skill resolution, root-orchestrator rejection, physical path overlap through symlink/junction and case aliases, staged rollback at each failure point, concurrent install serialization, metadata/project-config/DTO round-tripping, relative project replay, all five managed-link states, structured diagnostic propagation, discovery, analyze/audit traversal, omitted file hashes, copy/merge/symlink target semantics, collect protection, and safe uninstall. Integration tests cover install → list → sync → check/analyze/audit → uninstall, global/project bare-install rehydration, clean JSON/server diagnostics, concurrent linked/copy installs, and interrupted replacement recovery. A Windows unit/smoke job creates, recognizes, breaks, and removes both junction and symlink forms while asserting the external target remains untouched.

## Decisions for the first version

1. **Persistence and commit** — Persist `link: true` in install metadata and project skill entries. Machine-local metadata records the resolved absolute target; project config preserves a project-root-relative source when possible and otherwise records an explicitly machine-local absolute source. Link creation and required records form one interprocess-locked success boundary with rollback on partial failure. Bare installs recreate absent links when the selected target is valid, skip unavailable machine-local paths with a diagnostic, and never convert them to copies; reconciliation preserves all managed-link states.

2. **Windows** — Create an absolute directory junction first, matching current sync behavior for non-relative links; fall back to a directory symlink only if junction creation fails. All adjacent commands recognize and remove either form without requiring Administrator or Developer Mode for the normal path, and a Windows CI smoke job verifies those guarantees.

3. **`skillshare collect`** — Skip the linked skill with a warning. Converting ownership to a snapshot is an explicit uninstall/reinstall operation.

4. **Local install scope and name override** — Support exactly one selected local skill. The resolved selected `SkillInfo.Path` is the link target and machine-local metadata source; project config retains its replayable source expression. Reject a root-orchestrator selection that has descendant skills because it has no safe directory boundary. `--name` and `--into` change only the destination. Multi-skill selections, agents, tracked installs, and remote sources remain ordinary installs.

5. **Conflicts and safety** — `--force` may replace the existing destination only through the staged transaction, but it must never modify the external target. Reject self-referential or ancestor/descendant source-destination relationships using physical filesystem identity so aliases cannot bypass the check. Update, sync, and uninstall never repair or remove a replaced/wrong-target entry implicitly.

6. **Audit and hashes** — Audit once before linking, warn that future external changes are trusted, omit copied-skill file hashes, and make explicit audits scan the current link target.

## Non-goals

- Watching the external application path or automatically running sync after it changes. Copy-mode targets still require an explicit sync.
- Re-auditing or providing snapshot-integrity guarantees for every later external update.
- Treating arbitrary manual symlinks as managed links; management begins only with `install --link` metadata.
- Linking git/remote sources, tracked repositories, agents, or multiple skills in one invocation in the first version.
- Reviving `registry.yaml` or adding a global declarative `skills` list to `config.yaml`.
