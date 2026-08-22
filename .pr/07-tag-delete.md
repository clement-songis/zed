<!-- Titre PR proposé :
     git: Add tag deletion from the commit context menu

     ⚠️ AVANT DE PROPOSER : si la PR "tag creation" est déjà mergée upstream,
     renuméroter GitDeleteTag de 480 à 481 dans crates/proto/proto/zed.proto.
     Deux messages proto ne peuvent pas partager un numéro de champ, et ça
     compile sans erreur avec un doublon.
-->

# Objective

Follows the tag creation work: tags could be created but not removed.

## Solution

Adds **Delete Tag** to the commit context menu, mirroring the existing "Copy Tag"
structure — a single entry when the commit carries one tag, a submenu when it carries
several. No new UI convention.

Deletion always confirms. A deleted tag is only recoverable by someone who still knows
the commit it pointed at, so there is no undo to offer instead.

This removes the local tag only. Deleting it on a remote needs `push --delete`, which
belongs with the tag-pushing work; the documentation says so rather than leaving users
to discover that the tag is still on the remote.

### Test

`test_delete_tag` covers the absent side effect as much as the visible one: deleting a
tag must leave `HEAD` where it was, and deleting a tag that does not exist must report
an error rather than silently succeed.

## Testing

<!-- À COMPLÉTER PAR TOI avant d'ouvrir la PR -->

- Plateforme testée :
- Supprimé un tag depuis un commit qui n'en porte qu'un (entrée simple).
- Supprimé un tag depuis un commit qui en porte plusieurs (sous-menu).
- Vérifié que « Cancel » dans la confirmation n'efface rien.
- Vérifié qu'un tag déjà poussé reste sur le remote.

## Self-Review Checklist

- [ ] I've reviewed my own diff for quality, security, and reliability
- [ ] Unsafe blocks (if any) have justifying comments
- [ ] The content adheres to Zed's UI standards
- [ ] Tests cover the new/changed behavior
- [ ] Performance impact has been considered and is acceptable

## Showcase

<!-- Capture du menu -->

---

Release Notes:

- Added tag deletion from the commit context menu in the Git Graph and Git Panel history
