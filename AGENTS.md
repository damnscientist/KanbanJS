# KanbanJS — Projet anti-procrastination

Fichier unique `index.html` (~6000 lignes) + `sw.js` (service worker, 26 lignes). Aucune dépendance, pas de bundler. S'ouvre dans un navigateur moderne. Publié sur GitHub Pages : https://damnscientist.github.io/KanbanJS/

Ce fichier décrit l'état actuel, les conventions et la roadmap. L'historique des versions est dans [`CHANGELOG.md`](CHANGELOG.md) — à ne lire que pour retrouver le pourquoi d'une décision passée.

## Concept

Transformer une tâche lourde en micro-actions à coût cognitif nul via IA. L'utilisateur décrit une corvée → un agent d'IA la décompose en actions minuscules (30s-3min), créées dans une liste "Suggestions" dédiée. L'utilisateur les glisse ensuite dans "In Progress" → "Done".

## Positionnement

Le public naturel est les gens avec TDAH, les étudiants qui procrastinent, les freelances submergés. L'angle "fichier unique, pas de compte, données locales" est un argument fort face aux Trello/Todoist. Le projet est fonctionnellement complet pour un usage personnel. Pour le rendre accessible à d'autres, le travail restant est surtout la distribution (P0 de la roadmap) et l'UX d'onboarding (P1), pas du code.

## Analyse du projet

### Forces

**Positionnement & concept**
- Fichier unique, zéro dépendance, zéro compte — différenciant fort face à Trello/Todoist. Le format monolithique est un choix assumé : il simplifie le développement par agents d'IA (pas de build, pas de modules à chercher, tout le contexte est dans un seul fichier)
- Ciblage précis : TDAH, procrastinateurs, freelances. L'UX est pensée pour le public (quotes bienveillantes, streak, confettis, WIP limit) — pas un énième todo générique
- **Templates-first** : l'IA est un bonus, pas un prérequis. 12 templates de corvées universelles avec matching fuzzy, zéro barrière à l'entrée
- Onboarding intégré : liste "Tuto" + "Ranger une surface" avec 6 micro-tâches, premier drag & drop en <10s

**Architecture technique**
- Séparation DB / DnD / App propre, chaque module a une API publique bien définie
- `mutate(fn)` comme seul point d'entrée pour les snapshots undo — convention sans ambiguïté
- Moteur DnD générique robuste (seuil 4px, cache `getBoundingClientRect`, auto-scroll, `pointercancel`)
- Abstraction IA multi-fournisseur (OpenAI, Anthropic, Google, Ollama) centralisée dans `queryAI`
- Undo/redo 30 niveaux avec streak synchronisé, reconstruction DOM sans `location.reload()`
- Parsing robuste du JSON IA (équilibrage bracketing, détection guillemets, strip code fences)
- Gestion d'erreur complète : bannière `QuotaExceededError`, crash recovery dans `init()`, `AbortController` 60s
- Tests : 234 tests via iframe/postMessage (`test.html`) — DB, fichier compagnon, multiboard, interface (DOM de l'iframe)

**UX & design**
- Raccourcis clavier vim-like complets : navigation h/l/j/k, actions carte/liste, recherche `Ctrl+K`, undo/redo
- 8 thèmes (4 sombres, 4 clairs) avec mode système et 3 clés de persistance
- Streak + compteur du jour + paliers emoji (👍⚡🔥🚀💪) + confettis canvas à 100% — boucle motivationnelle
- 50 quotes bienveillantes en rotation, WIP warning, repli de listes, markdown riche dans les notes, tags colorés
- Accessibilité : `aria-label` sur 20+ boutons, `label[for]`, `prefers-reduced-motion`

### Faiblesses

**Persistance & sécurité**
- **localStorage = perte totale au `clear` navigateur** ou changement de machine. Atténué par la copie de secours (`kanbanjs:state.bak`), le self-heal, le stockage persistant, le **fichier compagnon** (Chrome/Edge/Opera) et le **repli export manuel + rappel** pour les autres navigateurs. **Safari et Firefox n'implémentent pas la File System Access API ; Brave la désactive par défaut** (activable dans `brave://flags`)
- **Sync multi-onglets partielle** — les changements distants sont répercutés (`storage` event → debounce → reload/rebuild), mais les conflits d'écriture simultanée restent en « dernier écrivain gagne »
- **Clé API en clair** dans `localStorage` — lisible par toute extension ou script cross-origin. Volontairement **exclue** du fichier compagnon

**Fonctionnel**
- Pas de **capture mobile** — `companion.md` reste une spec non implémentée. Décision assumée (roadmap #7, **abandonné**) : KanbanJS est un outil desktop, la capture rapide depuis le téléphone n'a pas été un besoin ressenti
- Pas de **dates d'échéance** sur les cartes — utile pour les freelances avec deadlines
- Pas de **notifications** — le streak est silencieux hors de l'onglet

**Architecture — points d'attention (non bloquants)**
- `refreshHeaderInfo()` appelé après chaque mutation (création, suppression, move, cut/paste, toggle). ✅ **Corrigé (2026-09-20)** — debounce de 50 ms : `refreshHeaderInfo()` n'est plus qu'un planificateur, le rendu est fait par `_applyHeaderInfo()`
- `extractCardData()` lit le DOM (`textContent`, `value`) plutôt que la DB — en cas de désynchronisation, le clipboard contient des données périmées
- `_rebuild()` reconstruit tout le DOM à chaque undo/redo/reset/import. Correct pour la taille actuelle (~50-100 cartes), deviendrait coûteux au-delà

## Fonctionnalités existantes

### Board
- **Plusieurs boards** indépendants (listes, cartes, historique undo, fichier de sauvegarde propres). Le board d'origine est le board **principal** (`main`, non supprimable). Menu `▾` à côté du nom (raccourci `b`) : filtre, changement, création. Configuration → onglet **Boards** : « compte pour le streak », seuil de suivi, suppression. Un nouveau board démarre en Backlog / In Progress / Done, sans Tuto ni gamification.
- Renommage du board (input dans le header, `maxlength="200"`)
- Header info : nombre de listes et cartes
- Thème jour/nuit persisté dans 3 clés (`kanbanjs:theme-mode`, `kanbanjs:theme-dark`, `kanbanjs:theme-light`)
- Sélecteur de thème dans la modale Configuration (onglet "Thèmes") : 4 sombres, 4 clairs
- Bouton reset (remet le **board actif** à son gabarit, reconstruction DOM)
- Export / Import JSON du board actif (boutons ↓ ↑ dans le header ; fichier nommé d'après le board pour les boards secondaires)
- Indicateur de progression : barre + pourcentage dans le header (`3/8`, `38%`)

### Clavier
- Raccourcis vim-like : minuscule = carte, majuscule = liste
- Navigation : `h`/`l` listes ←/→, `j`/`k` cartes ↓/↑, `1-9` focus liste n
- Actions carte : `n` créer (fallback Backlog si aucune liste sélectionnée), `r` renommer (dblclick), `e` ouvrir notes (cycle édition→preview→fermé), `i` entrée de journal datée, `c` dupliquer, `t` ajouter un tag, `x` couper, `y` copier, `p` coller, `d` marquer terminée
- Boards : `b` ouvre le menu des boards (filtre, ↑↓, Entrée, Échap)
- Actions liste : `N` créer, `R` renommer, `D` toggle terminé, `C` replier/déplier, `X` couper, `Y` copier, `P` coller
- `E` plier/déplier toutes les notes (preview), `Ctrl+drag` = copier une carte au lieu de déplacer
- `Esc` désélectionne la carte/liste courante ou ferme l'overlay actif, `?` affiche l'aide clavier
- Presse-papier unifié : couper/copier une liste copie titre + cartes
- Sélection visuelle : bordure accent, actions toujours visibles sur carte sélectionnée
- Ignore automatiquement quand un input/textarea a le focus ou une modale est ouverte
- `Ctrl+K` : recherche fuzzy sur les cartes du board (texte, notes, tags, titre de liste)
- `u` undo, `Ctrl+Y` redo : 30 niveaux de snapshots par board (session uniquement)

### Listes
- Création inline (bouton + formulaire "Ajouter une liste")
- Renommage (clic sur le titre)
- Suppression (confirmation)
- Défaut au premier lancement : "Tuto", "Backlog", "In Progress", "Waiting", "Done"
- Drag & drop souris pour réordonner horizontalement (DnD générique)
- **Liste "terminé"** : chaque liste a un bouton `✓` qui la marque comme "terminée".
  Les cartes d'une liste "terminée" s'affichent barrées/grissées.
  Par défaut, la liste "Done" est marquée terminée.
  Le statut se met à jour automatiquement au drag & drop entre listes.
  Les cartes d'une liste "terminée" affichent aussi la date de complétion.
- **WIP warning** : bordure et badge accent quand une liste "WIP" ou "In Progress" contient ≥ 3 cartes
- **Repli** : bouton ▾/▸ pour replier/déplier une liste (cartes + footer masqués), état persisté

### Cartes
- Création inline ("Ajouter une carte" dans chaque liste)
- Édition inline (textarea avec Sauver/Annuler, Enter valide, Escape annule)
- **Notes** : chaque carte a un bouton `≡` qui ouvre une textarea de note personnelle en édition. Cycle 3 états : édition (≡, `e`) → preview markdown (Esc) → fermé (Esc ou ≡). Rendu markdown (gras, italique, code inline, liens, titres, listes, ligne horizontale). Pastille colorée sur les cartes ayant des notes.
- Suppression
- Drag & drop souris entre listes (ou au sein de la même liste)
- Drag ghost + placeholder visuel

- **Journal daté** : `i` ou bouton 📅 → entête `### AAAA-MM-JJ` en haut de la note, curseur dessous ; une insertion laissée vide est annulée.
- **Projets non suivis** : carte atténuée + badge `⏳ N j` si la note n'a pas bougé depuis le seuil du board (`notesUpdatedAt`, 0 = désactivé, défaut 14 j sur les nouveaux boards, 0 sur le principal).
- **Tags** : chaque carte a un champ `tags: []`, chips colorés (couleur par hash du nom), bouton `+ tag`, raccourci `t`, tags préservés au copier/coller/dupliquer, recherchables via `Ctrl+K`.

### IA (Décomposition)
- FAB flottant `＋` en bas à droite
- Modal "Décomposer une tâche" : si l'API est configurée → textarea + bouton Décomposer (IA) + suggestion template fuzzy. Si pas d'API → grille des 12 templates cliquables + lien vers la config.
- **Abstraction fournisseur** : supporte OpenAI/compatible, Anthropic, Google Gemini, Ollama (switch dans `queryAI`)
- Appel POST à l'API avec format spécifique par fournisseur
- Prompt système : décomposition en micro-actions anti-procrastination
- Parsing robuste de la réponse JSON (équilibrage des crochets, détection des guillemets, strip code fences)
- Résultat : liste "IA" créée en position 0 avec les cartes générées
- Panneau de debug toggleable (logs requête/réponse/parsing)
- **Templates** : 12 templates de corvées universelles intégrés, matching fuzzy sur le texte saisi dans le textarea (💡 "Template suggéré"). Zéro API requise.

### Persistance
- `localStorage` :
  - `kanbanjs:state` → board principal (listes, cartes) ; `kanbanjs:state:<id>` → boards secondaires
  - `kanbanjs:state.bak` / `kanbanjs:state:<id>.bak` → copie de secours (état précédent), utilisée par le self-heal de `DB._load()`
  - `kanbanjs:backup` / `kanbanjs:backup:<id>` → `{ firstSeen, lastExport }` pour le repli export manuel
  - `kanbanjs:boards` → index `[{ id, streak, staleDays }]`, écrit à la création du 2ᵉ board ; `kanbanjs:board-active` → dernier board utilisé (aussi en `sessionStorage`, par onglet)
  - `kanbanjs:config` → provider, endpoint, apiKey, model
  - `kanbanjs:streak` → `{ streak, lastDate, todayCount, todayDate }`, unique pour tous les boards (export/import inclus seulement pour un board qui compte pour le streak)
  - Thème (3 clés) :
    - `kanbanjs:theme-mode` → 'dark' | 'light' | 'system'
    - `kanbanjs:theme-dark` → 'warm-night' | 'deep-ocean' | 'forest' | 'tokyo-night'
    - `kanbanjs:theme-light` → 'soft-sand' | 'mint' | 'lavender' | 'tokyo-light'
- **Self-heal** : si `kanbanjs:state` est illisible ou invalide, `DB._load()` restaure `kanbanjs:state.bak` et répare la clé principale
- **Sync multi-onglets** : un `storage` event (déclenché par un autre onglet) déclenche après 200 ms un `DB.reload()` immédiat puis un `_rebuild()` différé tant qu'une édition est en cours (les données sont rafraîchies pour éviter d'écraser un changement distant par un `blur` sur état périmé ; seul le DOM attend). Les piles undo/redo sont purgées à la synchro (conflits simultanés : dernier écrivain gagne)
- **Stockage persistant** : `navigator.storage.persist()` demandé au premier streak (évite l'éviction implicite par le navigateur)

### Fichier compagnon (`AutoSave`)

Sauvegarde automatique du board dans un fichier choisi par l'utilisateur, via la File System Access API (Chrome, Edge, Opera). Zéro compte, zéro backend : l'app apporte le mécanisme, l'utilisateur apporte le stockage.

- **Activation** : Configuration → onglet **Données** → « Choisir un fichier… » (`showSaveFilePicker`), ou « Ouvrir un fichier existant… » (`showOpenFilePicker`).
- **Handle** : conservé dans IndexedDB (base `kanbanjs`, store `handles`, clé `companion`) avec `lastWriteAt` — survit aux redémarrages. Un `_store` swappable permet de tester sans IndexedDB.
- **Écriture** : `DB.onSave()` → `schedule()` (debounce 800 ms) → `flush()` écrit `exportJSON()` (`{state, streak}`), jamais la clé API. Flush aussi sur `visibilitychange` (onglet caché).
- **États** : `unsupported` (API absente) | `idle` | `active` | `paused` (permission à réactiver) | `error` (écriture échouée).
- **Permission** : Chromium exige un geste utilisateur à chaque session. La puce `💾 Réactiver` du header déclenche `resume()`.
- **Bandeaux** : `restore` au boot si `DB.loadSource() === 'default'` et qu'un handle existe (données locales disparues) ; `reload` si `file.lastModified > lastWriteAt` (fichier modifié ailleurs — multi-machine via dossier synchronisé). Jamais de remplacement silencieux.
- **Règle d'or** : tant qu'un bandeau est affiché (`pending` non nul), **aucune écriture** n'a lieu — `schedule()` et `flush()` refusent d'écrire. « Ignorer » demande confirmation (le fichier sera écrasé par le board local) puis reprend l'écriture. Sans cette règle, la première frappe après un « Ignorer » détruisait la seule sauvegarde existante.
- **Armement sûr** : `attach()` lit et valide le fichier **avant** de mémoriser le handle. Un fichier choisi par erreur n'est jamais écrasé.
- **Robustesse des écritures** : compteur de génération `_gen` (une `disable()` pendant une écriture laisse l'état `idle`, jamais `active`) et drapeau `_dirty` (une écriture déclenchée pendant un flush en vol est rejouée, jamais perdue).
- **Détection externe** : `_externalNewer()` compare `file.lastModified` à `lastWriteAt` avec 1 s de tolérance. Granularité et dérive d'horloge entre machines incluses : détection fiable à la seconde près, pas à la milliseconde.
- **Détection en session** : `checkExternal()` toutes les 60 s (`POLL_MS`, suspendue quand l'onglet est caché) et au retour de visibilité. Garde-fous : jamais pendant un flush (vérifié **avant et après** la lecture du fichier, car un flush peut démarrer pendant l'`await`), jamais si une décision est déjà en attente, jamais sans handle ni en dehors de l'état `active`. Le bandeau n'est levé qu'après **comparaison du contenu** : un fichier simplement « touché » par un client de synchronisation (mtime plus récent, contenu identique) ne suspend pas les écritures. Remplace l'ancienne limite « seulement au boot ou au clic Réactiver ».
- **Sonde de capacité** : `capability()` renvoie `ok` | `insecure` | `no-picker` | `no-idb`. Jamais de sniffing d'UA (Brave se présente comme Chrome) : c'est la présence réelle de l'API qui décide. `isBrave()` (drapeau `navigator.brave.isBrave`) ne sert qu'à choisir le message, jamais à décider du support.
- **Repli export manuel** : quand l'API est absente (`unsupported`) ou non configurée (`idle`), que `DB.hasState()` est vrai et que le dernier export (ou `firstSeen`) date de ≥ 7 jours, une puce `⬇ Sauvegarder` apparaît dans le header et déclenche `exportBoard()`. L'onglet Données affiche « Dernier export… » et « Télécharger une sauvegarde (JSON) ». Rappel **persistant** jusqu'à l'export, sans snooze.
- **Limites assumées du rappel** : il n'est pas réévalué dans la session où le board passe de « jamais modifié » à « modifié » (fenêtre étroite : `firstSeen` ≥ 7 j **et** absence de clé `state`) ; et `lastExport` est horodaté même si le téléchargement est bloqué par le navigateur — aucune API ne permet de le savoir.
- **Pièges** : écriture non atomique (`createWritable` tronque avant d'écrire) ; IndexedDB protégée par un timeout de 8 s pour ne jamais bloquer l'app ; `AutoSave.init()` n'est pas attendu par `App.init()`.

## Architecture

### Modules (dans l'ordre dans le fichier)

| Module | Responsabilité | API publique |
|---|---|---|
| `initTheme` | IIFE, lit/applique/persiste le thème | lecture au load |
| `DB` | Adapter localStorage | `key, activeBoard, listBoards, createBoard, useBoard, deleteBoard, getBoardMeta, setBoardMeta, countsStreak, setCardNotesUpdatedAt, getBoard, renameBoard, getLists, createList, renameList, deleteList, reorderList, toggleListDone, toggleCollapsed, getCards, getAllCards, createCard, updateCard, updateCardNotes, updateCardTags, setCardDoneAt, setCardsDoneAt, deleteCard, moveCard, reload, onSave, loadSource, hasState, getBackupInfo, setBackupInfo, exportJSON, importJSON, reset, restoreJSON, batch` |
| `AutoSave` | Fichier compagnon (File System Access API + handle en IndexedDB) | `init, enable, attach, disable, resume, restore, flush, schedule, dismiss, refreshPending, checkExternal, available, capability, isBrave, switchBoard, forget, status, pending, fileName, lastWriteAt, permission, onChange` |
| `DnD` | Moteur de drag & drop générique | `start(dragEl, id, { ghostEl?, ghostClass?, phClass?, getZone, getAfter, getPos, skip? }, onDrop)` |
| `App` | UI Kanban | `init, switchBoard, createBoard, deleteBoard, setBoardMeta` |

### Multiboard (v1.2.0)

Référence technique, à lire avant de toucher à `DB`, `AutoSave`, aux piles undo ou au changement de board.

- **Stockage — sans migration** : le board existant reçoit l'id réservé `main` et **garde ses clés historiques** (`kanbanjs:state`, `.bak`, `kanbanjs:backup`, handle IndexedDB `companion`). Les nouveaux boards utilisent des clés suffixées (`kanbanjs:state:<id>`, `kanbanjs:state:<id>.bak`, `kanbanjs:backup:<id>`). Index `kanbanjs:boards` = `[{ id, streak, staleDays }]`, écrit seulement à la création d'un second board ; les noms restent dans chaque état (`state.board.name`, source unique). Board actif : `sessionStorage` (par onglet), repli sur `localStorage` (dernier utilisé). L'identifiant d'un board vient de l'index, jamais de `state.board.id` (identique pour tous les états historiques, `'board-1'`). Retour à la v1.1.2 possible tel quel.
- **Index auto-réparé** : `_readIndex()` complète toujours l'index par un scan des clés `kanbanjs:state:*` — un index corrompu ou incomplet ne fait jamais disparaître un board. `deleteBoard()` retire donc les clés **avant** l'index.
- **Board supprimé ailleurs** : `_save()` n'écrit jamais pour un board absent de l'index (pas de clé orpheline), et `DB.reload()` rebascule sur `main` si le board actif a disparu.
- ✅ **Module `DB` livré (2026-11-01)** : `activeBoard`, `listBoards`, `createBoard`, `useBoard` (refusé pendant un `batch`), `deleteBoard` (ni `main` ni le board actif), `getBoardMeta`/`setBoardMeta`, `countsStreak`, `key()` (remplace `DB.KEY`), constantes `MAIN`, `INDEX_KEY`, `STREAK_KEY`. Streak exporté/importé seulement si le board compte pour le streak. `notesUpdatedAt` posé à la création et à chaque changement réel de note (absent des cartes antérieures : à ignorer).
- ✅ **`AutoSave` livré (2026-11-01)** : un fichier par board (handle IndexedDB `companion` pour `main`, `companion:<id>` sinon). Le handle est **lié à un board** (`_boundId`) : `_persist` écrit sous la clé de ce board, `flush`/`schedule`/`checkExternal` refusent d'agir si le board actif n'est plus celui du handle (garde-fou contre tout changement qui contournerait `switchBoard`, ex. `DB.reload()` après suppression distante). `switchBoard(doSwitch)` est **le seul chemin** pour changer de board : il vide l'écriture en attente **et en vol** dans le fichier quitté (boucle jusqu'à stabilité, une édition pendant l'écriture finale n'est pas perdue), détache, change, rattache (`_bind`, aussi utilisé par `init`), puis rattrape un fichier resté périmé (`_catchUp`, comparaison de contenu). `enable`, `attach`, `disable`, `resume`, `restore` et `switchBoard` sont **sérialisés** (`_serial`, file en chaîne de promesses) : un picker ou une demande de permission en cours fait attendre le changement de board. `forget(id)` supprime le handle d'un board non actif. Côté App, `_syncFromStorage` repasse par `switchBoard` quand `DB.reload()` a changé le board actif. La règle d'or tient : un bandeau en attente bloque l'écriture du fichier quitté et réapparaît au retour.
- ✅ **Étape `App` livrée (2026-11-01)** : `App.switchBoard(id)` (refusé pendant une édition, un autre changement ou une synchro ; passe par `AutoSave.switchBoard` ; reconstruit le DOM même si `AutoSave` échoue après le changement), `App.createBoard`, `App.deleteBoard` (oublie handle et piles), `App.setBoardMeta`. **Piles undo par board** (`sessionStorage`, clés historiques pour `main`, suffixées sinon) ; filet `_stacksOk()` si le board actif a changé hors App. Purge des piles d'un board modifié dans un autre onglet, même non actif. **Streak et undo** : les piles stockent des valeurs absolues du streak ; elles sont **neutralisées** (jamais restaurées) quand le streak a changé pendant l'absence du board ou quand on bascule « compte pour le streak » — sinon un `u` effacerait la progression faite sur un autre board. Gamification conditionnée par `DB.countsStreak()` (streak, badge, compteur du jour, confettis, citation) ; un changement de streak dans un autre onglet n'affecte pas un board qui ne compte pas. **Quota `sessionStorage`** : `_saveStack` purge les piles des autres boards puis tronque, et `mutate()` n'est jamais bloqué par un instantané impossible.
- ✅ **Interface livrée (2026-11-01)** : menu des boards (`BoardMenu`) dans l'en-tête — bouton `▾`, raccourci `b` (ferme l'aide clavier si ouverte), combobox ARIA (filtre `role=combobox`, `aria-activedescendant`), filtre insensible à la casse et aux accents, entrée « Créer « … » » masquée si le nom existe déjà, ↑↓ Entrée Échap (Échap ne remonte pas au handler global), fermeture au clic extérieur ou à la perte de focus, focus rendu au bouton. Création sans nom → « Nouveau board » puis focus sur le champ du nom. Onglet **Boards** de la Configuration (`renderBoardsTab`) : streak, seuil de suivi (vide ou invalide → valeur conservée), Supprimer (désactivé pour `main` et le board actif, confirmation) ; re-rendu à chaque ouverture et sur changement d'index. **Garde de bascule** `_switchBlocked()` distincte de `_editingActive()` : les aperçus de notes dépliés et l'aide clavier ne bloquent pas le changement de board (ils bloquent toujours le rebuild de synchro). Retour au principal après suppression du board actif dans un autre onglet : message explicite dans l'en-tête. Nom du fichier d'export suffixé par le nom du board (boards secondaires).
- ✅ **Suivi de projet livré (2026-11-01)** : raccourci carte `i` et bouton 📅 (actions de carte) → entête `### AAAA-MM-JJ` en haut de la note, curseur dessous, pas de doublon le même jour. Une insertion laissée sans saisie est **annulée exactement** à l'enregistrement (drapeau `_journal` = valeur d'origine + valeur insérée, jamais de retrait par motif : une note de l'utilisateur commençant par l'entête du jour reste intacte, minuit sans effet). `saveNotes` n'écrit rien si la note est inchangée (pas d'instantané undo parasite). Badge `⏳ N j` + carte atténuée si `notesUpdatedAt` ≥ seuil du board (`_staleDays`, chargé par `_rebuild`, re-rendu par `App.setBoardMeta`), libellé « Aucune note depuis… » pour une carte sans note, rien en liste terminée (CSS `:not(.card-done)`), recalcul au retour sur l'onglet. `updateCardNotes` compare les notes à fins de ligne normalisées (`\r\n`). Couper/coller conserve `notesUpdatedAt` (`DB.setCardNotesUpdatedAt`), copier/dupliquer/Ctrl+glisser repartent d'une date neuve.
- **Correctifs annexes (hors chantier)** : `duplicateCard` (`c`) construisait la copie avant `updateCardNotes`/`updateCardTags` — note et tags absents de l'affichage, puis **effacés en base** au premier aller-retour ≡ ou `+ tag` (même famille que C7 en v1.1.1, oubliée ici). `autoResize` oubliait l'épaisseur des bordures (`border-box`) : barre de défilement parasite dans toutes les notes.
- ✅ **Audit d'ensemble (2026-11-01)** : la décision « restaurer » est **persistée** dans l'enregistrement IndexedDB du handle (`pending: 'restore'`), relue par `_bind` — elle survit à un changement de board et à un rechargement ; `_resume` ne la lève pas. `_bind` compare le contenu (`_differs`) avant `reload`, comme `checkExternal`. `_enable`/`_attach`/`_readInto` relèvent le board de départ et abandonnent s'il a changé (suppression distante pendant un sélecteur). `App.switchBoard` reconstruit le DOM **dans** le `doSwitch` (avant `_bind`), annule une synchro en attente (`_syncPending`) en purgeant l'historique du board quitté. Changement de l'index dans un autre onglet : `_metaSeen` comparé → neutralisation des piles ou recalcul du seuil. Décomposition IA liée au board de lancement.
- **Règle** : `App.switchBoard(id)` est **le seul chemin** pour changer de board (il passe par `AutoSave.switchBoard`, rebinde les piles undo, reconstruit le DOM). Ne jamais appeler `DB.useBoard()` directement hors tests. Garde de bascule `_switchBlocked()` (édition réelle, modale, recherche) ≠ garde de synchro `_editingActive()` (inclut aussi aperçus de notes et aide clavier).

### Mécanique DnD
- `pointerdown`/`pointermove`/`pointerup` — API unifiée
- Seuil 4px avant démarrage (évite les faux positifs)
- Ghost : clone de l'élément ou `ghostBuilder` optionnel, fixed, `pointer-events:none`
- `getZone(x,y)` : itère sur `getBoundingClientRect()` des cibles (pas de `elementFromPoint`)
- Guard `lastPos` : mutation DOM uniquement si la position a changé
- Cache des `getBoundingClientRect()` construit au démarrage du drag, évite les reflows à chaque `pointermove`
- `cleanup()` après `dropCb()` (capture locale de la callback)
- Handler `pointercancel` + `cleanup()` défensif (libération capture persistante)
- `restoreOrigin()` : un drop hors de toute zone (ou une annulation) remet l'élément à sa position d'origine via `_origParent`/`_origAfter` — au lieu de le laisser à l'emplacement du placeholder, ce qui affichait un déplacement jamais écrit en base. En mode copie (Ctrl), l'original ne bouge jamais.

### Design
- Dark/light mode via `data-theme` sur `<html>` + CSS custom properties
- Modals : overlay `z-index: 20000`, centrées
- FAB : `z-index: 5000`, `position: fixed` bottom-right
- 4 thèmes sombres (Warm Night, Deep Ocean, Forest, Tokyo Night) et 4 clairs (Soft Sand, Mint, Lavender, Tokyo Light)
- Couleurs CSS : `color-mix(in srgb, var(--accent) X%, transparent)` + fallback `rgba(var(--accent-rgb), .X)` pour compatibilité

### Limites techniques
- **Ouverture en `file://`** : l'API IA est bloquée par CORS. L'app fonctionne en `file://` pour le board, les templates et la persistance, mais les appels IA nécessitent d'être servi en `http(s)` — d'où la publication GitHub Pages.
- **Service worker (`sw.js`)** : cache-first, chemins relatifs (`'./'`), guard non-GET (les POST IA ne doivent pas passer par la Cache API). Enregistré dans `index.html` uniquement en `https:`. **Toute modification publiée d'`index.html` ou `sw.js` doit bumper le nom de cache (`kanbanjs-v1` → `v2` → `v3` → `v4` → `v5` → `v6` → `v7`)**, sinon les visiteurs gardent l'ancienne version.
- **Fournisseur IA centralisé** : tout le branchement fournisseur est dans `queryAI` (switch provider → body/headers/parsing)

### Déploiement
- **GitHub Pages** : https://damnscientist.github.io/KanbanJS/ (branche `main`, dossier `/ (root)`)
- Repo : https://github.com/damnscientist/KanbanJS (public, `origin` en SSH)
- Fichiers publiés : `index.html`, `sw.js`, `test.html`, `README.md`, `AGENTS.md`, `CHANGELOG.md`, `companion.md`, `LICENSE`, `VERSION`, `.nojekyll`
- Fichiers de process (prompts `01`–`05`, `audit.md`, `github.md`) : non publiés, purgés de l'historique, ignorés via `.gitignore`. **Ne pas les re-tracker.**
- Auth push : clé SSH ed25519 ajoutée au compte damnscientist. Le remote `origin` est en `git@github.com:...`

### Décisions non retenues
- ~~**Multi-boards** : sur-ingénierie pour un outil mono-utilisateur. Les listes séparent déjà les contextes.~~ **Décision levée (2026-11-01)** : le besoin réel est apparu — suivre les projets d'une équipe (1 carte = 1 projet, suivi dans la note). C'est un usage distinct de l'anti-procrastination personnelle : streak, confettis, citations et Tuto n'y ont pas leur place, et le mélanger au board personnel dans des listes ne tient pas. Voir le chantier **Multiboard** dans la roadmap.
- **Dates d'échéance, notifications, dashboard statistiques, import/export CSV** (ex-P2 #11, #12, #15, #16) : **en attente** (2026-11-01). Ils rapprochent KanbanJS d'un Trello générique sans servir le concept central — casser le blocage de la première étape. À ne reprendre que sur un besoin ressenti.

## Pour reprendre le développement

### Conventions et points d'entrée
- `mutate(fn)` est le seul point d'entrée pour les snapshots undo. Ne jamais appeler `_snapshot()` directement — utiliser `await mutate(() => DB.xxx(...))`.
- `DB.batch(fn)` suspend les écritures `localStorage` le temps de l'exécution, puis persiste en une fois. Utiliser pour les opérations groupées (création multiple de cartes, import de liste avec cartes).
- Les helpers `toggleForm`, `registerOverlay`, `formatDoneAt`, `extractCardData`, `extractListData`, `removeListDom` sont dans le scope App, juste après `removeListDom`.
- **Dates** : toujours passer par `dateKey(date)` (clé `YYYY-MM-DD` **locale**) et `isToday(date)` pour toute logique de jour (streak, compteur, comparaison `doneAt`). Ne jamais utiliser `toISOString().slice(0, 10)` pour une date logique — c'est la date UTC. Seul usage légitime restant : le nom de fichier d'export.
- Le thème est capturé au démarrage de App via `const Theme = window.__themeAPI`. Plus aucun accès à `window.__themeAPI` dans le code métier.
- `Clipboard` est un objet (plus un `let _clipboard`), méthodes : `copyCard/cutCard/copyList/cutList/paste/clear/isEmpty/type`.
- **Documents à tenir à jour** : toute évolution fonctionnelle ou visible du logiciel doit être répercutée dans le `README.md` (fonctionnalités, quick start, raccourcis, nombre de tests) en plus de `CHANGELOG.md`. Le README est le seul document lu par les utilisateurs ; un README périmé est un bug de documentation.

### Architecture du code
- L'ordre des modales et du debug suit le flow : config → décompose
- Le prompt système est dans `PROMPT_SYSTEM` (template literal, substitution `{{TASK}}`)
- La config API (provider, endpoint, apiKey, model) est stockée en localStorage, lue par `readAIConfig()` dans App
- Le fournisseur est branché dans `queryAI` via un switch (`'openai'` / `'anthropic'` / `'google'` / `'ollama'`), chaque branche construit body + headers + parsing de réponse
- Le board est initialisé avec 6 listes par défaut via `DB._default` (dont "Tuto" avec cartes-exemples et "Ranger une surface" avec micro-tâches d'onboarding)
- `buildList(list, cards)` est synchrone — les cartes sont passées en paramètre (pré-fetchées par l'appelant)
- `buildCard(card, done)` est synchrone (reçoit une carte déjà construite)
- Les listes ont un champ `done` (booléen) ; si vrai, leurs cartes affichent `.card-done` (barré + grisé)
- Les cartes ont un champ `notes` (chaîne) persisté via `updateCardNotes`, éditable inline (textarea toggleable)
- Les cartes ont un champ `tags` (array de strings) persisté via `updateCardTags`, affiché en chips colorés (hash du nom), éditable inline (bouton `+ tag`)
- Les cartes ont un champ `doneAt` (ISO string ou null) stocké automatiquement quand glissées dans une liste "terminée"
- `refreshHeaderInfo()` est un planificateur debouncé (50 ms) qui **renvoie une promesse résolue après le rendu** (via `_headerWaiters`) ; le rendu lui-même est fait par `_applyHeaderInfo()`, async. `refreshCountBadge(id)` reste async. Les deux lisent les données via DB (pas le DOM)

### Convention de code
- `make(tag, className)` pour créer des éléments
- `autoResize(ta)` pour les textareas
- `flashMessage(el, msg, cls, ms)` pour les messages de statut temporaires (stocke le timer sur `el._timer`)
- Tableaux de bord en `const`, fonctions helpers en closures
- Pas de commentaires dans le code (sauf en-têtes de sections)
- Noms en français (utilisateur francophone)

### Points ouverts connus (assumés, non planifiés)

Issus des relectures successives. Aucun n'est bloquant ; aucun n'est dans la roadmap.

- **Copie d'une carte vers une liste « terminée »** : le Ctrl+drag pose la classe visuelle « faite » mais **pas** `doneAt`, contrairement au drag normal — donc pas de date de complétion ni de streak. Correctif ≈ 5 lignes dans la branche `copy` de `buildCard`, via `DB.setCardDoneAt`.
- **Rappel d'export non réévalué en session** : `refreshAutoSaveUI()` n'est appelé qu'au boot, sur `AutoSave.onChange`, après un export et à l'ouverture de l'onglet Données. Un board qui passe de « jamais modifié » à « modifié » dans la même session n'affiche la puce `⬇ Sauvegarder` qu'au rechargement suivant. Fenêtre étroite (`firstSeen` ≥ 7 j **et** absence de clé `state`).
- **`lastExport` horodaté sans confirmation** : si le navigateur bloque le téléchargement, le rappel est tu pour 7 jours. Aucune API ne permet de savoir qu'un téléchargement a abouti.
- **Changement distant pendant un glisser-déposer** : `_syncFromStorage()` diffère le `_rebuild()` pendant une édition, mais ne détecte pas un drag en cours. Un changement distant pile à ce moment peut détacher l'élément saisi. Fenêtre très étroite (200 ms de debounce).
- **Undo et streak** : les piles undo stockent des valeurs absolues du streak. Elles sont neutralisées dès que le streak a bougé pendant l'absence du board, ou à la bascule « compte pour le streak » : un undo n'efface jamais la progression faite ailleurs, mais n'ajuste alors plus le streak de la complétion annulée (léger excédent possible du compteur du jour).
- **Boards secondaires après un effacement du `localStorage`** : seul le board principal se voit proposer « Restaurer ». Les handles `companion:<id>` restent en IndexedDB sans être proposés ; recours : recréer le board puis « Ouvrir un fichier existant… ». Une restauration des handles orphelins reste à faire.
- **Badge de non-suivi** : recalculé au rendu et au retour sur l'onglet, pas en continu. Un Ctrl+glisser (copie) repart d'une date neuve, un couper/coller la conserve.
- **Confettis** : un aller-retour `100 % → < 100 % → 100 %` en moins de 50 ms (auto-répétition clavier) est manqué par le debounce de `refreshHeaderInfo()`.
- **`localStorage` hors du module `DB`** : `streak`, `config` et les 3 clés de thème sont lus/écrits directement depuis `App` et `initTheme`. Assumé, mais la frontière « tout passe par `DB` » n'est pas tenue.

Voir aussi la section **Faiblesses** pour `extractCardData()` (lit le DOM) et le coût de `_rebuild()`.

## Roadmap

### Prochains chantiers (ordre arrêté le 2026-11-01)

Après six semaines d'usage quotidien sans incident, l'ordre retenu est le suivant. La traduction passe **avant** le mode focus pour que les nouvelles fonctionnalités naissent avec le système de traduction, au lieu d'être migrées après coup.

0. **Bugs visuels** — remontés au fil de l'eau par l'utilisateur, traités en priorité dès qu'ils arrivent.
1. ✅ **Multiboard (v1.2.0)** — **Livré (2026-11-01)**. Plusieurs boards indépendants pour suivre les projets d'une équipe à côté du board personnel (1 carte = 1 projet, suivi dans la note : journal daté `i`, badge de non-suivi). Référence technique : section **Multiboard** de l'Architecture ; historique : `CHANGELOG.md`.
2. **Traduction (i18n)** — Interface et README en anglais en plus du français. Dictionnaire de chaînes `t(key)` dans le fichier unique, langue détectée via `navigator.language` et surchargeable dans Configuration. Inclut le prompt système IA, les 12 templates, les 50 citations et le Tuto. Premier levier de distribution : le public visé est majoritairement anglophone.
3. **Mode focus « une seule carte »** — Masquer le board et n'afficher que la prochaine micro-action (première carte de In Progress, sinon de la liste sélectionnée), avec un gros bouton « Fait » qui passe à la suivante. Un board de 50 cartes peut lui-même être paralysant pour le public TDAH : le mode focus retire le choix.
4. **Minuteur de démarrage** — Compte à rebours de 2-3 min sur la carte en cours (règle des 2 minutes). Pensé avec le mode focus, utilisable hors focus.
5. **Son à la complétion** (ex-P2 #13) — Feedback auditif optionnel (toggle), généré via Web Audio (zéro fichier, zéro dépendance).
6. **Fournisseur LM Studio** (ex-P2 #17) — voir le détail en P2.
7. **Checklist de distribution** — voir ci-dessous.

### Checklist de distribution

Le produit est prêt ; il n'a pas encore de public. Rien ici n'exige de compte, de tracker ni de backend — la philosophie « données locales » vaut aussi pour la mesure d'audience.

**Vitrine**
- [ ] README : capture d'écran + GIF de 15 s (décomposition → premier drag → confettis) en tête de fichier
- [ ] README anglais (après le chantier i18n), lien croisé FR ↔ EN
- [ ] Section « Vie privée » explicite : rien n'est envoyé nulle part, sauf au fournisseur IA choisi
- [ ] Dépôt GitHub : description, topics (`adhd`, `kanban`, `procrastination`, `productivity`, `local-first`, `single-file`), image de prévisualisation sociale
- [ ] `index.html` : `<meta name="description">`, balises Open Graph / Twitter Card, favicon
- [ ] PWA installable : `manifest.json` + icônes (le service worker existe déjà)
- [ ] Release GitHub taguée (`v1.2.0`…) avec notes tirées de `CHANGELOG.md`

**Validation avant diffusion**
- [ ] Test d'onboarding avec 2-3 personnes non techniques, sans explication : premier drag en < 30 s ?
- [ ] Vérification Chrome / Firefox / Safari / Brave sur le site publié (thèmes, DnD, sauvegarde, rappel d'export)
- [ ] Canal de retour : GitHub Issues activées + modèle « bug » / « idée », lien discret dans l'app (onglet Système ou aide `?`)

**Diffusion** (une à la fois, en lisant les règles d'autopromotion de chaque communauté)
- [ ] Communautés TDAH francophones (avant l'anglais : l'app l'est déjà)
- [ ] r/ADHD, r/adhdwomen, r/productivity
- [ ] Show HN (angle : fichier unique, zéro dépendance, local-first)
- [ ] AlternativeTo (alternative à Trello / Todoist / Goblin.tools)
- [ ] Article de fond (dev.to ou blog) : un logiciel développé intégralement avec des agents d'IA, pourquoi le fichier unique

**Mesure sans tracker**
- [ ] GitHub Insights (trafic, clones, stars) et Issues comme seuls indicateurs

### P0 — Indispensable pour un usage partagé

1. **Hébergement statique** — Déployer sur GitHub Pages / Netlify / Vercel. Le fichier ne fonctionne pas en `file://` (CORS bloque fetch). Sans ça, aucun non-technicien ne peut utiliser l'outil. Alternative : un script shell/batch qui lance un serveur local (`python -m http.server`). ✅ **Déployé** — https://damnscientist.github.io/KanbanJS/ (branche `main`, racine, service worker actif).
2. **Supprimer le mur de la clé API** — Demander à un utilisateur lambda de créer un compte OpenAI et coller une clé API est exactement la friction que l'outil est censé éliminer. Options : backend léger qui proxy les appels IA, intégration d'un modèle local (WebLLM / Ollama), ou mode dégradé avec templates de décomposition pré-faits (pas d'IA requise). ✅ **Templates first** — 12 corvées universelles avec matching fuzzy, grille de templates dans la modale quand l'API n'est pas configurée, FAB toujours invitant (plus de `!` rouge).
3. **Onboarding** — La liste "Tuto" est un bon début mais ne montre pas pourquoi c'est différent d'un Trello. Ajouter une première décomposition guidée ("Essayez : ranger mon bureau") qui rend le concept tangible en 30 secondes. ✅ **Onboarding interactif** — Tuto reformulé en consignes actionnables, liste "Ranger une surface" avec 6 micro-tâches d'exemple (template réel) pour un premier drag en <10s.
4. **Corriger le bug de streak (dates UTC)** — Voir Faiblesses. Fix : un helper unique `dateKey(date)` retournant `YYYY-MM-DD` **local** (formatage manuel `getFullYear`/`getMonth`/`getDate`, ou `toLocaleDateString('sv-SE')`), utilisé partout où `toISOString().slice(0, 10)` apparaît : `updateStreak`, `adjustStreak`, `refreshHeaderInfo` (compteur du jour, badge paused), et les comparaisons `doneAt.slice(0, 10) === today` des handlers de complétion/décomplétion (conversion de l'ISO en `Date` avant extraction de la clé locale, sinon `doneAt` reste en UTC et la comparaison décale). ~20 lignes. Ajouter un test dans `test.html` : une date construite à 23h30 locale doit donner la clé du jour local, pas celle du jour UTC. ✅ **Corrigé (2026-09-20)** — helpers `dateKey(date)` et `isToday(date)` dans le scope App, tous les usages remplacés ; 3 tests ajoutés (0h30, 23h30, 1er janvier). Le seul `toISOString().slice(0, 10)` restant est le nom de fichier d'export (légitime).

### P1 — Utile au quotidien

5. **Persistance robuste — le "fichier compagnon"** — localStorage est fragile (clear navigateur, changement de machine = perte totale, streak inclus). Principe retenu : **l'app apporte le mécanisme, l'utilisateur apporte le stockage** (zéro compte, zéro backend, zéro dépendance).
   - **Fichier compagnon (File System Access API)** : action « Choisir un fichier… » → `showSaveFilePicker()` ; le `FileSystemFileHandle` est conservé dans IndexedDB (survit aux redémarrages). `DB.onSave()` déclenche un flush debouncé (800 ms). Même format que `exportJSON()` (`{state, streak}`) — zéro nouveau format. ✅ **Implémenté (2026-09-20)** — module `AutoSave`, onglet Données, clé API exclue du fichier.
   - **Restauration en un clic** : au boot, si `kanbanjs:state` est vide/corrompu mais que le handle existe et que le fichier contient des données → bannière inline "Vos données ont disparu — Restaurer ?" → un seul clic → `_rebuild()`. Le public cible ne complètera jamais un flow de restauration en 5 étapes. ✅ **Implémenté (2026-09-20)** — bandeau `restore`, déclenché par `DB.loadSource() === 'default'` + handle présent.
   - **Multi-machine gratuit** : si l'utilisateur range le fichier dans un dossier synchronisé (Drive, Dropbox, Syncthing), il obtient la sync sans une ligne de code côté app. ✅ **Implémenté (2026-09-20, avec garde-fou)** — bandeau `reload` si `file.lastModified > lastWriteAt`. Coût : la permission Chromium ne dure qu'une session, donc **un clic « 💾 Réactiver » par session** ; c'est ce clic qui permet de détecter la version externe.
   - **`navigator.storage.persist()`** au premier `updateStreak()` — empêche l'éviction implicite par le navigateur. Ne protège pas du "effacer les données de navigation", d'où le fichier compagnon. ✅ **Implémenté** — `requestPersist()` (flag `_persistAsked`, vérifie `persisted()` avant de demander).
   - **Sync multi-onglets** (absorbé depuis P2) : `window.addEventListener('storage', …)` → debounce → `_rebuild()`. ~15 lignes. Prérequis si le companion voit le jour (la spec `companion.md` prévoit aujourd'hui d'"accepter le race"). ✅ **Implémenté** — `_onStorage()` filtre les clés (`state`, `streak`, `clear`), debounce 200 ms, `DB.reload()` immédiat puis `_rebuild()` différé pendant une édition ; piles undo purgées ; les conflits simultanés restent en dernier écrivain gagne.
   - **Self-heal** : dans `DB._load()`, si le JSON est corrompu, tenter une copie de secours avant de retomber sur `_default` — perdre son board pour un JSON cassé est insupportable. ✅ **Implémenté** — `kanbanjs:state.bak` (état précédent, écrit par `_save()`), restauré et réinjecté dans la clé principale par `_load()`.
   - **Replis et pièges** — permission à réactiver à chaque session Chromium ✅ **Implémenté** (puce `💾 Réactiver` dans le header, `resume()`, jamais d'`alert()`) ; conflits multi-machines = dernier écrivain gagne ✅ (documenté dans le README, bandeau `reload` au lieu d'un écrasement silencieux) ; repli Safari/Firefox/Brave ✅ **Implémenté (2026-09-20)** — sonde de capacité + message ciblé + export manuel avec rappel persistant à 7 jours ; détection d'un fichier modifié pendant que l'app est ouverte ✅ **Implémentée (2026-09-20)** — `AutoSave.checkExternal()` toutes les 60 s (suspendue quand l'onglet est caché) et au retour de visibilité, via `getFile().lastModified > lastWriteAt`. Aucun trigger pendant un flush ni si une décision est déjà en attente.
   - **Ne PAS faire** : WebDAV, GitHub Gist, backend proxy, compte utilisateur — chacun brise la philosophie zéro-compte sans mieux protéger que le fichier chez l'utilisateur.
   - **Estimation** : ~175 lignes au total — ✅ ~25 livrées le 2026-09-20 (persist + storage event + self-heal), ✅ ~230 livrées le 2026-09-20 au titre du fichier compagnon (le budget initial de ~150 était sous-estimé : ni l'UI, ni les permissions, ni les tests n'étaient comptés), ✅ ~120 livrées le 2026-09-20 au titre des replis navigateurs (sonde de capacité, messages ciblés, export manuel + rappel).
6. **Feedback de progression** — La barre de progression existe mais il n'y a pas de gratification quand on termine une tâche. ✅ **Streak + pulse + compteur du jour** implémentés.
7. **UX mobile / Companion** — ❌ **Abandonné (2026-09-20)**. La spec `companion.md` décrit un entonnoir minimal mobile (champ texte → liste Inbox), mais la décision est prise : le mobile n'est pas le lieu de l'exécution, KanbanJS reste une expérience desktop, et le besoin de capture rapide depuis le téléphone ne s'est pas fait sentir. `companion.md` est conservé comme trace de la réflexion, mais `companion.html` ne sera pas implémenté. Aucun autre chantier n'en dépend.

### P2 — Nice to have

8. **Notes markdown** — ✅ **Implémenté** — cycle édition/preview/fermé, rendu complet (gras, italique, code, liens, titres, listes, ligne horizontale).
9. **Tags** — ✅ **Implémenté** — chips par hash couleur, bouton `+ tag`, raccourci `t`, préservés au clipboard, recherchables.
10. **Mode offline complet** — ✅ **Implémenté** — service worker cache-first (`sw.js`), la page fonctionne hors-ligne après la première visite.
11. **Dates d'échéance** — ⏸ **En attente** (voir Décisions non retenues). Champ `dueDate` sur les cartes avec indication visuelle (couleur selon proximité). Utile pour les freelances avec deadlines.
12. **Notifications navigateur** — ⏸ **En attente**. `Notification API` : rappel quotidien du streak ou notification quand une carte arrive dans Done. Renforce la boucle motivationnelle hors de l'onglet.
13. **Son au clic de complétion** — ➡ **Planifié** (prochains chantiers, #5). Feedback auditif optionnel (toggle), renforce le sentiment d'accomplissement pour le public TDAH.
14. **Réorganisation clavier** — Alt+J/K pour déplacer une carte vers le haut/bas sans souris.
15. **Dashboard statistiques** — ⏸ **En attente**. Vues semaine/mois (cartes terminées, répartition par liste, évolution streak).
16. **Import/export CSV et Markdown** — ⏸ **En attente**. Interopérabilité avec d'autres outils.
17. **Fournisseur LM Studio** — ➡ **Planifié** (prochains chantiers, #6). Ajouter `lmstudio` au switch `queryAI`, sur le modèle de `ollama` : serveur local compatible OpenAI (`http://localhost:1234/v1/chat/completions` par défaut), clé API optionnelle (LM Studio n'en exige pas), timeout long (~5 min) car les modèles locaux sont lents. Parsing identique à OpenAI. Complète l'offre « IA locale sans compte » à côté d'Ollama — utile pour les utilisateurs qui préfèrent l'interface graphique de LM Studio pour gérer/télécharger leurs modèles.

## Changelog

L'historique complet des versions est dans [`CHANGELOG.md`](CHANGELOG.md). **Toute évolution doit y être consignée** (entrée datée, version, tests, bump du service worker) — c'est le seul endroit où l'historique est tenu.
