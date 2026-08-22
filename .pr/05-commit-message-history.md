<!-- Titre PR proposé :
     git_ui: Add a picker to reuse a recent commit message
-->

# Objective

There is no way to get a previous commit message back into the commit editor. Rewording
an amend, following an existing convention in the repository, or recovering a message you
just used means scrolling the History tab and retyping it.

## Solution

Adds `git::CommitMessageHistory`, also reachable as "Reuse Commit Message…" in the commit
button's menu. It lists the messages of the last 50 commits on the current branch in a
searchable picker; confirming puts the full message — body included, not just the subject
— into the commit editor and focuses it.

### Where the messages come from

The repository's own log, not a stored history of what was typed in Zed.

That is a deliberate difference from the IDEs that keep a local list of previously entered
messages. Reading the log needs no new persistence or migration, works on a fresh clone
and on another machine, and reflects what was actually committed rather than what happened
to be typed in this editor. The cost is that a message you drafted but never committed is
not offered — Zed already keeps per-branch drafts for that case.

The existing `graph_data` and `fetch_commit_data` plumbing supplies this, so there is no
backend change. Each commit's data is awaited because the picker needs every message up
front to match against.

### Details

- Messages are trimmed and **deduplicated**. Repositories accumulate repeats ("wip", "fix
  tests"), and listing the same text several times only makes the picker harder to search.
  Order is preserved so the most recent commit stays first.
- The entry is hidden on a repository with no commits yet.
- No default keybinding: the commit editor context is already dense, and a binding here
  would risk colliding with existing ones. The menu entry and the command palette are
  enough to discover it.

`commit_message_history_entries` is a free function so the trimming, deduplication and
subject extraction can be tested without driving the picker.

## Testing

<!-- À COMPLÉTER PAR TOI avant d'ouvrir la PR -->

- Plateforme testée :
- Ouvert le picker sur un dépôt avec des messages répétés → vérifié qu'ils n'apparaissent
  qu'une fois.
- Vérifié qu'un message multi-lignes est réinséré en entier (corps compris).
- Vérifié la recherche dans le picker.
- Vérifié sur un dépôt sans commit (l'entrée de menu doit être absente).
- Vérifié que le texte déjà présent dans l'éditeur est bien remplacé.

## Self-Review Checklist

- [ ] I've reviewed my own diff for quality, security, and reliability
- [ ] Unsafe blocks (if any) have justifying comments
- [ ] The content adheres to Zed's UI standards
- [ ] Tests cover the new/changed behavior
- [ ] Performance impact has been considered and is acceptable

## Showcase

<!-- Capture du picker -->

---

Release Notes:

- Added a "Reuse Commit Message…" picker that fills the commit editor from a recent commit
