# Client Git de Zed vs JetBrains — rapport d'écart et plan

Base analysée : `crates/git` (backend), `crates/project/src/git_store.rs` (état/RPC),
`crates/git_ui` (UI, ~40k lignes), `crates/git_ui_core` (worktrees, askpass).

Légende : ✅ déjà présent — 🟡 partiel — ❌ absent

---

## 0. Ce que Zed a déjà (à ne pas refaire)

- **Panel Git** avec onglets `Changes` / `History`, vue liste **ou arbre**, tri/groupement
  configurables, diff-stats par entrée, badge de compteur.
- **Cases à cocher tri-état** (`Selected` / `Unselected` / `Indeterminate`) déjà implémentées
  par fichier et par section — `git_panel.rs:7402-7407`.
- **Staging au hunk** via `set_index_text` (`ToggleStaged`, `StageAndNext`, `StageRange`).
- **Éditeur de commit** inline + modal, `--amend` (HEAD), `--signoff`, `--no-verify`,
  co-auteurs, templates, **génération IA du message**, brouillons persistés par branche.
- **Git Graph** en onglet dédié : lanes colorées, recherche incrémentale, détails du commit
  en split, fichiers modifiés en liste/arbre, colonnes masquables, ordres `date/topo/author`.
- **Diff** : multibuffer projet, **split (côte à côte) ou unifié**, **word-diff**,
  vues dédiées staged / unstaged / branch-diff / file-history / solo-diff.
- **Conflits inline** dans l'éditeur : `Use <ours>` / `Use <theirs>` / `Use Both` /
  **`Resolve with Agent`** (`conflict_view.rs:323-420`).
- **Worktrees** de première classe (création, checkout, renommage, suppression, picker) —
  **en avance sur JetBrains**.
- Branches (create/rename/delete/checkout), remotes CRUD, stash (all/pop/apply/drop),
  push (`--set-upstream`, `--force-with-lease`), pull, pull --rebase, fetch, blame,
  permalinks, `.gitignore` / `info/exclude`, clone, init, multi-repo.

**Verdict :** l'ossature (backend job-queue, RPC collaboratif, multibuffer diff, graphe) est
solide. L'écart avec JetBrains est concentré sur **la réécriture d'historique**, **le merge
3 volets**, **les changelists**, et **les boîtes de dialogue d'options**.

---

## 1. Sélection des modifications & staging

| # | Fonction JetBrains | Zed | Détail |
|---|---|---|---|
| 1 | **Changelists** nommées (plusieurs listes, active, déplacer des fichiers entre listes) | ❌ | Zed n'a que l'index git. Pas de notion de groupe logique. | C'est non
| 2 | **Shelve / Unshelve** (patch hors-git, par fichier, réappliquable) | ❌ | Seul `git stash` existe, sans message ni sélection UI. | Passer tout par des statsh
| 3 | **Cocher ligne par ligne** dans le diff pour un commit partiel | ❌ | Zed est au **hunk**, pas à la ligne. | C'est oui
| 4 | Case à cocher **par hunk** directement dans la vue diff | 🟡 | Action `ToggleStaged` sur le hunk, mais pas de case cliquable dans la gouttière. | C'est oui
| 5 | Arbre de fichiers à cases tri-état | ✅ | Déjà là. |
| 6 | Stash **partiel** (fichiers choisis) + **message** | 🟡 | `stash_paths` existe côté backend, aucune UI ; pas de message. | C'est oui
| 7 | **Créer un patch** `.diff` / **appliquer un patch** (fichier ou presse-papier) | ❌ | — | C'est oui
| 8 | Marquer un fichier "résolu" / `git add` sur conflit | 🟡 | Passe par le staging générique. | C'est oui

## 2. Commit

| # | Fonction JetBrains | Zed | Détail |
|---|---|---|---|
| 9 | `--amend` sur HEAD | ✅ | `git::Amend`, toggle persistant. |
| 10 | **Amend d'un commit arbitraire** (`--fixup` + `--autosquash`) | ❌ | Le plus demandé côté JetBrains ("Fixup into commit"). | C'est oui
| 11 | **Reword** d'un commit arbitraire (F2 dans le log) | ❌ | Amend HEAD seulement. | C'est oui
| 12 | Surcharger l'**auteur** (`--author`) | ❌ | — | Pas nécessaire dans l'imédiat
| 13 | **Historique des messages** de commit (déroulant) | 🟡 | Brouillon par branche persisté, pas d'historique consultable. | C'est oui
| 14 | **Checks avant commit** (reformat, optimize imports, analyse, TODO, tests) | ❌ | Seuls les hooks git natifs. | C'est non
| 15 | `--signoff`, `--no-verify` | ✅ | Toggles dans le menu commit. |
| 16 | Co-auteurs, templates | ✅ | — |

## 3. Réécriture d'historique — **le plus gros trou**

Aucune de ces commandes n'existe dans `crates/git` (vérifié : 0 occurrence de
`rebase`, `cherry-pick`, `revert`, `merge`, `tag`, `bisect`, `reflog`).

| # | Fonction JetBrains | Zed | Détail |
|---|---|---|---|
| 17 | **Rebase interactif GUI** (glisser-déposer pour réordonner, pick/squash/fixup/reword/drop/edit) | ❌ | — |
| 18 | Rebase simple (`onto`, `--onto`, `continue`/`abort`/`skip`) | ❌ | `PullRebase` existe, mais pas de rebase de branche. |
| 19 | **Cherry-pick** depuis le log | ❌ | — |
| 20 | **Revert** d'un commit | ❌ | — |
| 21 | **Reset** `--hard` / `--keep` depuis le log, sur un commit choisi | 🟡 | Backend : `--soft` / `--mixed` seulement ; UI : `Uncommit` sur HEAD. |
| 22 | **Squash** de N commits sélectionnés | ❌ | — |
| 23 | **Drop** / **Edit** d'un commit | ❌ | — |
| 24 | **Split** d'un commit | ❌ | — |
| 25 | **Bandeau d'état** rebase/merge en cours + continue/abort | ❌ | Aucun indicateur `REBASE-i 3/7`. |
C'est tout oui
## 4. Merge & résolution de conflits

| # | Fonction JetBrains | Zed | Détail |
|---|---|---|---|
| 26 | `git merge <branche>` (`--no-ff`, `--squash`, `--abort`) | ❌ | Impossible de fusionner une branche depuis Zed. |
| 27 | **Éditeur 3 volets** (Local / Résultat éditable / Distant) | ❌ | Zed = marqueurs inline dans le fichier. |
| 28 | **Baguette magique** « Resolve simple conflicts » | ❌ | — |
| 29 | **Appliquer tous les changements non conflictuels** (+ gauche seul / droite seul) | ❌ | — |
| 30 | **Chevrons `»` / `«` par changement** pour accepter | 🟡 | Boutons texte sur le bloc entier, pas de flèches par sous-changement. |
| 31 | Dialogue **« Files Merged with Conflicts »** : Accept Yours / Accept Theirs en masse | ❌ | — |
| 32 | Navigation **conflit suivant / précédent** | ❌ | Aucun raccourci par défaut. |
| 33 | Résolution **inline dans le vrai fichier** | ✅ | Avantage Zed (pas de modale bloquante). |
| 34 | **Résolution par agent IA** | ✅ | Avantage Zed, absent de JetBrains. |
C'est oui mais peut être tout faire dans l'éditeur à la manière de Zed et non pas des modales bloquantes.
## 5. Log / graphe

| # | Fonction JetBrains | Zed | Détail |
|---|---|---|---|
| 35 | Graphe multi-lanes + détails commit | ✅ | `git_graph.rs`, bon niveau. | Coté GitKraken c'est encore mieux
| 36 | **Filtres combinables** : branche, **auteur**, **date**, chemin, regex | 🟡 | `LogSource` = All/Branch/Sha/Path + une seule requête texte (`SearchCommitArgs { query, case_sensitive }`). Pas de filtre auteur/date, pas de regex. | C'est oui et encore plus pousser que jetbrains serait le mieux je pense, notement filtre sur plusieurs branches, par remote ou "dossier si submodules ou plusieurs .git", recherche de diff dans les commits ?
| 37 | **Panneau arbre des branches** dans le log (favoris, locales/distantes) | ❌ | Branch picker en modale uniquement. | Dans Zed c'est un onglet Git Graph maintenant c'est encore mieux.
| 38 | **Comparer deux commits sélectionnés** | ❌ | Un seul commit à la fois. | C'est un grand oui trop utile
| 39 | **Parcourir le dépôt à une révision** (snapshot du projet) | ❌ | — | Bof et un peu déjà fait par fichier/modifs
| 40 | Go to hash / branch / tag | 🟡 | Recherche par SHA (7-40 hex) OK ; pas de saut branche/tag. | Si y'a une recherche c'est déjà pas mal
| 41 | Surligner « mes commits » / commits non cherry-pickés | ❌ | — | C'est oui
| 42 | **Menu contextuel riche sur un commit** (cherry-pick, revert, reset, checkout, tag, branche depuis ici) | 🟡 | Actuel : View Diff, Copy SHA/Ref/Tag, Show in Git Graph. | C'est oui

## 6. Branches, remotes, tags

| # | Fonction JetBrains | Zed | Détail |
|---|---|---|---|
| 43 | **Tags** : créer / supprimer / checkout / pousser | ❌ | Les tags sont seulement *affichés*. | C'est oui
| 44 | Merge / rebase **depuis le popup de branche** | ❌ | Le picker ne fait que checkout/rename/delete. | C'est oui
| 45 | Dialogue **« Update Project »** (merge vs rebase, stash auto des modifs) | ❌ | `Pull` / `PullRebase` bruts, pas de gestion des modifs locales. | Peut être pas un dialog y'a déjà un dropdown
| 46 | **Dialogue Push** : cible par dépôt, tags, force-with-lease, **aperçu des commits poussés** | ❌ | Actions de menu uniquement. | C'est oui, d'ailleur faudra aussi un push jusqu'a ce commit dans la vue Git Graph
| 47 | **Branches protégées** (garde-fou force-push) | ❌ | — | C'est non Git et les hébergeurs c'en ocupent 
| 48 | Créer une branche **depuis un commit du log** | ❌ | `create_branch(name, base_branch)` existe au backend, pas exposé au log. |  C'est oui
| 49 | **Worktrees** | ✅ | Avantage Zed net. | Y'a aussi sur idea + on peut aller sur le worktree depuis la branche correspondante et tout reregarde l'implémentation y'a des trucs à prendre coté jetbrains
| 50 | Remotes CRUD | ✅ | — |

## 7. Divers

| # | Fonction JetBrains | Zed | Détail |
|---|---|---|---|
| 51 | **Local History** (historique local indépendant de git) | ❌ | Des `checkpoints` existent mais servent à l'agent IA, pas exposés. | C'est oui/gamechanger
| 52 | **Bisect** | ❌ | — | C'est oui
| 53 | **Reflog** / « Recover lost commits » | ❌ | — | C'est oui
| 54 | **Submodules** | ❌ | — | C'est oui
| 55 | **Git LFS** | ❌ | — | C'est oui mais comment c'est gèrer coté jetbrains qui améliore l'experience c'est pas juste git qui fait tout ?
| 56 | **Console des commandes git exécutées** | 🟡 | `git_runtime_diagnostics.rs` existe, pas un vrai journal de commandes. | C'est oui
| 57 | Blame / annotate (gouttière + vue) | ✅ | — |
| 58 | Intégration PR/MR | 🟡 | « Create Pull Request » ouvre le navigateur ; pas de revue in-IDE. | C'est oui

---

## 8. Plan de travail proposé

Découpage en **6 lots**, ordonnés par dépendance et par rapport valeur/coût.
Chaque lot est indépendamment livrable (PR séparée).

### Lot 0 — Socle backend (prérequis de tout le reste)
Étendre le trait `GitRepository` (`crates/git/src/repository.rs`) + le passe-plat
`git_store.rs` + le proto pour le mode collaboratif :
- `rebase`, `rebase_continue/abort/skip`, `rebase_interactive` (via `GIT_SEQUENCE_EDITOR`
  pointant sur un binaire Zed qui écrit le todo-list généré par l'UI)
- `cherry_pick`, `revert`, `merge`, `reset --hard/--keep`
- `create_tag`, `delete_tag`, `push_tags`
- `sequencer_state()` → détecte `.git/rebase-merge`, `MERGE_HEAD`, `CHERRY_PICK_HEAD`
- Enrichir `SearchCommitArgs` : auteur, plage de dates, regex, chemin

*Sans ce lot, rien de la réécriture d'historique n'est possible.*

### Lot 1 — Actions sur commit dans le graphe et le panel
Menu contextuel complet sur un commit : cherry-pick, revert, reset (soft/mixed/hard),
checkout, créer branche/tag ici, reword, fixup into. Bandeau d'état
« Rebase en cours 3/7 — Continuer / Abandonner ».
→ **Débloque 80 % de la valeur JetBrains pour ~20 % de l'effort.**

### Lot 2 — Merge 3 volets
Nouvel `Item` `MergeView` : trois `Editor` synchronisés (Local read-only / Résultat
éditable / Distant read-only), chevrons par sous-changement, barre d'outils
« Appliquer tous les non conflictuels / gauche / droite », baguette magique,
navigation conflit suivant/précédent, dialogue de liste des fichiers en conflit avec
Accept Yours / Accept Theirs en masse.
S'appuie sur `ConflictSet` (`project/src/git_store.rs:1817`) et sur le split diff
existant (`editor/src/split.rs`).
→ Le lot le plus lourd en UI. **Garder l'inline actuel en parallèle** (avantage Zed).

### Lot 3 — Rebase interactif GUI
Modale listant les commits à rebaser, réordonnables au clavier et à la souris, avec
action par ligne (pick / reword / squash / fixup / drop / edit) et aperçu du résultat.
Génère un todo-list transmis à git via `GIT_SEQUENCE_EDITOR`.
Inclut squash de N commits sélectionnés, split de commit.

### Lot 4 — Changelists & shelve
Modèle de données `Changelist { name, description, paths }` persisté en base workspace,
rendu dans le panel comme sections repliables au-dessus de Tracked/Untracked.
Shelve = `git diff` sérialisé en `.patch` dans le stockage Zed + vue de réapplication.
→ Gros changement conceptuel dans `git_panel.rs`. **À arbitrer : est-ce que tu veux
vraiment ce modèle, ou tu préfères rester sur l'index git ?**

### Lot 5 — Dialogues d'options & polish
- Dialogue Push (cible, tags, force-with-lease, aperçu des commits)
- Dialogue Update Project (merge/rebase + stash automatique des modifs locales)
- Stash partiel avec message, patch create/apply
- Filtres du log (auteur, date, regex) + arbre des branches en panneau latéral
- Comparaison de deux commits, snapshot du projet à une révision
- Console des commandes git

### Hors périmètre proposé (à confirmer)
Bisect, reflog UI, submodules, LFS, Local History, checks avant commit
(reformat/tests) — faible ratio valeur/coût, ou redondant avec l'écosystème Zed.

---

## 9. Questions ouvertes

1. **Changelists (lot 4)** : c'est le point le plus divergent philosophiquement.
   Zed est aligné sur l'index git, JetBrains sur des listes logiques. On les ajoute
   ou on reste sur l'index ? Non reste sur l'index
2. **Merge 3 volets (lot 2)** : en remplacement de l'inline, ou en plus (choix par
   réglage `git.merge_editor: "inline" | "three_way"`) ? Règlage + les 2 types
3. **Staging à la ligne (#3)** : nécessaire, ou le hunk suffit ? Evidement c'est ultra utile et limite le faire aussi pour quand on ajoute un fichier (ne pas commit certaines lignes)
4. **Checks avant commit (#14)** : hooks git seulement, ou intégration formatter/tests Zed ? hooks git seulement
5. Ordre de priorité entre les lots 1, 2 et 3. Fait comme tu le sens

Vue des diffs par fichier par défaut (setting pour activer) et voir tout le fichier d'un coup pas juste les lignes autour du diff
---

# 10. État de l'art upstream (zed-industries/zed) — août 2026

## 10.0 Découverte structurante

**Les feature requests de Zed ne vivent PAS dans les issues, mais dans les Discussions**
(8032 au total). Les issues sont réservées aux bugs : une demande de fonctionnalité
est systématiquement `converted_to_discussion` par les mainteneurs — ce qui la marque
`closed / completed` alors qu'elle n'est **pas** implémentée.

*Exemple :* l'issue #51522 « Auto-resolve non-conflicting changes in merge conflicts »
apparaît comme `state: closed, reason: completed` — en réalité elle a été **convertie
en discussion**. Ne jamais se fier au statut des issues fermées pour juger de ce qui existe.

→ **Le signal de demande réelle, ce sont les upvotes des Discussions.**

Le label `area:integrations/git` ne compte que **120 issues ouvertes**, presque toutes
des bugs (perfs, worktrees, encodages, scan de fichiers).

## 10.1 Demande upstream, croisée avec nos 58 points

| Notre # | Fonction | Demande upstream | Vol. |
|---|---|---|---|
| — | **Full File View dans le diff** (ta demande ligne 206) | D#33773 + D#56213, D#55060 (diff par fichier) | **⬆109** |
| 51 | **Local History** | **D#24004**, D#54484, D#23793 (« IntelliJ-like compare with branch/revision/local history » ⬆53) | **⬆273** |
| 49 | Worktrees | D#26084 (⬆139, largement livré), D#54966 ⬆20, D#56758, D#57326, D#52406 | ⬆139 |
| 17 | **Rebase interactif** | **D#26716**, D#26516 ⬆22, D#42183 ⬆15, D#56631 ⬆12 | **⬆74** |
| 26 | **Merge de branche** | **D#50549**, D#38296 ⬆24, issue #57982 | **⬆61** |
| 27 | **Merge 3 volets** | **D#49245**, issues #34813 (11 💬), #58974 | **⬆55** |
| 42 | **Menu contextuel commit** | **D#53112**, D#53593 ⬆19, D#56631 ⬆12, D#56630 | **⬆45** |
| 36 | Filtres du log | D#56028 ⬆25, D#59144 ⬆19, D#59361, D#60972, D#62230 | ⬆25 |
| 3 | Staging à la ligne | D#35073 (⬆24 « stage selected »), D#51076 ⬆14, issue #45295 (10 💬), D#43116, D#26523 | ⬆24 |
| 6 | Stash de fichiers choisis | D#46791 | ⬆21 |
| 43 | **Tags (UI)** | D#46168, D#53730, D#62532 | ⬆20 |
| 54 | Submodules | D#30727 ⬆30, D#48659 ⬆17, D#31689 ⬆17, D#26700 ⬆10 | ⬆30 |
| 28/29 | Baguette magique / auto-résolution | D#41019 (« one-click Use HEAD for all conflicts »), issue #51522 | ⬆16 |
| 58 | Revue de PR in-IDE | D#26960 ⬆12, D#60336 ⬆12 | ⬆12 |
| 38 | Comparer deux commits | D#56854 | ⬆7 |
| 1 | **Changelists** | D#51827 | ⬆7 |
| 19/20 | Cherry-pick, revert | **aucune discussion dédiée** | ⬆0 |
| 52/53 | Bisect, reflog | **aucune discussion, aucune issue** | ⬆0 |
| 7 | Patch create/apply | **aucune discussion** | ⬆0 |
| 55 | Git LFS | seulement le bug #21126 (hunks parasites) | — |

**Lecture :** ta liste est très bien calibrée sur la demande réelle. Deux écarts notables :

- **Tu as dit non aux changelists (#1)** → aligné, c'est la demande la plus faible (⬆7).
- **Local History (#51), que tu qualifies de « gamechanger », est LA demande n°1
  de tout le domaine (⬆273)** — et elle est indépendante de git. À remonter en priorité.
- Cherry-pick, revert, bisect, reflog, patch : **zéro demande upstream**. Ça ne veut pas
  dire inutile (toi tu les veux), mais ça ne sera pas un argument de merge upstream.

## 10.2 Travail déjà en vol — à ne pas refaire

### PR ouverte majeure : le merge 3 volets existe déjà

**PR #57800 « Git 3-way merge editor » (@miguelvr)** — `+2233 / -17`, 15 fichiers,
ouverte le 2026-05-27, **dernière activité 2026-06-29, aucune revue Zed**.
C'est notre lot 2 quasi entier. À lire avant d'écrire une ligne.
Débat en cours dans les commentaires : 3 colonnes (façon JetBrains) vs résultat en bas
(façon VSCode) — **ton choix « réglage + les 2 types » tranche ce débat**, c'est un
argument fort si tu reprends cette PR.

### Autres PR ouvertes qui chevauchent nos lots

| PR | Titre | Recoupe |
|---|---|---|
| #62173 | `git_ui: Add a diff base menu and a Committed section to the git panel` — **par @mikayla-maki (core Zed)** | Structure du panel — **risque de conflit fort, à surveiller** |
| #61443 | `Support author queries in Git Graph search` | #36 (filtre auteur) |
| #61719 | `git_ui: Resolve merge conflicts from the keyboard` | #32 (navigation conflits) |
| #62254 | `Added Tracked, Staged options to stash` | #6 (stash partiel) |
| #61111 | `git_ui: Add multi-repository Git panel view` | multi-repo (D#52191 ⬆38) |
| #62161 | `git_ui: Add collapsible sections to git panel tree view` | structure du panel |
| #61507 | `Show staged or unstaged diff in solo diff view` | D#61505 ⬆15 |
| #61318 | `git: Diff a file with its own history` | #38 (comparaison) |
| #61311 | `Add inline visualization of pull request comments` | #58 (revue PR) |
| #62615 | `git_ui: Enable navigation for single-hunk diffs` | navigation diff |
| #62679 | `git_graph: Show time in commit details` | graphe |

### Déjà mergé récemment (à intégrer au rebase de notre base)

- **#62439 (2026-08-11) — `Add optional message support to git stash`** → notre #6 est
  **à moitié fait** : le message existe déjà, il ne reste que la sélection de fichiers.
- #61529 (2026-07-23) — `--no-verify` dans l'UI.
- #61501 (2026-07-31) — `diff_base` : voir les changements depuis la branche par défaut.
- #61859 (2026-07-30) — `Register Git submodules under their own project path` → le
  chantier submodules (#54) **a démarré côté Zed**.
- #61304 — largeur de gouttière git configurable ; #61185 — fix timeouts + hook commit-msg.

## 10.3 Ce que ça change pour le plan

1. **#57800 devient le point de départ du lot 2**, pas une réécriture. Décision à prendre :
   reprendre la branche (rebase sur `main`) ou réimplémenter en s'en inspirant.
2. **Le lot 0 (socle backend) reste intégralement à faire** : personne, dans aucune PR
   ouverte ou mergée, n'a touché à `rebase` / `cherry-pick` / `revert` / `merge` /
   `tag` dans `crates/git`. Le champ est libre — et c'est le chemin critique.
3. **Local History (#51) mérite son propre lot, en tête** : ⬆273, indépendant de git,
   aucune PR en vol, et l'infra existe déjà à moitié (`checkpoint()` /
   `create_archive_checkpoint()` dans `repository.rs`, aujourd'hui réservés à l'agent IA).
4. **Full File View (⬆109) est un quick win** à isoler tout de suite : c'est du réglage
   d'expansion de contexte dans le multibuffer, pas du git.
5. **Surveiller #62173 (mikayla-maki, core Zed)** : elle restructure le panel avec un
   menu de base de diff et une section « Committed ». Nos lots 1 et 5 touchent la même zone.
6. Zed **ne relit pas les PR communautaires rapidement** (#57800 : 1 mois sans réponse,
   #34813 ouverte depuis juillet 2025). À intégrer dans la stratégie : viser des PR
   petites et autonomes plutôt qu'un gros bloc.

## 10.4 Ordre proposé (révisé)

| Lot | Contenu | Justification |
|---|---|---|
| **A** | Full File View + diff par fichier (réglages) | ⬆109, isolé, aucun conflit |
| **B** | Socle backend : rebase / cherry-pick / revert / merge / tag / reset --hard / état du séquenceur | Chemin critique, champ libre |
| **C** | Menu contextuel commit + bandeau d'état rebase/merge | ⬆45+, débloque le plus de valeur |
| **D** | Merge 3 volets — **repartir de #57800** + réglage inline/3-volets + baguette magique | ⬆55, code existant |
| **E** | Rebase interactif GUI | ⬆74, dépend de B |
| **F** | Local History | ⬆273, indépendant, infra à moitié là |
| **G** | Staging à la ligne + patch + stash partiel | ⬆24 ; stash déjà à moitié fait (#62439) |
| **H** | Filtres du log, comparaison de 2 commits, tags UI, push jusqu'à un commit | ⬆25/⬆20 ; coordonner avec #61443 |
| **I** | Submodules, LFS, bisect, reflog, console git, revue PR | demande faible ou chantier Zed déjà démarré |
