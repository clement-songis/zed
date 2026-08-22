<!-- Titre PR proposé :
     git: Add tag creation from the commit context menu
-->

# Objective

Zed displays tags but has never been able to create one. Tagging a release means
leaving the editor for the terminal.

Requested in discussion #46168 ("Git tag UI", 20 upvotes), and in #53730 and #62532.

## Solution

Adds **New Tag…** to the commit context menu in the Git Graph and the Git Panel's
History tab, plus a `git_graph::CreateTag` action for the Git Graph's selected commit.

Both kinds of tag are supported, because the distinction is git's rather than ours: a
message produces an annotated tag object recording the tagger and date, and no message
produces a lightweight tag that just points at the commit. The modal therefore has two
fields — name and optional message — with `git::FocusTagMessage` moving between them,
following the pattern already used for `git_panel::FocusEditor`.

`create_tag` takes the commit environment, since annotated tags record a tagger identity.

### Test

`test_create_tag_lightweight_and_annotated` asserts against git rather than against our
own assumptions:

- `cat-file -t` reports `commit` for a lightweight tag and `tag` for an annotated one;
- the tag lands on the requested commit rather than on `HEAD`;
- reusing an existing name **fails** rather than silently moving the tag, which is the
  classic `git tag` trap.

## Testing

<!-- À COMPLÉTER PAR TOI avant d'ouvrir la PR -->

- Plateforme testée :
- Créé un tag léger et un tag annoté sur un commit ancien.
- Vérifié `git show <tag>` pour l'annoté (taggeur, date, message).
- Vérifié le message d'erreur sur un nom déjà pris.
- Vérifié la navigation Tab entre les deux champs, et qu'un nom vide ne ferme pas la modale.

## Self-Review Checklist

- [ ] I've reviewed my own diff for quality, security, and reliability
- [ ] Unsafe blocks (if any) have justifying comments
- [ ] The content adheres to Zed's UI standards
- [ ] Tests cover the new/changed behavior
- [ ] Performance impact has been considered and is acceptable

## Showcase

<!-- Capture du menu contextuel + de la modale -->

---

Release Notes:

- Added tag creation from the commit context menu in the Git Graph and Git Panel history
