# KanbanJS — Projet anti-procrastination

Fichier unique `kanban.html` (~3100 lignes). Aucune dépendance, pas de bundler. S'ouvre dans un navigateur moderne.

## Concept

Transformer une tâche lourde en micro-actions à coût cognitif nul via IA. L'utilisateur décrit une corvée → un agent d'IA la décompose en actions minuscules (30s-3min), créées dans une liste "Suggestions" dédiée. L'utilisateur les glisse ensuite dans "In Progress" → "Done".

## Fonctionnalités existantes

### Board
- Board unique avec listes et cartes
- Renommage du board (input dans le header, `maxlength="200"`)
- Header info : nombre de listes et cartes
- Thème jour/nuit persisté dans 3 clés (`kanbanjs:theme-mode`, `kanbanjs:theme-dark`, `kanbanjs:theme-light`)
- Sélecteur de thème dans la modale Configuration (onglet "Thèmes") : 4 sombres, 4 clairs
- Bouton reset (purge `kanbanjs:state`, rechargement)
- Export / Import JSON du board (boutons ↓ ↑ dans le header)
- Indicateur de progression : barre + pourcentage dans le header (`3/8`, `38%`)

### Clavier
- Raccourcis vim-like : minuscule = carte, majuscule = liste
- Navigation : `h`/`l` cartes ↑/↓, `j`/`k` listes ←/→, `1-9` focus liste n
- Actions carte : `n` créer, `r` renommer, `e` ouvrir notes, `x` couper, `y` copier, `p` coller
- Actions liste : `N` créer, `E` renommer, `D` toggle terminé, `X` couper, `Y` copier, `P` coller
- `Esc` désélectionne la carte/liste courante, `?` affiche l'aide clavier
- Presse-papier unifié : couper/copier une liste copie titre + cartes
- Sélection visuelle : bordure accent, actions toujours visibles sur carte sélectionnée
- Ignore automatiquement quand un input/textarea a le focus ou une modale est ouverte
- `Ctrl+K` : recherche fuzzy sur les cartes du board (texte, notes, titre de liste)
- `u` undo, `Ctrl+Y` redo : 30 niveaux de snapshots (session uniquement)

### Listes
- Création inline (bouton + formulaire "Ajouter une liste")
- Renommage (clic sur le titre)
- Suppression (confirmation)
- Défaut au premier lancement : "Tuto", "Backlog", "In Progress", "Waiting", "Done"
- Drag & drop souris et tactile pour réordonner horizontalement (DnD générique)
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
- **Notes** : chaque carte a un bouton `≡` qui ouvre une textarea de note personnelle, sauvegardée au blur
- Suppression
- Drag & drop souris et tactile entre listes (ou au sein de la même liste)
- Drag ghost + placeholder visuel

### IA (Décomposition)
- FAB flottant `＋` en bas à droite (passe en `!` rouge si l'API n'est pas configurée)
- Modal "Décomposer une tâche" avec textarea
- **Abstraction fournisseur** : supporte OpenAI/compatible, Anthropic, Google Gemini (switch dans `queryAI`)
- Appel POST à l'API avec format spécifique par fournisseur
- Prompt système : décomposition en micro-actions anti-procrastination
- Parsing robuste de la réponse JSON (équilibrage des crochets)
- Résultat : liste "Suggestions" créée en position 0 avec les cartes générées
- Panneau de debug toggleable (logs requête/réponse/parsing)

### Persistance
- `localStorage` avec 3 clés :
  - `kanbanjs:state` → board, listes, cartes (reset nettoie uniquement celle-ci)
  - `kanbanjs:config` → provider, endpoint, apiKey, model
  - Thème (3 clés) :
    - `kanbanjs:theme-mode` → 'dark' | 'light' | 'system'
    - `kanbanjs:theme-dark` → 'warm-night' | 'deep-ocean' | 'forest' | 'tokyo-night'
    - `kanbanjs:theme-light` → 'soft-sand' | 'mint' | 'lavender' | 'tokyo-light'

## Architecture

### Modules (dans l'ordre dans le fichier)

| Module | Responsabilité | API publique |
|---|---|---|
| `initTheme` | IIFE, lit/applique/persiste le thème | lecture au load |
| `DB` | Adapter localStorage | `getBoard, renameBoard, getLists, createList, renameList, deleteList, reorderList, toggleListDone, toggleCollapsed, getCards, getAllCards, createCard, updateCard, updateCardNotes, setCardDoneAt, setCardsDoneAt, deleteCard, moveCard, exportJSON, importJSON, reset, restoreJSON, batch` |
| `DnD` | Moteur de drag & drop générique | `start(dragEl, id, { ghostEl?, ghostClass?, phClass?, getZone, getAfter, getPos, skip? }, onDrop)` |
| `App` | UI Kanban | `init()` boot la session |

### Mécanique DnD
- `pointerdown`/`pointermove`/`pointerup` — API unifiée souris + tactile
- Seuil 4px avant démarrage (évite les faux positifs)
- Ghost : clone de l'élément, fixed, `pointer-events:none`
- `getZone(x,y)` : itère sur `getBoundingClientRect()` des cibles (pas de `elementFromPoint`)
- Guard `lastPos` : mutation DOM uniquement si la position a changé
- `cleanup()` après `dropCb()` (capture locale de la callback)

## Design
- Dark/light mode via `data-theme` sur `<html>` + CSS custom properties
- Modals : overlay `z-index: 20000`
- FAB : `z-index: 5000`, `position: fixed` bottom-right
- Couleurs : palette dark (bg `#1a1410`) et light (bg `#f7f3ee`)

## Limitations / À faire

### À implémenter (priorité décroissante)
1. *(aucune — tout est fait)*

### Nice to have (si projet < 3000 lignes)
4. **Notes markdown** — rendu basique (gras, italique, listes, code inline) dans la textarea de notes, toggle édition/aperçu (70-110 LOC).
5. **Tags** — champ `tags: []` sur les cartes, chips colorés, filtrable via la recherche (70-85 LOC).

### Non retenu
- **Multi-boards** : sur-ingénierie pour un outil mono-utilisateur. Les listes séparent déjà les contextes.

### Limites techniques
- **Pas de déploiement** — fichier local, provoque des erreurs CORS si ouvert en `file://` (l'API fetch y est bloquée, à servir via un serveur local)
- **Fournisseur IA centralisé** : tout le branchement fournisseur est dans `queryAI` (switch provider → body/headers/parsing)

## Corrections récentes (audit 2026-06-21 — bugs finaux)

- **Bug 1 — `getAfter` cartes recevait `x` au lieu de `y`** : le moteur DnD appelle `cfg.getAfter(container, x, y, ghostEl)` mais le callback carte déclarait `(container, y)`, donc `y` recevait la valeur de `x`. Le placement vertical des cartes pendant le drag était calculé à partir de la coordonnée horizontale. Corrigé en `(container, x, y)`.
- **Bug 2 — Cache `_cardCache` jamais invalidé entre listes** : le cache des rects des cartes était construit pour une liste et réutilisé tel quel quand le drag survolait une autre liste. Les cartes atterrissaient toujours en fin de liste cible. Corrigé en keyant le cache par container (`_cardCache._container`).
- **Bug 3 — Double snapshot undo sur « Sauver »** : le blur de l'éditeur déclenchait `saveEdit()` puis le click sur le bouton la redéclenchait, créant un snapshot undo fantôme. Corrigé avec un guard `if (!wrap.classList.contains('editing')) return;` en tête de `saveEdit`.
- **Bug 4 — « Annuler » sauvegardait quand même** : cliquer « Annuler » appelait `exitEdit()` (retire `.editing`), mais le blur subséquent appelait `saveEdit()` qui persistait la modification. Le guard du Bug 3 corrige aussi ce bug.
- **Bug 7 — `importFile.value = ''` hors callback** : le reset de l'input file était exécuté de manière synchrone après `reader.readAsText()`, avant que `onload` ne se déclenche. Déplacé dans un `finally` du `onload` et dans `onerror`.
- **Bug 8 — `_selList` orphelin après suppression de liste** : le handler de suppression appelait `selectCard(null)` mais ne désélectionnait pas `_selList`, qui pointait vers un élément DOM retiré. Remplacé par `clearSelection()`.
- **Bug 12 — Recherche sélectionnait carte dans liste repliée** : cliquer sur un résultat de recherche dans une liste collapsed sélectionnait la carte mais ne la rendait pas visible. Ajout d'un dépliage automatique de la liste avant la sélection.

## Corrections récentes (audit 2026-06-21 — visuel)

- **Point 1 — Couleurs hardcodées** : tous les `rgba(232,168,90,…)` / `#e8a85a` dans les règles CSS (`.list.drag-over`, `.btn-icon.active`, `.drop-placeholder`, `.list-placeholder`, `.card.card-selected`, `.list.list-selected`, `.add-list-btn:hover`) remplacés par `color-mix(in srgb, var(--accent) X%, transparent)`.
- **Point 2 — `.add-list-btn` thèmes clairs** : suppression du `background: rgba(255,255,255,.03)` de base et des 16 lignes de surcharges par thème (4 thèmes × 2 règles). Remplacé par un `:hover` unique avec `color-mix(in srgb, var(--accent) 10%, transparent)`.
- **Point 3 — `.btn-icon.danger:hover`** : `rgba(217,90,74,.15)` → `color-mix(in srgb, var(--danger) 15%, transparent)`.
- **Point 4 — Debug panel** : couleurs hardcodées (`#0a0a0f` / `#a0e0a0` / `#88aacc` / `#e06060`) remplacées par les variables de thème (`--surface-2`, `--text`, `--accent`, `--danger`) + bordure `var(--border)`.
- **Point 5 — Ghost DnD** : `box-shadow: 0 8px 32px rgba(0,0,0,.6)` → `box-shadow: var(--shadow)` (s'adapte au thème).
- **Point 6 — `.cfg-status.success`** : `color: #2ecc71` → `color: var(--accent)` (chaque thème a sa propre couleur de succès).
- **Point 7 — Surcharges `.add-list-btn`** : voir Point 2.
- **Point 8 — Responsive** : ajout de `.cheatsheet-intro { display: none }` sur mobile (cohérent avec `.cheatsheet-shortcuts`). `--list-width` réduit à 260px sur mobile.
- **Point 9 — États visuels** : ajout de `:focus-visible` global (outline `var(--accent)`), `:active` sur `.btn-primary` et `.btn-ghost`, `:disabled` sur `.btn` / `.btn-icon` (opacity + `cursor: not-allowed`), `cursor: default` sur `.cheatsheet .row`.
- **Point 10 — Fallback `color-mix()`** : ajout des variables `--accent-rgb` / `--danger-rgb` dans `:root` et les 8 thèmes. Fallback `rgba(var(--accent-rgb), .XX)` avant chaque `color-mix()` pour les WIP warnings (`.list.wip-warn`, `.list-count.wip-warn`).

## Corrections récentes (audit 2026-06-21 — architecture)

- **Point 1 — Frontières DB** : ajout des méthodes `DB.exportJSON()`, `DB.importJSON(json)`, `DB.reset()`, `DB.restoreJSON(json)`, `DB.getAllCards()`. Plus aucun accès à `localStorage` / `DB.KEY` hors du module DB. L'import passe par `DB.importJSON()` avec validation + gestion QuotaExceededError.
- **Point 2 — DOM source de vérité** : `refreshHeaderInfo()` et `refreshCountBadge()` sont async et lisent désormais les données via `DB.getLists()`, `DB.getAllCards()`, `DB.getCards(id)` au lieu de `querySelectorAll('.card')`.
- **Point 3 — Init parallèle** : `init()` utilise `DB.getAllCards()` + `lists.forEach(l => buildList(l, cards))` au lieu d'un `for...of await`. Une seule lecture DB au lieu de N.
- **Point 4 — Uniformisation sync** : `buildList(list, cards)` est maintenant synchrone. Les cartes sont pré-fetchées par l'appelant (`DB.getCards()` ou `DB.getAllCards()`).
- **Point 5 — Presse-papier encapsulé** : `Clipboard` remplace la variable globale `_clipboard`. Méthodes : `copyCard/cutCard/copyList/cutList/paste/clear/isEmpty/type`.
- **Point 6 — mutate helper** : `mutate(fn)` fait `_snapshot()` puis exécute `fn()`. Remplace les ~15 `_snapshot(); await DB.xxx()`. Le collapse toggle bénéficie maintenant d'un undo. `_snapshot()` utilise `DB.exportJSON()` au lieu de `localStorage.getItem(DB.KEY)`.
- **Point 7 — Theme API** : `const Theme = window.__themeAPI` capturé au début de App. La modale configuration utilise `Theme` au lieu de `window.__themeAPI`.
- **Point 8 — Helper DOM** : `removeListDom(listId)` extrait la suppression DOM répétée. Utilisé dans `createSuggestionsList`.
- Undo/Redo utilisent `DB.exportJSON()` / `DB.restoreJSON()` au lieu de `localStorage` direct.
- Reset et bouton de récupération utilisent `DB.reset()` au lieu de `localStorage.removeItem(DB.KEY)`.

## Corrections récentes (audit 2026-06-21 — bugs)

- **Bug 1 — Guard DOM null** : ajout de guards `if (!el) return;` / optional chaining sur les `querySelector` dans `cutCard`, `copyCard`, `cutList`, `copyList` et la boucle de recherche. Évite des TypeError si le DOM est modifié concurrentiellement.
- **Bug 2 — Undo null** : déjà corrigé (le code utilise `DB.exportJSON()` et non `localStorage.getItem(DB.KEY)` depuis l'audit architecture).
- **Bug 3 — Double _save() toggle done** : `DB.toggleListDone(id, doneAt)` accepte maintenant un second paramètre optionnel pour setter `doneAt` sur toutes les cartes en une seule écriture. Le handler passe `now` ou `null`, supprimant l'appel redondant à `DB.setCardsDoneAt()`.
- **Bug 4 — FileReader onerror** : ajout de `reader.onerror` avec `alert('Erreur de lecture du fichier.')` sur l'import JSON.
- **Bug 5 — Ctrl+R dans AGENTS.md** : corrigé en `Ctrl+Y` (convention standard, `Ctrl+R` est le rechargement du navigateur).
- **Bug 6 — Collapse undo** : déjà corrigé (le handler utilise `mutate()` depuis l'audit architecture).
- **Bug 7 — Centralisation Escape** : le handler Escape dédié de la cheatsheet est supprimé. Le BINDINGS `Escape` gère maintenant les deux cas : fermer la cheatsheet si ouverte, sinon désélectionner.
- **Bug 8 — _selCard orphelin** : le handler de suppression de liste (`delListBtn`) appelle `selectCard(null)` si la carte sélectionnée appartient à la liste supprimée.
- **Bug 9 — Ordre DnD DOM/DB** : `wrap.dataset.listId = toListId` déplacé après `await mutate(() => DB.moveCard(...))` pour éviter une désynchronisation DOM/DB si la DB échoue.

## Corrections récentes (audit 2026-06-20)
- Raccourcis clavier vim-like : minuscule = carte, majuscule = liste. Navigation `h`/`l`/`j`/`k` + actions `n`/`r`/`e`/`x`/`y`/`p` + `N`/`E`/`D`/`X`/`Y`/`P` + `1-9` + `Esc` + `?` aide (~160 LOC)
- Sélection visuelle carte/liste : bordure accent, actions visibles en permanence sur carte sélectionnée
- Presse-papier unifié : `x`/`y`/`p` sur cartes, `X`/`Y`/`P` sur listes (titre + cartes)
- Escape ferme la textarea de notes (cohérent avec l'éditeur de carte)
- Cheatsheet toggleable avec `?` (overlay semi-transparent, deux colonnes carte/liste)
- `L` → `N` (nouvelle liste), `R` → `E` (renommer liste) : cohérence majuscule = liste
- `Ctrl+K` : recherche fuzzy sur les cartes du board (texte, notes, titre de liste)
- Indicateur de progression : barre + pourcentage dans le header (`3/8 cartes`, `38%`)
- WIP warning : bordure et badge en `var(--accent)` quand une liste "WIP" / "In Progress" contient ≥ 3 cartes
- Cheatsheet enrichie : 5 principes kanban anti-procrastination en style kbd, section philosophique masquée sur mobile, raccourcis masqués sur mobile, lien `?` dans le header
- Header mobile : `overflow-x: auto` (les boutons restent accessibles, la barre de progression est visible)
- Bugfixes : bleed-through clavier sur overlay recherche, Escape cheatsheet vidait la sélection, éditeur carte bloqué si texte vide, import quota, _selCard orphelin après suppression UI, contextmenu DnD non retiré
- Undo/Redo : `u` undo, `Ctrl+Y` redo, 30 snapshots (session uniquement)
- Repli des listes : bouton ▾/▸ pour masquer cartes + footer, état persisté
- Liste "Waiting" ajoutée par défaut (kanban canonique : Backlog → In Progress → Waiting → Done)

## Corrections récentes (audit 2026-06-19)
- `DB.setCardsDoneAt(listId, doneAt)` : méthode batch pour éviter N écritures localStorage quand on toggle une liste terminée
- `_save()` wrappé dans try/catch avec alerte en cas de `QuotaExceededError` (stockage saturé)
- `parseAITasks` : détection des guillemets dans le compteur de crochets (les `[`/`]` dans les chaînes JSON ne perturbent plus le parsing)
- Prompt OpenAI : ajout d'un message `role: 'user'` en complément du `system` (compatibilité élargie avec les modèles exigeant un message user)
- `flashMessage(el, msg, cls, ms)` : fonction partagée remplaçant les doublons `showStatus`/`show`
- `close()` de la modale décomposition nettoie le timer de statut
- `maxlength="200"` sur les inputs de nom (board et listes)

## Corrections récentes (audit 2026-06-17)
- Timeout 60s sur l'appel fetch API (`AbortController`)
- `init()` wrappé dans try/catch avec bouton de réinitialisation en cas de crash
- `createSuggestionsList` : écritures DB avant DOM (réduit les incohérences si échec partiel)
- Chaînes vides filtrées dans `parseAITasks`
- `readAIConfig()` unifiée (suppression du doublon `loadCfg`)
- Clé localStorage exposée via `DB.KEY` (plus de littéral dupliqué)
- Blur sur l'éditeur de carte → sauvegarde automatique
- `esc()` remontée au niveau App, cartes créées en parallèle (`Promise.all`)
- Liste "terminé" : bouton `✓` dans chaque liste, cartes barrées/grissées, mis à jour au drag & drop, timestamp de complétion affiché
- Sélecteur de thème : 4 sombres (Warm Night, Deep Ocean, Forest, Tokyo Night) et 4 clairs (Soft Sand, Mint, Lavender, Tokyo Light) via onglet dans la modale Configuration
- `initTheme` refactoré avec 3 clés (mode / dark / light) et mode système (`prefers-color-scheme`)
- Abstraction fournisseur IA : `queryAI` branche via switch (`'openai'` / `'anthropic'` / `'google'`) pour body, headers et parsing de réponse. Endpoint optionnel pour Anthropic, ignoré pour Google.
- DnD tactile : migration `mousedown/mousemove/mouseup` → `pointerdown/pointermove/pointerup` + `setPointerCapture` + `touch-action: none`
- Responsive : media query ≤640px, header compact, boutons tactiles, modales scrollables
- Export / Import JSON du board
- FAB : badge rouge `!` quand l'API n'est pas configurée, redevient `+` après sauvegarde de la config
- Timestamp de complétion : `doneAt` stocké quand une carte glisse dans une liste "terminée", affiché sous le texte
- Tuto : liste "Tuto" avec cartes-exemples dans `DB._default` au lieu d'une carte flottante
- DnD : handler `pointercancel` + `cleanup()` défensif (libération capture persistante)

## Corrections récentes (audit 2026-06-21 — qualité/DRY)

- **Point 1 — `toggleForm(btn, form, input, open)`** : helper réutilisable pour le pattern open/close de formulaire. Remplace `openCardForm`/`closeCardForm` dans `buildList` et `openForm`/`closeForm` dans `buildAddListWidget`.
- **Point 2 — `registerOverlay(overlay, closeFn)`** : helper qui enregistre les handlers click (fermeture au clic extérieur) et keydown (Escape) pour une modale/overlay. Utilisé par Config modal, Decompose modal, Cheatsheet, Search.
- **Point 3 — `formatDoneAt(iso)`** : formatage de date unifié (3 occurrences → 1 fonction). Format : "Terminé le JJ/MM/AAAA à HH:MM".
- **Point 4 — `extractCardData(el)` / `extractListData(el)`** : extraction des données carte/liste depuis le DOM. Utilisées par `cutCard`, `copyCard`, `cutList`, `copyList`.
- **Point 5 — Uniformisation `mutate()`** : tous les appels `_snapshot()` + DB mutante passent désormais par `mutate(fn)`. Suppression de 7 `_snapshot()` raw (submitCard, doneToggle, submitList, _paste ×2, createSuggestionsList).
- **Points 6-7-8-9-10** : non appliqués. Le CSS a déjà été traité lors de l'audit visuel. La délégation d'événements (point 9) et le rebuild board (point 8) apporteraient un risque de régression disproportionné pour le gain. `esc()` est légitime tel quel (sécurité XSS).

## Corrections récentes (audit 2026-06-21 — perf/robustesse)

- **Point 1 — Cache `getBoundingClientRect()` DnD** : les rects des listes et des cartes sont capturés une fois au début du drag (dans `_zoneCache` / `_cardCache`) et réutilisés pendant tout le déplacement. Évite les reflows forcés à chaque `pointermove`. Caches réinitialisés dans le callback `onDrop`.
- **Point 3 — `DB.batch(fn)`** : nouveau mécanisme de batching. Suspend `_save()` (flag `_saving`), exécute `fn`, puis `_save()` une seule fois. Utilisé dans `createSuggestionsList()` (1 liste + 35 cartes = 1 écriture au lieu de 37) et dans `_paste()` pour les listes.
- **Point 5 — Ghost DnD liste simplifié** : au lieu de `wrap.cloneNode(true)` (clone du DOM complet de la liste), un `ghostBuilder` crée un élément minimal (header sans boutons). L'option `ghostBuilder` est supportée par le moteur DnD.
- **Point 8 — Import quota pre-check** : avant d'importer, estimation de la taille via `new Blob([jsonString]).size`. Si > 4.5 MB, avertissement avec confirmation avant l'écriture.
- **Point 9 — `AbortController` cleanup** : le `clearTimeout(timer)` est maintenant dans un bloc `finally` autour du `fetch`, garantissant le nettoyage même en cas d'erreur réseau.
- **Point 10 — Taille snapshot limitée** : `_snapshot()` ignore les snapshots > 200 KB (évite l'épuisement du `sessionStorage` avec 30 snapshots volumineux).
- **Bugfix — `await` dans `forEach`** : le `lists.forEach()` dans `init()` utilisait un `await` dans un callback non-async (erreur de syntaxe introduite dans l'audit 01). Remplacé par `for...of`.
- **Bugfix — Double `DB.exportJSON()`** : `_snapshot()` appelait `DB.exportJSON()` deux fois (une pour la vérification de taille, une pour le push). Corrigé avec une seule variable `json`.

## Pour reprendre le développement

- Les helpers `toggleForm`, `registerOverlay`, `formatDoneAt`, `extractCardData`, `extractListData` sont dans le scope App, juste après `removeListDom`.
- `mutate(fn)` est le seul point d'entrée pour les snapshots undo. Ne jamais appeler `_snapshot()` directement — utiliser `await mutate(() => DB.xxx(...))`.
- `DB.batch(fn)` suspend les écritures `localStorage` le temps de l'exécution, puis persiste en une fois. Utiliser pour les opérations groupées (création multiple de cartes, import de liste avec cartes).

- L'ordre des modales et du debug suit le flow : config → décompose
- Le prompt système est dans `PROMPT_SYSTEM` (template literal, substitution `{{TASK}}`)
- La config API (provider, endpoint, apiKey, model) est stockée en localStorage, lue par `readAIConfig()` dans App
- Le fournisseur est branché dans `queryAI` via un switch (`'openai'` / `'anthropic'` / `'google'`), chaque branche construit body + headers + parsing de réponse
- Le board est initialisé avec 5 listes par défaut via `DB._default` (dont "Tuto" avec cartes-exemples)
- `buildList(list, cards)` est synchrone — les cartes sont passées en paramètre (pré-fetchées par l'appelant)
- `buildCard(card, done)` est synchrone (reçoit une carte déjà construite)
- Les deux fonctions `buildList` et `buildCard` sont maintenant synchrones
- Les listes ont un champ `done` (booléen) ; si vrai, leurs cartes affichent `.card-done` (barré + grisé)
- `buildCard(card, done)` accepte un second paramètre pour le style initial
- Les cartes ont un champ `notes` (chaîne) persisté via `updateCardNotes`, éditable inline (textarea toggleable)
- Les cartes ont un champ `doneAt` (ISO string ou null) stocké automatiquement quand glissées dans une liste "terminée"
- `refreshHeaderInfo()` et `refreshCountBadge(id)` sont async, lisent les données via DB (pas le DOM)
- `Clipboard` est un objet (plus un `let _clipboard`), méthodes : `copyCard/cutCard/copyList/cutList/paste/clear/isEmpty/type`
- Le thème est capturé au démarrage de App via `const Theme = window.__themeAPI`, plus aucun accès à `window.__themeAPI` dans le code métier
- `removeListDom(listId)` est une fonction partagée pour retirer une liste du DOM par ID

## Convention de code
- `make(tag, className)` pour créer des éléments
- `autoResize(ta)` pour les textareas
- `flashMessage(el, msg, cls, ms)` pour les messages de statut temporaires (stocke le timer sur `el._timer`)
- Tableaux de bord en `const`, fonctions helpers en closures
- Pas de commentaires dans le code (sauf en-têtes de sections)
- Noms en français (utilisateur francophone)
