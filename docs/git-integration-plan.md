# Git navigation and control implementation plan

Extend this Helix fork with searchable Git history and, in later phases, Git control. Use Helix selections, pickers, ordinary document navigation, splits, and explicit commands throughout. This document records the agreed feature and UI plan and is the shared specification for the agent tasks below. Creating this plan and the tasks does not authorize starting implementation in this session.

## Interaction and document model

- Preserve `Space g` for changed files and `Space G` for the debugger. Introduce a discoverable, remappable `Space v` version-control menu and expose actions through named commands and the command palette.
- Proposed history bindings: `c` commits, `f` file history, `l` selection history, `b` blame, `r` refs, `e` changed files in the active commit, `t` revision tree, `d` comparison, and `w` corresponding working file. `Space v a` opens Git control actions.
- Pickers retain existing controls: typing edits the query; Ctrl-n/Ctrl-p, arrows, and Tab navigate; Enter opens; Ctrl-s/Ctrl-v open splits; Ctrl-t toggles preview; Esc closes. Do not reinterpret plain j/k as picker movement.
- Open commits as immutable documents containing message, author/date, actual parents, changed files, and unified patches. Historical source retains syntax highlighting and normal motions, selections, search, and yank. Enforce immutability for transactions and writes; the existing filesystem read-only indicator is insufficient.
- Historical identity is repository plus full object ID plus historical path, independent of a working-file path. Do not attach historical source to the working file's language server. Provide explicit export into an editable copy.
- In patch documents, [g/]g select hunks. `gf` on a file entry opens historical source; on a removed patch line it opens the parent source, on an added line the commit source, and on context lines the selected after-side source. Record historical targets and selections in the jumplist so Ctrl-o/Ctrl-i works.
- Proposed [v/]v visit older/newer entries in the active result order. Parent/child pickers navigate real ancestry separately. Show all merge parents and the active comparison parent; default patches compare against the first parent. Root commits compare against the empty tree.
- Open working file maps to the current path/location when known without discarding dirty buffers. Deleted files remain accessible historically. Do not claim exact line mapping when it is ambiguous.

## Phase 1 Commit browsing and filtered history search

Deliver repository discovery, typed history queries, virtual document support, commit picker/previews, patch navigation, and historical source opening. The commit picker displays hash, age/date, author, refs, and subject; preview shows message/details or the matching paths/hunks.

Default scope is commits reachable from HEAD, including merged branches. Scopes include selected branch/tag, all known refs, revision range, and first-parent history. Ref selection changes browsing scope and does not check out a branch. Direct revision expressions are resolved to full object IDs.

| Field | Meaning | Matching |
| --- | --- | --- |
| Unprefixed, %m, %message | Full message, subject and body | Existing Helix fuzzy syntax |
| %p, %path | Changed paths, including rename source and destination | Existing Helix fuzzy syntax, evaluated per path |
| %d, %diff | Added and removed text lines | Literal substring by default, explicit visible regex mode |
| %a, %author | Author name and email | Fuzzy |
| %h, %hash | Commit object ID | Prefix or exact |
| %r, %ref | Branch/tag names directly decorating the commit | Fuzzy |

Fields combine with AND. Text after a field prefix belongs to that field until the next prefix, following the existing picker grammar. Unprefixed text targets the message only. Support escaped literal percent characters and clear field hints/errors.

Examples:

```text
authentication
%p helix-vcs
%d GIT_TERMINAL_PROMPT
fix %p helix-vcs %d GIT_TERMINAL_PROMPT
%a paul %p commands.rs
```

Positive path and diff constraints must match within the same changed file. Define exclusions precisely: a negative path term excludes a commit touching any matching old or new path; negative diff terms exclude a candidate file containing that text. Positive path terms must all match one old or new path; positive diff patterns match within a changed line by default. Handle pure-negative queries explicitly. Diff search excludes unchanged context and patch headers by default; add context inclusion and added/removed/both controls. Never match arbitrary concatenated text across files or line boundaries.

For merges, search the selected parent comparison and display that policy. Message matches highlight message text; path matches show matching files; diff matches show highlighted hunks. Enter opens the commit at its first match; a matches picker chooses other matching files/hunks.

Search the entire selected history, progressively, with debounce, bounded caches, cancellation, visible progress, and a distinction between interim and completed no-results. Extend dynamic picker callbacks to receive the full parsed query, not only the primary field. Diff text must not be accumulated as enormous hidden fuzzy columns.

## Phase 2 File and selection history

Provide file history following single-file rename ancestry, blame of selected lines, and selection history tracing changes to selected ranges. Deduplicate commits across multiple selections and retain range provenance. Historical buffers start from their own revision. Working-buffer ranges must map back to committed source; label added lines uncommitted and explain ambiguous/unavailable mappings. Rename/copy detection is heuristic and line tracing must not silently invent ancestry.

## Phase 3 Repository exploration and comparisons

Provide local branches, tags, remote-tracking refs, direct revision opening, actual parent/child navigation, revision file picker/tree including unchanged files, and historical source search. Compare any two revisions with a changed-files picker and patches. Clearly distinguish HEAD, index, on-disk working tree, and unsaved buffer when offering local comparisons. Add advanced added/removed text evolution queries alongside ordinary literal/regex diff search.

## Phase 4 Advanced history views

Add a compact topology graph in unfiltered history; filtered results must not imply false adjacency. Add reflog navigation for otherwise unreachable commits and paired before/after source comparisons with shared hunk navigation. Binary, oversized, missing/shallow, symlink, submodule, and malformed/non-UTF-8 data need explicit representations rather than misleading text previews.

## Phase 5A Staging and commits

The status navigation surface is a structured immutable status/diff document with ordinary selections; commands operate on selected semantic entries. Show staged, unstaged, untracked, renamed, deleted, and conflicted states separately. Retain Space g and provide index/worktree patch views. Implement file staging/unstaging, then whole-hunk staging/unstaging, then precise changed-line staging. Whole-hunk actions must make their scope apparent.

Commit opens an editable message draft beside the staged diff. Writing saves the draft; a separate submit action creates the commit. Retain drafts on hook/signing failure. Before submission verify the reviewed index and HEAD; if changed, refresh and require review of the new state. Add amend, fixup selection from history, and cherry-pick in subsequent tasks.

Space v a initially exposes s stage, u unstage, c commit, f fetch, l pull, p push, and r rebase. Keep operation-specific continue/skip/abort actions discoverable.

## Phase 5B Network operations and conflict resolution

Use the Git executable for mutations, respecting Git hooks, signing, credentials, configuration, and the existing workspace trust decision. A shared controller handles structured arguments, progress/output, editor callbacks, credentials, error states, repository serialization, cancellation semantics, and operation-aware document refresh. Provide an interactive terminal handoff when a credential or signing flow cannot be served inside Helix.

Fetch chooses a remote and refreshes refs. Pull fetches and previews incoming/divergent commits, defaults to fast-forward-only, and offers explicit merge/rebase choices for divergence. Show branch/upstream identity. Push previews source, destination, and outgoing commits; configure absent upstream explicitly. A separate force-with-lease action pins the expected remote object ID from review; never fall back to unconditional force.

Operations changing working files must detect dirty buffers and offer explicit save, postpone, or deliberate stash handling instead of overwriting editor state. Cancellation does not imply rollback. Text undo does not undo Git mutations.

Conflict picker and resolution views show base, both correctly labelled operation-specific sides, and editable result. In a rebase distinguish upstream from replayed change rather than relying on ambiguous ours/theirs labels. Navigate conflicts with selections, choose sides or edit result, explicitly mark resolved, and expose continue/skip/abort where supported. Handle add/delete/rename and binary conflicts as well as text conflicts.

## Phase 5C Interactive rebase

Choose a base/ref or contiguous commit range, review affected commits and merge topology policy, then edit a normal rebase todo document. Provide selection-based pick, reword, edit, squash, fixup, and drop helpers and line movement. Validate supported todo syntax and commit identities before explicit start. Do not silently flatten merge topology or execute arbitrary todo shell commands.

Integrate Git sequence/message editor callbacks with Helix documents. Track paused edit/reword/conflict states and continue/skip/abort; resume operations initiated outside Helix. Preserve drafts, show failures, and offer abort/recovery. Rewriting published history must be visible and link to the guarded push workflow.

## Phase 6 Tutorial and real screenshots

Create a deterministic example repository with branches, merges, a bug, renamed/deleted files, staging examples, conflicts, and a local bare remote. Explain prerequisites and resets so exercises can be repeated without using a user's repository.

Cover every shipped feature: message/path/diff filters and combinations; commit/hunk/source navigation; historical-to-working return; file/selection history and blame; refs, trees, source search, revision comparisons, parent/child navigation; graph/reflog/advanced patch search/paired comparison; file/hunk/line staging; commit/amend/fixup/cherry-pick; fetch/pull/push/guarded rewritten-history push; rebase planning, conflict resolution, continue/skip/abort, and failed-operation recovery.

Each exercise has starting state, exact keys/commands, actual implemented-editor screenshots, and expected results. Capture reproducibly using fixed terminal dimensions/theme and stable repository metadata. Store screenshots and capture instructions with the repository documentation. Add a short in-editor introduction that links to the full tutorial. Run every exercise against the implemented build and verify screenshot/document links. Do not fabricate screenshots or describe proposed behavior as implemented.

## Implementation references and verification

- Existing backend: helix-vcs/src/lib.rs, git.rs, diff.rs, status.rs, git/test.rs, Cargo.toml.
- Documents/editor/jumplist: helix-view/src/document.rs, editor.rs, view.rs.
- UI: helix-term/src/commands.rs, commands/typed.rs, keymap/default.rs, ui/picker.rs, ui/picker/query.rs, ui/picker/handlers.rs, ui/document.rs, ui/statusline.rs.
- Documentation: book/src/keymap.md, pickers.md, commands.md, summary/navigation files and docs/architecture.md.
- Existing integration harness: helix-term/tests/integration.rs and tests/test/helpers.rs; examine actual crate test conventions before adding tests.
- Test real semantics with temporary Git repositories, especially root/merge commits, renames/deletions, linked worktrees/submodules, dirty buffers, changed index/HEAD between review and action, failed hooks, remote divergence, lease rejection, conflict recovery, and cancellation. Performance checks must demonstrate that the editor thread remains responsive and incomplete scans are labelled.

Primary behavioral references: [Helix pickers](https://docs.helix-editor.com/master/pickers.html), [Helix keymap](https://docs.helix-editor.com/master/keymap.html), [git log](https://git-scm.com/docs/git-log), [git blame](https://git-scm.com/docs/git-blame), [git add](https://git-scm.com/docs/git-add), [git commit](https://git-scm.com/docs/git-commit), [git rebase](https://git-scm.com/docs/git-rebase), [git pull](https://git-scm.com/docs/git-pull), and [git push](https://git-scm.com/docs/git-push).

## Task index

All tasks are pending, tagged `agent`, `gitIntegration`, and `rocky`, with phase tags and explicit dependencies. No `auto` tag was added. Alias IDs below are the IDs printed by `ask`.

| Alias | Phase | Task | Prerequisite aliases |
| --- | --- | --- | --- |
| q03 | 1 | Add repository discovery and typed Git history backend | None |
| r03 | 1 | Add immutable historical and structured Git document types | None |
| s03 | 1 | Add reusable temporary Git repository fixtures for integration tests | None |
| t03 | 1 | Implement full-history message path and diff filter query engine | q03, s03 |
| u03 | 1 | Extend picker infrastructure for parsed queries and virtual previews | r03 |
| v03 | 1 | Add searchable commit picker and version control entry commands | q03, t03, u03 |
| w03 | 1 | Implement patch hunk to historical source navigation and jumplist return | v03 |
| x03 | 1 | Verify responsive complete-history search and bound history caches | w03, s03 |
| y03 | 2 | Add current and historical file history with rename tracing | x03 |
| z03 | 2 | Add selection-based blame with committed line provenance | x03 |
| 013 | 2 | Add multi-selection line history and revision-aware range tracing | y03, z03 |
| 113 | 3 | Add ref scopes direct revision opening and ancestry navigation | x03 |
| 213 | 3 | Add revision tree file picker and historical source search | 113, u03 |
| 313 | 3 | Add arbitrary revision comparisons and explicit local diff targets | w03, 113 |
| 413 | 3 | Add advanced added and removed text evolution searches | t03, v03 |
| 513 | 4 | Add compact commit topology graph with honest filtered results | 113, v03 |
| 613 | 4 | Add reflog picker and unreachable commit navigation | 113, v03 |
| 713 | 4 | Add paired before and after source views with linked hunk navigation | w03, 313 |
| 813 | 4 | Validate completed history workflows and document the history feature set | x03, 013, 213, 313, 413, 513, 613, 713 |
| 913 | 5A | Add shared Git mutation controller and interactive process handling | 813 |
| a13 | 5A | Add selectable Git status documents and file staging or unstaging | 913, 313 |
| b13 | 5A | Implement selection-based whole-hunk staging and unstaging | a13, w03 |
| c13 | 5A | Add precise changed-line staging and unstaging from selections | b13 |
| d13 | 5A | Add commit message drafts staged review and explicit submission | b13 |
| e13 | 5A | Add amend and history-selected fixup commit workflows | d13, 113 |
| f13 | 5B | Add remote fetch with progress credentials and ref refresh | d13, 913, 113 |
| g13 | 5B | Add conflict picker and selection-driven resolution documents | d13, 913, 313 |
| h13 | 5B | Add selected commit cherry-pick and resumable conflict handling | d13, g13 |
| i13 | 5B | Add pull preview fast-forward integration and explicit divergence handling | f13, g13 |
| j13 | 5B | Add reviewed normal push and explicit upstream configuration | f13, d13 |
| k13 | 5B | Add separate force-with-lease push pinned to reviewed remote tip | j13 |
| l13 | 5C | Add editable validated interactive rebase plans and selection helpers | i13, e13, k13 |
| m13 | 5C | Execute and resume interactive rebases with editor callbacks and recovery | l13, g13 |
| n13 | 5C | Validate complete Git control workflows and document recovery behavior | c13, e13, h13, i13, k13, m13 |
| o13 | 6 | Create deterministic tutorial repository and complete exercise scripts | n13 |
| p13 | 6 | Capture reproducible real-editor screenshots for every tutorial workflow | o13 |
| q13 | 6 | Write and verify complete Git tutorial with screenshots and in-editor introduction | p13 |
