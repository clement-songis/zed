<!-- Titre PR proposé :
     git_ui: Add a setting to show full files in multi-file diffs
-->

# Objective

Single-file diffs got a full-file mode in v1.6.3 (`git.file_diff.show_full_file`, on by
default). Multi-file diffs did not: the Project Diff, branch diffs, the staged and
unstaged changes views, and commit views still show only the changed hunks plus
`editor.excerpt_context_lines` of surrounding context.

This is the remaining half of the request in discussion #33773 ("Git Diff - Full File
View", 109 upvotes), which the thread itself flags after the single-file view shipped:

> Some users still request similar functionality for comparative branch diffs and git
> history viewing.

Reading a change with only a couple of lines of context means repeatedly expanding
excerpts to recover the surrounding code.

## Solution

Adds `git.multi_file_diff.show_full_file`, defaulting to `false`.

When enabled, each file in a multi-file diff is inserted as a single excerpt spanning
the whole buffer, with a context line count of `0` (the excerpt already covers the file,
so padding it would push the range past the end).

The setting is deliberately separate from `git.file_diff.show_full_file` rather than
reusing it:

- The performance tradeoff is not comparable. One file in full is cheap; every changed
  file in full, for a changeset spanning many or large files, is not. Zed already has
  open reports of slowness in these views (#56376, #52410, #58894), so this defaults to
  off while the single-file setting defaults to on.
- The two settings target views a user thinks about separately.

The logic lives in one helper, `multi_file_diff_excerpt_ranges` in `diff_multibuffer.rs`.
It takes the hunk-range computation as a closure so that callers don't pay for collecting
hunks when the whole buffer is going to be used anyway. Two call sites use it:

- `diff_multibuffer.rs` — covers the Project Diff, branch diffs, and the staged and
  unstaged changes views, which all route through `DiffMultibuffer`.
- `commit_view.rs` — covers commit views and file history, which build excerpts
  separately.

In `diff_multibuffer.rs` the helper also supersedes the conflict-range branch: when a
file has conflicts, the excerpts are normally the conflict regions rather than the diff
hunks, and full-file mode covers both.

Binary files in `commit_view.rs` keep their existing behaviour — they were already
inserted as a single whole-buffer excerpt, independent of this setting.

### Applying the setting to already-open diffs

`DiffMultibuffer` watches `SettingsStore` to refresh when the git panel's grouping
settings change; this setting is added to that check so open diffs follow it rather than
only newly opened ones.

That needs one extra step. `MultiBuffer::update_excerpts_for_path` merges the ranges it
is given with the excerpts already present for that path — as its doc comment says, it
"expands the provided ranges to cover any overlapping existing excerpts". So excerpts can
grow but never shrink, and turning the setting back off would leave the whole-file
excerpts in place. `DiffMultibuffer::clear_excerpts` drops them first when the setting
flips, so the following refresh rebuilds from scratch. (`SoloDiffView::set_showing_full_file`
already handles the equivalent case with an explicit `remove_excerpts_for_path`.)

The regression test covers the toggle in both directions, not just the initial open,
since the collapse direction is the one that silently misbehaves.

## Testing

<!-- À COMPLÉTER PAR TOI avant d'ouvrir la PR -->

- Plateforme testée :
- Vérifié avec le réglage à `false` (défaut) : les vues multi-fichiers sont inchangées.
- Vérifié avec le réglage à `true` sur : Project Diff / branch diff / staged / unstaged /
  commit view.
- Vérifié le comportement sur un fichier en conflit.
- Vérifié qu'un basculement du réglage sur un onglet déjà ouvert se comporte comme attendu.

## Self-Review Checklist

- [ ] I've reviewed my own diff for quality, security, and reliability
- [ ] Unsafe blocks (if any) have justifying comments
- [ ] The content adheres to Zed's UI standards
- [ ] Tests cover the new/changed behavior
- [ ] Performance impact has been considered and is acceptable

## Showcase

<!-- Capture avant/après sur le Project Diff -->

---

Release Notes:

- Added a `git.multi_file_diff.show_full_file` setting to show whole files in the project diff, branch diffs, staged and unstaged changes, and commit views
