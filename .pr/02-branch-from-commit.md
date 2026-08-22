<!-- Titre PR proposé :
     git_ui: Add "New Branch from Here" to the commit context menu
-->

# Objective

Branches can only be created from the tip of an existing branch, via the branch picker.
There is no way to start a branch at an arbitrary commit without dropping to the terminal
— a common need when you realise work should have branched off earlier, or when you want
to build on a commit that is no longer at the tip.

Part of the commit context menu requests in discussion #53112 ("Add interactivity and
context menus to git graph feature", 45 upvotes) and #53593.

## Solution

Adds a **New Branch from Here** entry to the commit context menu, available from both the
Git Graph and the Git Panel's History tab, plus a `git_graph::CreateBranchFromCommit`
action bound to the Git Graph's selected commit.

The entry opens a small modal asking for the branch name, closely mirroring the existing
`RenameBranchModal`, and reuses `branch_picker::normalize_branch_name` so names behave
the same however they are created.

No new backend work: `GitRepository::create_branch(name, base_branch)` and its proto
plumbing already existed and already accepted an arbitrary base — the branch picker just
never passed anything but a branch name. This wires a commit SHA through the same path.

Two details worth calling out:

- **The base shown in the modal prefers a ref name over a SHA.** Right-clicking a commit
  that carries a ref shows "New Branch from main" rather than a SHA the user did not
  pick; commits without a ref fall back to the SHA.
- **The new branch is checked out.** `create_branch` runs `git switch -c`, so creation
  and checkout are one step. The modal says so explicitly rather than leaving the user to
  discover that their working copy moved.

An empty name leaves the modal open rather than dismissing it, since there is nothing
sensible to do with it.

### Test

`test_create_branch_from_commit_sha` covers the path that was previously unreachable:
creating a branch from a raw commit SHA rather than a branch name, asserting both that
the branch is checked out and — the part that actually matters — that `HEAD` lands on the
requested commit rather than staying where it was.

## Testing

<!-- À COMPLÉTER PAR TOI avant d'ouvrir la PR -->

- Plateforme testée :
- Créé une branche depuis un commit ancien dans le Git Graph → vérifié que `HEAD` est bien
  sur ce commit.
- Idem depuis l'onglet History du Git Panel.
- Vérifié le libellé de la modale sur un commit portant un ref (doit afficher le nom du
  ref) et sur un commit sans ref (doit afficher le SHA).
- Vérifié qu'un nom vide ne ferme pas la modale.
- Vérifié qu'un nom avec des espaces est normalisé en tirets.
- Vérifié le cas d'erreur : nom de branche déjà existant.

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

- Added a "New Branch from Here" option to the commit context menu in the Git Graph and Git Panel history
