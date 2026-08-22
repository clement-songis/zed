<!-- Titre PR proposé :
     git_ui: Add tag pushing from the commit context menu
-->

# Objective

Creating a tag only creates it locally. Publishing it meant going back to the terminal,
which defeats the point of having tag creation in the editor at all.

Requested in discussion #62532 ("Git Graph: create and push Git tags to origin").

## Solution

Adds **Push Tag** to the commit context menu, next to the existing tag entries. With a
single remote it pushes there; with several, it asks which one.

### No new git command

Pushing a tag is pushing a ref, so this reuses `push` with an explicit
`refs/tags/<name>:refs/tags/<name>` refspec.

That was a deliberate choice over adding `push_tag` to `GitRepository`: a new command
would have needed its own proto message, envelope number, and a duplicate of the askpass
and environment handling that `push` already carries — all to run the same git
invocation. The whole change is UI plus one enum variant.

### Why `RemoteAction::PushTag` rather than reusing `Push`

The success toast differs. A branch push reports a file count and, when it cannot find a
pull request URL in the output, falls back to offering "Create Pull Request". Neither
makes sense for a tag, and the fallback would have offered to open a pull request for a
tag push.

### Test

`test_push_tag_refspec` runs against a real remote: tag locally, push, then assert the
tag exists on the remote **and points at the same commit** — the refspec being wrong in
a way that still succeeds is the failure worth guarding against.

## Testing

<!-- À COMPLÉTER PAR TOI avant d'ouvrir la PR -->

- Plateforme testée :
- Poussé un tag vers un dépôt à un seul remote.
- Poussé vers un dépôt à plusieurs remotes (sélecteur).
- Re-poussé le même tag (doit indiquer qu'il est déjà à jour).
- Vérifié le toast de succès : pas de bouton « Create Pull Request ».
- Vérifié le comportement sur un remote demandant une authentification (askpass).

## Self-Review Checklist

- [ ] I've reviewed my own diff for quality, security, and reliability
- [ ] Unsafe blocks (if any) have justifying comments
- [ ] The content adheres to Zed's UI standards
- [ ] Tests cover the new/changed behavior
- [ ] Performance impact has been considered and is acceptable

## Showcase

<!-- Capture du menu + du toast -->

---

Release Notes:

- Added "Push Tag" to the commit context menu in the Git Graph and Git Panel history
