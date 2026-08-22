# Roadmap — refonte du client Git de Zed

Compagnon de [`GIT_JETBRAINS_GAP.md`](./GIT_JETBRAINS_GAP.md) (rapport d'écart + arbitrages).
Ce document décrit **comment** on avance : conventions, découpage en branches, ordre.

---

## 1. Conventions de travail

### Branches

- Une branche par étape, nommée `git/<slug>`.
- **Branchée sur `main`** (décision du 17/08/2026), et **mergée dans `main` en `--no-ff`** :
  chaque étape reste un point de revert atomique.
- Les PR sont proposées **dans l'ordre**, chacune après le merge upstream de la précédente.

```sh
git switch -c git/ma-etape main
# ... travail, commits ...
git switch main && git merge --no-ff git/ma-etape
```

**Pourquoi `main` et pas `upstream/main`.** Une branche partant d'`upstream/main` entre
en conflit avec `main` dès que plusieurs étapes touchent les mêmes fichiers — constaté
sur les étapes 06 et 07 : 17 hunks, puis une **collision de numéro d'enveloppe proto**.
Partir de `main` supprime les deux.

Le diff d'une PR se calcule en *trois points* depuis le merge-base avec
`upstream/main`. Une branche issue de `main` embarque donc tout ce que `main` a en plus
d'upstream :

| Contenu de `main` | Effet sur le diff d'une PR |
|---|---|
| Commits de fonctionnalité | Normal — ils disparaissent à mesure qu'upstream les merge |
| Commits de merge `--no-ff` | Aucun — ils n'apportent pas de contenu |
| Docs de planification | **Écartés** : ils vivent sur la branche `planning` |

**Vérification avant toute proposition :**

```sh
git diff --name-only upstream/main...git/<slug>   # que le code attendu
```

**⚠️ Numéros d'enveloppe proto.** Deux branches issues du même point réclament le même
« prochain » numéro libre. Ce n'est pas un conflit textuel : les numéros de champ
identifient les messages sur le fil, et **ça compile parfaitement avec un doublon**. À
vérifier après chaque rebase touchant `zed.proto` :

```sh
python3 -c "
import re; s=open('crates/proto/proto/zed.proto').read()
n=re.findall(r'=\s*(\d+);', re.search(r'oneof\s+payload\s*\{(.*?)\n  \}', s, re.S).group(1))
print('doublons:', [x for x in set(n) if n.count(x)>1] or 'aucun')"
```

### La branche `planning`

`ROADMAP.md`, `GIT_JETBRAINS_GAP.md` et `.pr/` vivent sur la branche **`planning`**,
qui n'est **jamais mergée dans `main`**. Upstream n'en a aucun usage, et sans ça ils
apparaîtraient dans le diff de chaque PR.

### Le fichier de message de PR

Chaque étape a son texte de PR dans **`.pr/NN-slug.md`** (numéroté selon l'étape).

> **Pourquoi pas `MESSAGE.md` à la racine ?** Avec ~30 branches mergées successivement
> dans `main`, un fichier unique à la racine produit un conflit à chaque merge.
> `.pr/NN-slug.md` s'accumule sans conflit et te laisse l'archive complète des brouillons.

**Les branches de fonctionnalité ne contiennent que du code.** Les brouillons et les
documents de planification sont commités sur la branche `planning` (voir ci-dessus).

Structure imposée par le template Zed (`.github/pull_request_template.md`) et par
`CLAUDE.md` du projet :

```markdown
<!-- Titre PR : impératif, capitalisé, sans préfixe conventionnel, sans ponctuation finale.
     Préfixe de crate autorisé : "git_ui: Add ..." -->

# Objective
Fixes #NNNNN  (si applicable)
- Le problème traité.

## Solution
- L'approche retenue, et pourquoi celle-là.

## Testing
- Ce qui a été testé, comment, sur quelle plateforme.
- Comment un relecteur reproduit.

## Self-Review Checklist
- [ ] ...  (repris du template Zed)

## Showcase
- Capture / GIF si changement visuel.

---

Release Notes:

- Added ...
```

### ⚠️ Politique IA de Zed — à lire avant d'ouvrir la moindre PR

`CONTRIBUTING.md` (l. 86-94) est explicite :

- « **we don't accept contributions from autonomous agents** » — les PR qui en ont
  l'air peuvent être fermées **sans préavis**.
- « **Don't rely on LLMs to write the whole thing for you when communicating with the
  maintainers** (meaning replies to comments, **PR descriptions**, and alike). »
- Zed attend « **a human in the loop who genuinely understands the work** ».

**Conséquence directe sur `.pr/NN-slug.md` :** je le rédige comme un **brief technique
factuel** (ce qui change, pourquoi, comment tester) — c'est une **matière première pour
toi**, pas un texte à copier-coller tel quel. Le texte de la PR doit être le tien, dans
ta voix. Sinon on tombe pile dans ce que la politique interdit.

Rappel de ton `CLAUDE.md` global : **aucun `push`, `gh pr create` ou commentaire public
sans ton « vas-y » explicite dans le message courant.** Je travaille en local, point.

### Environnement de build (NixOS)

La machine n'a ni `cmake`, ni `pkg-config`, ni les en-têtes X11/Wayland. Sans eux :

- `cargo check` échoue sur `wasmtime-c-api-impl` (« failed to spawn `cmake` ») ;
- `./script/clippy` échoue en plus sur `x11` (build en `--release --all-features`).

**Ce qui ne marche pas :**

- `nix develop` sur le flake du repo → recompile tout l'arbre de dépendances depuis
  les sources (dérivations `cargo-src-*`). Inexploitable en boucle de dev.
- `nix shell nixpkgs#...` → met bien les binaires dans le `PATH`, mais **ne pose pas
  `PKG_CONFIG_PATH`**. `pkg-config` se lance et ne trouve aucun `.pc`.

**Ce qui marche** — `nix-shell -p` monte un vrai stdenv, qui configure `pkg-config` :

```sh
nix-shell -p cmake pkg-config protobuf perl curl fontconfig freetype openssl \
  sqlite zlib zstd libgit2 alsa-lib glib libxkbcommon wayland libglvnd libdrm \
  vulkan-loader libx11 libxcb libxcomposite libxdamage libxext libxfixes \
  libxrandr xorgproto \
  --run './script/clippy -p <crates>'
```

Liste dérivée de `nativeBuildInputs` + `buildInputs` dans `nix/build.nix`.
Vérification rapide que l'environnement est bon : `pkg-config --modversion x11`.

Pour les tests et `cargo check`, `cmake` seul suffit :
`nix shell nixpkgs#cmake --command cargo test -p git_ui`.

> Plus confortable à terme : ajouter ces paquets au profil NixOS, ou committer un
> `shell.nix` perso **dans le fork uniquement** (jamais dans une branche proposée
> upstream).

### Définition de « terminé » pour une étape

1. `./script/clippy` passe (pas `cargo clippy`), lancé comme ci-dessus.
2. Tests unitaires sur le nouveau comportement (les opérations git ont des tests
   d'intégration réels dans `repository.rs`, cf. `test_commit_runs_git_hooks`).
3. `FakeGitRepository` implémenté (sinon tout le reste du projet ne compile plus).
4. Chemin distant (SSH/collab) fonctionnel, pas seulement local.
5. `.pr/NN-slug.md` écrit.
6. Action documentée dans `docs/src/git.md` + keymap si pertinent.

---

## 2. Le coût réel d'une nouvelle opération git

Vérifié en traçant `GitReset` de bout en bout. **Toute** nouvelle commande git touche
**8 endroits** — c'est ça qui dimensionne chaque étape, pas la commande git elle-même :

| # | Fichier | Quoi |
|---|---|---|
| 1 | `crates/git/src/repository.rs` | déclaration dans le trait `GitRepository` |
| 2 | `crates/git/src/repository.rs` | impl dans `RealGitRepository` (la vraie commande) |
| 3 | `crates/fs/src/fake_git_repo.rs` | impl dans `FakeGitRepository` (~70 fns déjà) |
| 4 | `crates/proto/proto/git.proto` | message RPC |
| 5 | `crates/proto/proto/zed.proto` | champ dans l'enveloppe (**numéro à ne jamais réutiliser**) |
| 6 | `crates/proto/src/proto.rs` | **3 sites** : `messages!`, paire requête/réponse, entity id |
| 7 | `crates/project/src/git_store.rs` | méthode `Repository` via `send_job` + handler `handle_*` + branche distante |
| 8 | `crates/git/src/git.rs` + UI + keymap + docs | action, menu, raccourci |

**Ce coût fixe est une tentation à laquelle il ne faut pas céder.** Comme les 8 points
sont les mêmes quelle que soit la commande, on est tenté d'en grouper cinq dans une même
branche « puisque c'est presque gratuit ». C'est vrai pour l'effort de développement,
**et faux pour l'acceptation** : c'est exactement ce qui a tué #57800 et #61719.

Le coût fixe se paie donc à chaque PR, en connaissance de cause. Voir §3.0.

---

## 2bis. Bilan au 22/08/2026 — 24 étapes livrées

Toutes testées, mergées dans `main`, poussées sur `origin`. `upstream` jamais touché.

| Domaine | Livré |
|---|---|
| Diff | fichier entier dans les vues multi-fichiers |
| Commits | branche depuis un commit, réutilisation de message, surlignage des siens |
| Tags | créer (léger + annoté), supprimer, pousser, checkout |
| Séquenceur | bandeau d'état, abort, continue, skip |
| Historique | merge, rebase, reset (4 modes), cherry-pick, revert |
| Log | filtres auteur/date/regex, comparaison depuis un commit |
| Staging | **à la ligne**, stash sélectif |
| Outils | console des commandes, patch copier/appliquer |
| Correctifs | `LC_ALL=C` (bug préexistant), lecture des stages d'index |

### Ce qui reste, et pourquoi

**Éditeur 3 volets** — le backend est livré (`load_unmerged_stages`). L'UI de la PR
#57800 (1007 lignes) ne se cherry-picke pas : au moins **six ruptures d'API** depuis
mai, dont `highlight_rows` passé en pointeur de fonction, qui rend sa palette de
couleurs inexprimable sans modifier `editor`. C'est un portage, pas une reprise.

**Bisect, reflog, LFS** — zéro demande upstream (ni discussion ni issue).
**Submodules, revue PR** — chantiers Zed déjà en cours (#61859 mergée, #61311 en vol).

### Deux erreurs de méthode payées deux fois

- **`Entity::update` sur une entité *forte* renvoie la valeur, pas un `Result`.**
  Pas de `?`. Commis aux étapes 05, 08, et à nouveau plus tard.
- **Ne jamais changer de branche pendant qu'une validation tourne.** Elle compile
  l'arbre de travail : `stash-paths` a rendu un faux négatif portant sur du code
  d'une autre branche. Écrire du code pendant une validation est sans risque ;
  basculer de branche ne l'est pas.

## 3. La roadmap — ~40 PR calibrées pour être acceptées

### 3.0 Le critère de découpage a changé

La version précédente de ce document groupait les opérations proches « parce que ça coûte
à peine plus cher » (§2 : 8 points de passage mutualisés). **C'est optimiser mon effort,
pas l'acceptation.** Corrigé ici : chaque PR est calibrée pour un relecteur, pas pour moi.

**Ce que dit la donnée observée sur les PR git de Zed :**

| PR observée | Taille | Sort |
|---|---|---|
| #61443 | **+176 / 3 fichiers** | Approuvée |
| #62254 | +522 / 7 fichiers | Staff engagé, revue promise |
| #61719 | +895 / 10 fichiers | **Renvoyée : « somewhat big for a first contribution »** |
| #57800 | +2233 / 15 fichiers | **Ignorée 3 mois** |

→ **Cible : < 300 lignes, 3-5 fichiers, une seule chose visible, avec tests.**
Au-delà de ~500 lignes, la probabilité de revue s'effondre.

**Trois règles qui en découlent :**

1. **Pas de PR « socle backend » sans contrepartie visible.** L'ancien « Lot 0 » qui
   ajoutait rebase + cherry-pick + revert + merge + tag d'un coup est **abandonné**.
   Le backend arrive **par tranche, avec la fonctionnalité qui le consomme**.
2. **Une PR = une commande git**, pas une famille. `merge` et `rebase` sont deux PR.
3. **Chaque PR cite sa demande upstream** (numéro de Discussion + votes). C'est
   l'argument d'acceptation le plus efficace : on ne propose pas une idée, on répond
   à une demande existante.
4. **Vérifier que la demande n'a pas déjà été livrée.** Un compteur de votes élevé ne
   dit rien sur l'état d'implémentation : D#33773 (⬆109) et D#55060 étaient déjà
   traitées upstream. Lire la fin du fil et chercher le réglage dans le code **avant**
   de planifier l'étape.
5. **`cargo check` sur les crates modifiés, pas seulement sur ceux dont on dépend.**
   L'étape 01 a compilé sur `git_ui` alors qu'elle cassait `settings_ui`, qui n'en est
   pas une dépendance. Seul `./script/clippy` l'a vu.
6. **Repasser `cargo fmt` en dernier, après la toute dernière édition.** Un `fmt` lancé
   trop tôt puis suivi d'une correction laisse du code non formaté dans le commit.
7. **`Entity::update` sur une entité *forte* renvoie la valeur directement**, pas un
   `Result` — pas de `?` derrière. Sur une `WeakEntity`, si. Erreur commise deux fois
   (étapes 05 et 08).
8. **Chercher la réutilisation avant d'ajouter une commande git.** Pousser un tag est
   un `push` avec un refspec `refs/tags/…` : ~180 lignes au lieu de ~450, et un numéro
   d'enveloppe économisé. Idem pour toute opération qui n'est qu'une variante de refspec.
9. **Dans `repository.rs`, chaque commande apparaît deux fois** — déclaration du trait
   et implémentation — avec une signature identique. Ancrer une insertion sur la
   signature nue vise la déclaration ; ancrer sur un fragment **avec corps**.
10. **Un script d'édition doit asserter qu'il a trouvé son ancre.** Un remplacement de
    chaîne qui ne mord pas échoue *silencieusement*. Arrivé à l'étape 12 : `cargo fmt`
    avait reformaté un import entre l'écriture du script et son exécution, l'ancre ne
    correspondait plus, et l'import manquant n'a été vu que par `./script/clippy`.
11. **`cargo check` ne suffit pas : `./script/clippy` compile `--all-features
    --all-targets`.** Des chemins entiers (doubles de test, code sous `cfg(test)`) ne
    sont compilés que là. Et le lancer **après** le `cargo fmt` final, pas avant.

Classes de taille : **XS** < 150 l. · **S** 150-300 · **M** 300-500 · **L** > 500 (à éviter).

Colonne « Upstream » : ✅ à proposer · ⏸ à retenir (collision) · 🏠 fork seulement
(aucune demande upstream — inutile de consommer du capital de revue).

---

### Vague 1 — Sans backend : établir la crédibilité

Aucune de ces PR ne touche `crates/git`, le proto ou `FakeGitRepository`. Risque nul,
revue rapide, et elles installent le fait qu'on livre du propre avant de demander
qu'on nous suive sur des sujets lourds.

| # | Branche | Contenu | Taille | Demande | Upstream |
|---|---|---|---|---|---|
| 01 | `git/full-file-diff` | ✔ **Fait.** Fichier entier dans les diffs **multi-fichiers** (`git.multi_file_diff.show_full_file`). Le mono-fichier était déjà livré en v1.6.3. | S | **D#33773 ⬆109** | ✅ |
| ~~02~~ | ~~`git/per-file-diff`~~ | **Déjà livré upstream** — `git_panel.entry_primary_click_action: project_diff / file_diff / view_file`. Rien à faire. | — | D#55060 | — |
| 03 | `git/branch-from-commit` | ✔ **Fait.** « New Branch from Here » dans le menu contextuel de commit (graphe + History). `create_branch(name, base)` existait déjà → quasi pure UI | S | D#53112 ⬆45 | ✅ |
| 04 | `git/highlight-authored` | ✔ **Fait.** Nom d'auteur en `Color::Accent` pour ses propres commits, identité lue par dépôt (`git config user.email`) | XS | #41 | ✅ |
| 05 | `git/commit-message-history` | ✔ **Fait.** Picker « Reuse Commit Message… » alimenté par le log du dépôt (pas de persistance nouvelle) | S | #13 | ✅ |

### Vague 2 — Les tags : la première commande, sans risque

Choisis en premier parmi les commandes parce qu'un tag **ne peut pas produire de conflit
ni d'état intermédiaire**. C'est du CRUD pur : la traversée des 8 points du §2 se fait
sur le sujet le plus inoffensif possible.

| # | Branche | Contenu | Taille | Demande | Upstream |
|---|---|---|---|---|---|
| 06 | `git/tag-create` | ✔ **Fait.** Tags légers **et** annotés depuis le menu contextuel de commit. **Première traversée des 8 points du §2** — mesurée à +450/13 fichiers | M | **D#46168 ⬆20** | ✅ |
| 07 | `git/tag-delete` | ✔ **Fait.** Suppression locale, avec confirmation. **Enveloppe à renuméroter en 481 avant proposition** si 06 est déjà en amont | S | D#46168 | ✅ |
| 08 | `git/tag-push` | ✔ **Fait.** Pousser un tag, via un refspec `refs/tags/…` passé à `push` — **aucune nouvelle commande git ni message proto** | S | D#62532 | ✅ |
| 09 | `git/tag-checkout` | Checkout d'un tag. Détaché de l'étape 08 : le HEAD détaché est un sujet d'UX à part | XS | D#53730 | ✅ |

### Vague 3 — L'état du dépôt

| # | Branche | Contenu | Taille | Demande | Upstream |
|---|---|---|---|---|---|
| 10 | `git/sequencer-state` | ✔ **Fait.** Bandeau nommant l'opération en cours. Passe par `UpdateRepository` → **aucun numéro d'enveloppe**. Note : git ≥2.26 rapporte tout rebase comme *interactif* | M | implicite | ✅ |
| 11 | `git/sequencer-abort` | ✔ **Fait.** Bouton Abort, avec confirmation nommant ce qui est perdu | S | implicite | ✅ |
| 12 | `git/sequencer-continue` | ✔ **Fait.** Continue / Skip, avec les asymétries de git respectées (merge sans Skip, bisect sans les deux). `GIT_EDITOR=true` obligatoire | M | implicite | ✅ |

> 09 se justifie seule : savoir *pourquoi* le dépôt est dans un état bizarre a de la
> valeur même sans pouvoir agir. 10 se justifie seule : abandonner depuis Zed un rebase
> lancé au terminal.

### Vague 4 — Une commande par PR

| # | Branche | Contenu | Taille | Demande | Upstream |
|---|---|---|---|---|---|
| 12 | `git/merge` | `git merge <branche>` depuis le branch picker. `--no-ff`, `--ff-only`, `--squash` | M | **D#50549 ⬆61** + D#38296 ⬆24 | ✅ |
| 13 | `git/rebase` | Rebase **non interactif** : `<upstream>`, `--onto` | M | D#26516 ⬆22 | ✅ |
| 14 | `git/reset-hard` | Étendre `ResetMode` (`Soft`/`Mixed` seulement aujourd'hui, `repository.rs:634`) avec `Hard`/`Keep`, sur un commit arbitraire. **Confirmation obligatoire** (cf. perte de données #62535) | S | D#53112 | ✅ |
| 15 | `git/cherry-pick` | Multi-commits, `-x`, `--no-commit` | M | **⬆0** | 🏠 |
| 16 | `git/revert` | `--no-commit`, `-m <parent>` | M | **⬆0** | 🏠 |

> 15 et 16 n'ont **aucune demande upstream** — ni discussion, ni issue. On les fait pour
> toi, sans dépenser de capital de revue. Si l'occasion se présente plus tard, on les
> proposera adossées au menu contextuel (D#53112 ⬆45), pas seules.

### Vague 5 — Conflits

| # | Branche | Contenu | Taille | Demande | Upstream |
|---|---|---|---|---|---|
| 17 | `git/unmerged-stages` | Porter `load_unmerged_stages` / `merge_file_diff3` de **git2 → binaire `git`** (`git show :1:` / `:2:` / `:3:`). **Contrepartie visible : bouton « Use Base » dans l'UI inline existante.** | S | #34813 | ✅ |
| 18 | `git/merge-editor` | L'onglet 3 volets de **#57800** (1580 l., 0 conflit), derrière le réglage `git_panel.merge_editor` déjà présent. **Inclut F5** (`mark_as_resolved`) | **L** | **D#49245 ⬆55** | ✅ |
| 19 | `git/conflict-navigation` | Conflit suivant / précédent | S | — | ⏸ #61719 |
| 20 | `git/conflict-accept-all` | Appliquer tous les non conflictuels (+ gauche seul / droite seul) | S | **D#41019 ⬆16** | ✅ |
| 21 | `git/conflict-magic-wand` | Baguette magique « résoudre les conflits simples » | S | D#41019 | ✅ |
| 22 | `git/conflict-file-list` | Vue des fichiers en conflit, Accept Yours / Theirs en masse. **Panneau, pas modale** | M | — | ✅ |

> **17 est la clé de la reprise de #57800** : c'est le seul vrai obstacle (19 lignes git2),
> et l'adosser au bouton « Use Base » lui donne une valeur propre. Une fois 17 mergée,
> 18 devient une PR purement UI sur un backend déjà accepté — beaucoup plus facile à faire
> passer que les 2233 lignes d'origine.
>
> **18 reste hors budget (L).** C'est assumé : une fonctionnalité cohésive de 1580 lignes
> ne se découpe pas sans la casser. Elle passe *après* 17, 20 et 21 pour arriver devant un
> relecteur qui a déjà accepté trois morceaux du même chantier.
>
> **19 est retenue** : #61719 la couvre déjà. On ne repropose que si elle meurt.

### Vague 6 — Graphe et log

| # | Branche | Contenu | Taille | Demande | Upstream |
|---|---|---|---|---|---|
| 23 | `git/log-filter-date` | Filtre par plage de dates | S | D#59144 ⬆19 | ✅ |
| 24 | `git/log-filter-regex` | Recherche par expression régulière | S | D#59361 | ✅ |
| 25 | `git/log-filter-branches` | Plusieurs branches, par remote, masquer les distantes | S | **D#56028 ⬆25**, D#60972 | ✅ |
| 26 | `git/compare-commits` | Sélectionner 2 commits → diff. `diff_tree` existe déjà | M | D#56854 | ✅ |
| 27 | `git/push-to-commit` | « Pousser jusqu'à ce commit » + dialogue de push (cible, tags, force-with-lease, aperçu) | M | #46 | ✅ |

> **Le filtre auteur est absent de cette liste** : c'est #61443, déjà approuvée. On la
> laisse atterrir, et 23-25 s'y ajoutent une par une au lieu de la doubler par un gros
> « filtres combinables » qui l'écraserait.

### Vague 7 — Réécriture fine

| # | Branche | Contenu | Taille | Demande | Upstream |
|---|---|---|---|---|---|
| 28 | `git/sequence-editor` | Piloter le todo-list via `GIT_SEQUENCE_EDITOR`. **Zone à risque** : Zed traîne des bugs ici (#56215, #44604, #46488, #61278) | M | — | ✅ |
| 29 | `git/reword` | Reformuler un commit arbitraire | S | D#26716 | ✅ |
| 30 | `git/fixup-into` | `--fixup` + `--autosquash` vers un commit choisi | S | D#26716 | ✅ |
| 31 | `git/squash-commits` | Fusionner N commits sélectionnés | S | D#26716 | ✅ |
| 32 | `git/drop-commit` | Supprimer un commit | S | D#26716 | ✅ |
| 33 | `git/interactive-rebase-ui` | Liste réordonnable, action par ligne, split de commit | **L** | **D#26716 ⬆74** | ✅ |

> **29-32 contiennent l'essentiel de la valeur du rebase interactif sans construire
> l'UI de glisser-déposer** — chacune est une PR de 200 lignes qui génère son todo-list
> automatiquement. 33 ne fait plus qu'ajouter la manipulation directe par-dessus.

### Vague 8 — Staging fin

| # | Branche | Contenu | Taille | Demande | Upstream |
|---|---|---|---|---|---|
| 34 | `git/hunk-checkbox` | Case à cocher cliquable par hunk dans la gouttière | S | D#51076 ⬆14 | ✅ |
| 35 | `git/line-staging` | Stager **ligne par ligne**, y compris dans un fichier neuf. Le point dur : `set_index_text` reconstruit le blob entier | **L** | **D#35073 ⬆24** | ✅ |
| 36 | `git/stash-paths` | Stash de fichiers choisis. `stash_paths` existe déjà ; le message est fait (#62439) | XS | D#46791 ⬆21 | ⏸ #62254 |
| 37 | `git/patch-create-apply` | Créer un `.diff`, appliquer un patch | M | **⬆0** | 🏠 |
| 38 | `git/mark-resolved` | — | — | — | **absorbée par 18** |

### Vague 9 — Le reste

| # | Branche | Contenu | Upstream |
|---|---|---|---|
| 39 | `git/submodules` | Statut, diff, navigation. **Zed a démarré** (#61859 mergée) — s'y raccrocher | ✅ D#30727 ⬆30 |
| 40 | `git/command-log` | Console des commandes git exécutées | ✅ |
| 41 | `git/bisect` | Bisect | 🏠 ⬆0 |
| 42 | `git/reflog` | Récupérer un commit perdu | 🏠 ⬆0 |
| 43 | `git/lfs` | LFS — cadrage à faire (ta question reste ouverte) | 🏠 |
| 44 | `git/pr-review` | Revue de PR in-IDE | ⏸ #61311 |

---

## 4. Ordre d'exécution

**Principe : la crédibilité avant le volume.** On ne présente pas `merge-editor` (L) à un
relecteur qui ne nous connaît pas. On arrive avec cinq PR propres derrière soi.

```
Vague 1  01 02 03 04 05        sans backend, risque nul
   ↓
Vague 2  06 07 08              tags — 1re traversée des 8 points, sujet inoffensif
   ↓
Vague 3  09 ──► 10 ──► 11      état du séquenceur
   ↓
Vague 4  12 merge   13 rebase   14 reset-hard      │ 15 cherry-pick  16 revert
         (⬆61)      (⬆22)       (dépend de 03)     │      → fork, ⬆0
   ↓
Vague 5  17 unmerged-stages ──► 18 merge-editor
              └──► 20 accept-all ──► 21 magic-wand ──► 22 file-list
   ↓
Vague 7  28 sequence-editor ──► 29 30 31 32 ──► 33 interactive-rebase-ui
```

**Insérables à tout moment, sans dépendance :**
Vague 6 (23-27), 34, 37, 39, 40.

**Chemin critique** : `09 → 10 → 28 → 29..32 → 33`. Seule chaîne longue du plan.

**Deux PR hors budget assumées** : 18 (merge-editor) et 33 (rebase interactif). Toutes
deux arrivent tard, précédées de plusieurs PR acceptées sur le même chantier.

**Six PR qui ne partent pas upstream** (15, 16, 37, 41, 42, 43) : zéro demande amont,
elles vivent dans le fork. C'est le bénéfice concret de la stratégie hybride — on ne
mendie pas une revue pour ce qui n'intéresse que toi.

---

## 5. Décisions actées

| Sujet | Décision |
|---|---|
| Fichier de message | **`.pr/NN-slug.md`** |
| Ordre des étapes | À ma main — celui du §4 |
| Stratégie | **Hybride** : proposer la MR upstream ; si refus, la branche reste dans le fork |
| `main` | **= le fork** (`origin` = `clement-songis/zed`), remis à niveau sur `upstream/main` |

### Conséquences de la stratégie hybride sur la façon de coder

`main` du fork est désormais la **branche de référence**, pas un miroir d'upstream :
elle contient tout, mergé ou non par Zed. Ça impose trois choses.

1. **Remote `upstream` ajouté** (`https://github.com/zed-industries/zed.git`, lecture seule).
   Rythme conseillé : `git fetch upstream && git merge --ff-only upstream/main` avant
   chaque nouvelle branche, tant que `main` n'a pas divergé. **Une fois la première
   étape mergée, le `--ff-only` ne passera plus** → on bascule sur
   `git merge upstream/main` dans `main`, régulièrement, pour que la dérive reste petite.
   La leçon de #57800 : 3 mois de retard = 1586 commits = 17 blocs de conflit.

2. **Chaque branche part de `main` et doit rester proposable seule.** Donc pas de
   dépendance implicite d'une étape sur une autre non encore mergée upstream. Quand
   l'étape N s'appuie sur l'étape N-1 non mergée par Zed, la MR de N doit le dire
   (« depends on #XXXX ») — sinon elle est incompréhensible pour un relecteur.

3. **Rester petit est un impératif, pas un confort.** Zed a explicitement reproché sa
   taille à #61719 (+895) : « *somewhat big for a first contribution and currently
   clashes a bit with our repository contribution guidelines* ». Le découpage du §3 est
   calibré là-dessus.

---

## 6. Évaluation des PR upstream en collision

### 6.1 PR #57800 — Merge 3 volets : **récupérable, et largement**

**Mesuré**, pas estimé : `git merge-tree` en dry-run contre `main` à jour.

| Indicateur | Valeur | Lecture |
|---|---|---|
| Base de la PR | 2026-05-23, **1586 commits de retard** | 3 mois de dérive |
| Fichiers en conflit | **9 / 15** | |
| Blocs de conflit | **17**, ~576 lignes | Volume modéré |
| **`merge_editor.rs` (1580 l., 70 % de la PR)** | **0 conflit** | **Fichier neuf → atterrit intact** |
| `.expect()` en code de prod | **3**, tous sur des invariants documentés | À nettoyer, trivial |
| `let _ =` sur des faillibles | **0** | Conforme au `CLAUDE.md` du projet |
| Tests | 8 dans `merge_editor.rs` + 1 d'intégration | Correctement testée |

**Le vrai obstacle n'est aucun de ceux-là : c'est `git2`.**

La PR ajoute `load_unmerged_stages` et `load_index_text` en s'appuyant sur `git2`
(lecture des stages 1/2/3 de l'index). Or **upstream a supprimé `git2` entièrement**
depuis — absent de `crates/git/src/repository.rs` *et* du `Cargo.toml` du workspace.
En mai il y était encore (19 occurrences à la base de la PR), tout passe aujourd'hui par
l'appel au binaire `git`.

**Portée du problème : 19 lignes sur les 238 ajoutées au backend.** La réécriture consiste
à remplacer les lectures `git2` par `git show :1:<path>` / `:2:` / `:3:`
(ou `git ls-files -u`). L'API publique ajoutée est petite et bien dessinée :

```rust
pub struct UnmergedStages { pub base: Option<String>, pub ours: Option<String>, pub theirs: Option<String> }
fn load_unmerged_stages(&self, path: RepoPath) -> BoxFuture<'_, UnmergedStages>;
fn merge_file_diff3(&self, path: RepoPath) -> BoxFuture<'_, Result<String>>;
```

**Bonus non anticipés** — la PR contient déjà plus que E1 :

- **Le réglage `git_panel.merge_editor: false` existe déjà**, les deux modes cohabitent
  et l'inline reste le défaut. **C'est exactement ton arbitrage → E2 est fait.**
- `mark_as_resolved()` / `unstage_resolution()` + détection d'une résolution externe →
  **c'est F5.**
- `accept_all()` → **une partie de E3.**
- `toggle_base()` : afficher/masquer le volet base.
- `is_merge_creating_subcommand()` : force `merge.conflictStyle=diff3` **uniquement** sur
  les commandes qui créent un merge, avec tests unitaires. Détail soigné.
- `impl Item for MergeEditor` → c'est un **onglet de workspace, pas une modale**.
  Conforme à ton « tout dans l'éditeur, pas de modales bloquantes ».

**Verdict : reprendre la branche.** Réimplémenter jetterait 1580 lignes qui atterrissent
sans conflit, 8 tests, et un design déjà aligné sur tes arbitrages. Le travail réel est :
réécrire ~19 lignes git2 → CLI, résoudre 17 blocs mécaniques, nettoyer 3 `.expect()`.

→ **E1 et E2 fusionnent en une étape, et F5 disparaît** (absorbée).

### 6.2 Les trois autres — cohérentes, mais à traiter différemment

| PR | État réel | Ce qu'on en fait |
|---|---|---|
| **#61443** — *Author queries in Git Graph search* (+176/-60, 3 f.) | Approuvée par **@rubiin (communauté, pas Zed)**. L'auteur écrit lui-même : « *I see this PR as a small first step* ». Un commentaire demande déjà une refonte plus large de l'UX de recherche. | **Cohérente mais volontairement minimale.** Notre **C2** est le sur-ensemble qu'appelle la discussion (auteur + date + regex + multi-branches + remote + chemin). On construit **par-dessus**, en la créditant. Si elle stagne, C2 la remplace. |
| **#61719** — *Resolve merge conflicts from the keyboard* (+895/-25, 10 f.) | **Renvoyée par un mainteneur Zed** (@MrSubidubi) : trop grosse pour une première contribution, « *clashes a bit with our contribution guidelines* ». Sans activité depuis le 29/07. | **Cohérente sur le fond, mal calibrée sur la forme.** Elle valide notre **E3** — mais confirme aussi qu'il faut le **découper**. On reprend l'idée, pas le format. |
| **#62254** — *Tracked / Staged options to stash* (+522/-14, 7 f.) | **Zed s'y intéresse activement** : @ChristopherBiscardi (staff) — « *Conceptually the feature seems reasonable. Will try to give a review soon.* » 12 commentaires, MAJ le 14/08. | **La seule des trois qui va probablement atterrir.** → **On ne double pas F3.** On attend, et F3 se réduit à ce qui manquera après son merge. |

### 6.3 Effet sur le plan

- **E1 + E2 → une seule étape** `git/merge-editor` (reprise de #57800). **F5 absorbée.**
- **F3 mise en attente** de #62254 (déjà à moitié faite par #62439 sur le message).
- **E3 confirmée mais à découper** en tranches plus fines que #61719.
- **C2 maintenue** en sur-ensemble de #61443.
- Nouvelle sous-étape préalable à E1 : **`git/unmerged-stages`** — porter
  `load_unmerged_stages` / `merge_file_diff3` de git2 vers le binaire `git`.
  Isolable, testable seule, et c'est le seul vrai obstacle de la reprise.
