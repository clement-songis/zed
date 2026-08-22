<!-- Titre PR proposé :
     git_ui: Highlight your own commits in the Git Graph
-->

# Objective

Scanning a shared history in the Git Graph gives no way to pick out your own commits —
every author name renders identically, so finding "the thing I pushed yesterday" means
reading every row.

## Solution

Renders the author name in `Color::Accent` when the commit's author address matches the
current user, leaving every other row as it was.

### Which identity

The email is read with `git config user.email` **from inside the repository**, not from
the global config.

This matters: `get_git_committer` (used for commit co-authors) reads `--global` only, and
per-repository overrides of `user.email` are common — work versus personal addresses on
the same machine. Using the global identity would mislabel commits in exactly the
repositories where the distinction is most useful.

Reading git config is not supported for remote projects, so highlighting is skipped there
rather than silently falling back to an identity that may be wrong. It is likewise skipped
when no identity is configured at all.

The identity is per-repository, so `set_repo_id` re-resolves it instead of carrying the
previous repository's address over.

### Matching

Comparison is case-insensitive, since git records whatever case the author configured and
the same person can appear under several spellings.

`authored_by_local_user` is a free function so the matching rule can be tested directly.
The test pins the two cases that would be actively harmful if they regressed: an
unresolved identity must not match anything (otherwise *every* commit lights up), and a
commit with no recorded author must not be attributed to the local user.

## Testing

<!-- À COMPLÉTER PAR TOI avant d'ouvrir la PR -->

- Plateforme testée :
- Ouvert le Git Graph sur un dépôt à plusieurs auteurs → vérifié que seuls mes commits
  ressortent.
- Vérifié dans un dépôt avec `git config --local user.email` différent du global.
- Vérifié qu'un dépôt sans identité configurée n'allume rien.
- Vérifié le changement de dépôt actif (l'identité doit être re-résolue).
- Vérifié le rendu en thème clair et sombre.

## Self-Review Checklist

- [ ] I've reviewed my own diff for quality, security, and reliability
- [ ] Unsafe blocks (if any) have justifying comments
- [ ] The content adheres to Zed's UI standards
- [ ] Tests cover the new/changed behavior
- [ ] Performance impact has been considered and is acceptable

## Showcase

<!-- Capture du graphe sur un dépôt à plusieurs auteurs -->

---

Release Notes:

- Added highlighting for your own commits in the Git Graph
