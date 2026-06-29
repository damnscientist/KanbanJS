# KanbanJS — Projet anti-procrastination

Fichier unique `kanban.html` (~3200 lignes). Aucune dépendance, pas de bundler. S'ouvre dans un navigateur moderne.

## Concept

Transformer une tâche lourde en micro-actions à coût cognitif nul via IA. L'utilisateur décrit une corvée → un agent d'IA la décompose en actions minuscules (30s-3min), créées dans une liste "Suggestions" dédiée. L'utilisateur les glisse ensuite dans "In Progress" → "Done".

## Positionnement

Le public naturel est les gens avec TDAH, les étudiants qui procrastinent, les freelances submergés. L'angle "fichier unique, pas de compte, données locales" est un argument fort face aux Trello/Todoist. Le projet est fonctionnellement complet pour un usage personnel. Pour le rendre accessible à d'autres, le travail restant est surtout la distribution (P0 de la roadmap) et l'UX d'onboarding (P1), pas du code.

## Fonctionnalités existantes

### Board
- Board unique avec listes et cartes
- Renommage du board (input dans le header, `maxlength="200"`)
- Header info : nombre de listes et cartes
- Thème jour/nuit persisté dans 3 clés (`kanbanjs:theme-mode`, `kanbanjs:theme-dark`, `kanbanjs:theme-light`)
- Sélecteur de thème dans la modale Configuration (onglet "Thèmes") : 4 sombres, 4 clairs
- Bouton reset (purge `kanbanjs:state`, reconstruction DOM)
- Export / Import JSON du board (boutons ↓ ↑ dans le header)
- Indicateur de progression : barre + pourcentage dans le header (`3/8`, `38%`)

### Clavier
- Raccourcis vim-like : minuscule = carte, majuscule = liste
- Navigation : `h`/`l` listes ←/→, `j`/`k` cartes ↓/↑, `1-9` focus liste n
- Actions carte : `n` créer, `r` renommer, `e` ouvrir notes, `x` couper, `y` copier, `p` coller
- Actions liste : `N` créer, `E` renommer, `D` toggle terminé, `X` couper, `Y` copier, `P` coller
- `Esc` désélectionne la carte/liste courante ou ferme l'overlay actif, `?` affiche l'aide clavier
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
- Parsing robuste de la réponse JSON (équilibrage des crochets, détection des guillemets)
- Résultat : liste "Suggestions" créée en position 0 avec les cartes générées
- Panneau de debug toggleable (logs requête/réponse/parsing)
- **Templates** : 12 templates de corvées universelles intégrés, matching fuzzy sur le texte saisi dans le textarea (💡 "Template suggéré"). Zéro API requise.

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
- Ghost : clone de l'élément ou `ghostBuilder` optionnel, fixed, `pointer-events:none`
- `getZone(x,y)` : itère sur `getBoundingClientRect()` des cibles (pas de `elementFromPoint`)
- Guard `lastPos` : mutation DOM uniquement si la position a changé
- Cache des `getBoundingClientRect()` construit au démarrage du drag, évite les reflows à chaque `pointermove`
- `cleanup()` après `dropCb()` (capture locale de la callback)
- Handler `pointercancel` + `cleanup()` défensif (libération capture persistante)

### Design
- Dark/light mode via `data-theme` sur `<html>` + CSS custom properties
- Modals : overlay `z-index: 20000`
- FAB : `z-index: 5000`, `position: fixed` bottom-right
- 4 thèmes sombres (Warm Night, Deep Ocean, Forest, Tokyo Night) et 4 clairs (Soft Sand, Mint, Lavender, Tokyo Light)
- Couleurs CSS : `color-mix(in srgb, var(--accent) X%, transparent)` + fallback `rgba(var(--accent-rgb), .X)` pour compatibilité

### Limites techniques
- **Pas de déploiement** — fichier local, provoque des erreurs CORS si ouvert en `file://` (l'API fetch y est bloquée, à servir via un serveur local)
- **Fournisseur IA centralisé** : tout le branchement fournisseur est dans `queryAI` (switch provider → body/headers/parsing)

### Décisions non retenues
- **Multi-boards** : sur-ingénierie pour un outil mono-utilisateur. Les listes séparent déjà les contextes.

## Pour reprendre le développement

### Conventions et points d'entrée
- `mutate(fn)` est le seul point d'entrée pour les snapshots undo. Ne jamais appeler `_snapshot()` directement — utiliser `await mutate(() => DB.xxx(...))`.
- `DB.batch(fn)` suspend les écritures `localStorage` le temps de l'exécution, puis persiste en une fois. Utiliser pour les opérations groupées (création multiple de cartes, import de liste avec cartes).
- Les helpers `toggleForm`, `registerOverlay`, `formatDoneAt`, `extractCardData`, `extractListData`, `removeListDom` sont dans le scope App, juste après `removeListDom`.
- Le thème est capturé au démarrage de App via `const Theme = window.__themeAPI`. Plus aucun accès à `window.__themeAPI` dans le code métier.
- `Clipboard` est un objet (plus un `let _clipboard`), méthodes : `copyCard/cutCard/copyList/cutList/paste/clear/isEmpty/type`.

### Architecture du code
- L'ordre des modales et du debug suit le flow : config → décompose
- Le prompt système est dans `PROMPT_SYSTEM` (template literal, substitution `{{TASK}}`)
- La config API (provider, endpoint, apiKey, model) est stockée en localStorage, lue par `readAIConfig()` dans App
- Le fournisseur est branché dans `queryAI` via un switch (`'openai'` / `'anthropic'` / `'google'`), chaque branche construit body + headers + parsing de réponse
- Le board est initialisé avec 5 listes par défaut via `DB._default` (dont "Tuto" avec cartes-exemples)
- `buildList(list, cards)` est synchrone — les cartes sont passées en paramètre (pré-fetchées par l'appelant)
- `buildCard(card, done)` est synchrone (reçoit une carte déjà construite)
- Les listes ont un champ `done` (booléen) ; si vrai, leurs cartes affichent `.card-done` (barré + grisé)
- Les cartes ont un champ `notes` (chaîne) persisté via `updateCardNotes`, éditable inline (textarea toggleable)
- Les cartes ont un champ `doneAt` (ISO string ou null) stocké automatiquement quand glissées dans une liste "terminée"
- `refreshHeaderInfo()` et `refreshCountBadge(id)` sont async, lisent les données via DB (pas le DOM)

### Convention de code
- `make(tag, className)` pour créer des éléments
- `autoResize(ta)` pour les textareas
- `flashMessage(el, msg, cls, ms)` pour les messages de statut temporaires (stocke le timer sur `el._timer`)
- Tableaux de bord en `const`, fonctions helpers en closures
- Pas de commentaires dans le code (sauf en-têtes de sections)
- Noms en français (utilisateur francophone)

## Roadmap

### P0 — Indispensable pour un usage partagé

1. **Hébergement statique** — Déployer sur GitHub Pages / Netlify / Vercel. Le fichier ne fonctionne pas en `file://` (CORS bloque fetch). Sans ça, aucun non-technicien ne peut utiliser l'outil. Alternative : un script shell/batch qui lance un serveur local (`python -m http.server`).
2. **Supprimer le mur de la clé API** — Demander à un utilisateur lambda de créer un compte OpenAI et coller une clé API est exactement la friction que l'outil est censé éliminer. Options : backend léger qui proxy les appels IA, intégration d'un modèle local (WebLLM / Ollama), ou mode dégradé avec templates de décomposition pré-faits (pas d'IA requise). ✅ **Templates intégrés** — 12 corvées universelles avec matching fuzzy, zero API.
3. **Onboarding** — La liste "Tuto" est un bon début mais ne montre pas pourquoi c'est différent d'un Trello. Ajouter une première décomposition guidée ("Essayez : ranger mon bureau") qui rend le concept tangible en 30 secondes.

### P1 — Utile au quotidien

4. **Persistance robuste** — localStorage est fragile (clear navigateur, changement de machine = perte totale). Options : sync fichier local, WebDAV, GitHub Gist, ou au minimum un rappel périodique "Pensez à exporter". L'export JSON existe mais il est manuel.
5. **Feedback de progression** — La barre de progression existe mais il n'y a pas de gratification quand on termine une tâche. Un micro-feedback (animation, compteur de streak, "5 tâches terminées aujourd'hui") renforcerait la boucle motivationnelle — c'est central pour un outil anti-procrastination.
6. **UX mobile** — Le responsive existe mais les raccourcis clavier (cœur de l'UX power-user) disparaissent sur mobile. Repenser le pattern mobile avec des gestes ou des boutons d'action rapide pour "ouvrir, décomposer, cocher".

### P2 — Nice to have

7. **Notes markdown** — Rendu basique (gras, italique, listes, code inline) dans la textarea de notes, toggle édition/aperçu (~70-110 LOC).
8. **Tags** — Champ `tags: []` sur les cartes, chips colorés, filtrable via la recherche (~70-85 LOC).
9. **Mode offline complet** — Service worker pour un fonctionnement 100% hors-ligne, cohérent avec la philosophie zéro-dépendance.

## Changelog

### 2026-06-29 — Templates de décomposition
- **12 templates intégrés** : corvées universelles (vaisselle, lessive, rangement, courrier, facture, mail, dossier, réunion, RDV médical, départ, boîte mail, fichiers bureau) avec micro-tâches pré-générées.
- **Matching fuzzy** : au clavier dans le textarea de décomposition, détection automatique du template le plus proche et affichage d'un bandeau `💡 Template suggéré : X — Utiliser ce template`.
- **Zero API** : les templates créent une liste nommée d'après le template en position 0, sans aucun appel réseau.
- `createTemplateList(template)` : création atomique liste + cartes via `DB.batch()`.
- CSS : bandeau `.template-banner` avec `color-mix(var(--accent) …)`, hover accent.

### 2026-06-29 — Correctifs templates
- **Bug — Double-clic bandeau template créait des listes dupliquées** : le handler de clic du `.template-banner` n'avait pas de guard anti-double-clic (contrairement au bouton de décomposition IA qui fait `btn.disabled = true`). Ajout d'une classe `.disabled` (`pointer-events: none`, `opacity: .5`) + `finally` pour réactiver.
- **Bug — Aucune gestion d'erreur sur clic template** : si `createTemplateList` levait une exception (ex: quota localStorage), `close()` n'était jamais appelé, la modale restait ouverte sans feedback. Ajout d'un `try/catch` avec `flashMessage` sur le `statusEl`.
- `open()` : reset défensif de la classe `disabled` à l'ouverture de la modale.
- **Tab active le template** : dans le textarea de décomposition, `Tab` focus le template suggéré (tabindex dynamique), `Enter`/`Space` l'active. Un 2e `Tab` amène au bouton "Décomposer" pour l'IA.

### 2026-06-29 — Feedback de progression (streak)
- **Streak compteur** : `kanbanjs:streak` dans localStorage (`{ streak, lastDate, todayCount, todayDate }`). Incrémenté à chaque complétion (drag vers liste Done ou toggle liste terminée). Reset à 1 si un jour est sauté.
- **Badge streak** : badge `🔥 N` dans le header, visible quand streak ≥ 2. État "paused" (opacité réduite) quand le jour en cours n'a pas encore de complétion, "actif" sinon.
- **Compteur du jour** : `· X aujourd'hui` en couleur accent dans le header info.
- **Animation pulse** : la barre de progression pulse (brightness) à chaque nouvelle complétion.
- Survit au reset/import (clé séparée de `kanbanjs:state`), auto-correction naturelle en sautant un jour.

### 2026-06-29 — Robustesse undo/IA/paste
- **Snapshot atomique sur paste carte** : `_paste()` wrappe `createCard` + `updateCardNotes` dans un seul `mutate()`. Avant, l'undo après un collage avec notes perdait les notes.
- **Suppression de `location.reload()`** : undo, redo, reset et import reconstruisent le DOM via `_rebuild()` au lieu de recharger la page. Plus de flash visuel, scroll et sélection préservés.
- **`max_tokens` explicite** : ajout de `max_tokens: 2048` pour OpenAI et `maxOutputTokens: 2048` pour Google Gemini. Anthropic avait déjà `max_tokens: 4096`. Évite les troncatures de réponse JSON sur les petits modèles.
- Extraction de `_rebuild()` : la reconstruction DOM (board name, listes, cartes, widget, FAB badge) est extraite de `init()` et réutilisée par undo/redo/reset/import.

### 2026-06-21 — Bugs finaux
- **Bug 1 — `getAfter` cartes recevait `x` au lieu de `y`** : le moteur DnD appelle `cfg.getAfter(container, x, y, ghostEl)` mais le callback carte déclarait `(container, y)`, donc `y` recevait la valeur de `x`. Corrigé en `(container, x, y)`.
- **Bug 2 — Cache `_cardCache` jamais invalidé entre listes** : le cache des rects des cartes était construit pour une liste et réutilisé tel quel quand le drag survolait une autre liste. Corrigé en keyant le cache par container (`_cardCache._container`).
- **Bug 3 — Double snapshot undo sur « Sauver »** : blur + click déclenchaient `saveEdit()` deux fois. Corrigé avec un guard `if (!wrap.classList.contains('editing')) return;` en tête de `saveEdit`.
- **Bug 4 — « Annuler » sauvegardait quand même** : `exitEdit()` retirait `.editing`, puis le blur appelait `saveEdit()`. Le guard du Bug 3 corrige aussi ce bug.
- **Bug 7 — `importFile.value = ''` hors callback** : reset de l'input synchrone avant `onload`. Déplacé dans un `finally` du `onload` et dans `onerror`.
- **Bug 8 — `_selList` orphelin après suppression de liste** : le handler appelait `selectCard(null)` mais ne désélectionnait pas `_selList`. Remplacé par `clearSelection()`.
- **Bug 12 — Recherche sélectionnait carte dans liste repliée** : cliquer sur un résultat de recherche dans une liste collapsed sélectionnait la carte sans la rendre visible. Ajout d'un dépliage automatique.

### 2026-06-21 — Architecture
- **Frontières DB** : ajout de `DB.exportJSON()`, `DB.importJSON(json)`, `DB.reset()`, `DB.restoreJSON(json)`, `DB.getAllCards()`. Plus aucun accès à `localStorage` / `DB.KEY` hors du module DB.
- **DOM source de vérité** : `refreshHeaderInfo()` et `refreshCountBadge()` lisent les données via DB (pas `querySelectorAll`).
- **Init parallèle** : `init()` utilise `DB.getAllCards()` + `for...of` au lieu d'un `forEach` avec `await`.
- **buildList/buildCard synchrones** : les cartes sont pré-fetchées par l'appelant.
- **Presse-papier encapsulé** : `Clipboard` objet avec méthodes explicites, plus de variable globale `_clipboard`.
- **mutate helper** : `mutate(fn)` = `_snapshot()` + `fn()`. Remplace ~15 appels `_snapshot()` raw.
- **Theme API** : `const Theme = window.__themeAPI` capturé au début de App.
- **removeListDom(listId)** : fonction partagée pour retirer une liste du DOM par ID.
- `_rebuild()` reconstruit tout le DOM à partir de l'état DB. Utilisée par undo, redo, reset, import et `init()`. Remplace tous les anciens `location.reload()`.

### 2026-06-21 — Qualité/DRY
- `toggleForm(btn, form, input, open)` : helper pour le pattern open/close de formulaire.
- `registerOverlay(overlay, closeFn)` : helper pour fermeture overlay (clic extérieur + Escape).
- `formatDoneAt(iso)` : formatage de date unifié ("Terminé le JJ/MM/AAAA à HH:MM").
- `extractCardData(el)` / `extractListData(el)` : extraction des données depuis le DOM.

### 2026-06-21 — Perf/robustesse
- Cache `getBoundingClientRect()` DnD : rects listes et cartes capturés une fois au début du drag.
- `DB.batch(fn)` : suspend les écritures localStorage le temps de la batch, persiste en une fois.
- Ghost DnD liste simplifié : `ghostBuilder` crée un clone minimal (header sans boutons).
- Import quota pre-check : alerte si > 4.5 MB.
- `AbortController` cleanup : `clearTimeout` dans `finally` du fetch.
- Taille snapshot limitée à 200 KB.
- Bugfix : `await` dans `forEach` → `for...of`.
- Bugfix : double `DB.exportJSON()` dans `_snapshot()`.

### 2026-06-21 — Visuel (CSS)
- Couleurs hardcodées (`rgba(232,168,90,…)`) remplacées par `color-mix(in srgb, var(--accent) X%, transparent)` (10 règles CSS).
- `.add-list-btn` : suppression des 16 lignes de surcharge par thème, `:hover` unique avec `color-mix`.
- Debug panel : couleurs → variables de thème (`--surface-2`, `--text`, `--accent`, `--danger`).
- Ghost DnD : `box-shadow: 0 8px 32px rgba(0,0,0,.6)` → `box-shadow: var(--shadow)`.
- `.cfg-status.success` : `#2ecc71` → `var(--accent)`.
- Responsive : `.cheatsheet-intro` masqué, `--list-width` réduit à 260px sur mobile.
- États visuels : `:focus-visible`, `:active`, `:disabled` sur boutons, `cursor: default` sur `.cheatsheet .row`.
- Fallback `rgba(var(--accent-rgb), .X)` avant chaque `color-mix()` pour les WIP warnings.

### 2026-06-21 — Bugs
- Guards null DOM sur `querySelector` dans `cutCard`, `copyCard`, `cutList`, `copyList` et la recherche.
- `DB.toggleListDone(id, doneAt)` : second paramètre optionnel, supprime l'appel redondant à `setCardsDoneAt()`.
- `reader.onerror` ajouté sur l'import JSON.
- Centralisation Escape : le handler dédié de la cheatsheet est supprimé, géré par `BINDINGS['Escape']`.
- `_selCard` orphelin : guard dans `delListBtn` (si carte sélectionnée appartient à la liste supprimée).
- Ordre DnD DOM/DB : `wrap.dataset.listId = toListId` après `await mutate(() => DB.moveCard(...))`.

### 2026-06-20
- Raccourcis clavier vim-like (~160 LOC) : navigation h/l/j/k, actions n/r/e/x/y/p + majuscules, focus 1-9, cheatsheet `?`.
- Presse-papier unifié, sélection visuelle (bordure accent, actions toujours visibles).
- Cheatsheet toggleable (overlay, deux colonnes carte/liste, principes kanban, responsive).
- Indicateur de progression (barre + pourcentage dans le header).
- WIP warning (bordure + badge accent quand liste In Progress ≥ 3 cartes).
- Repli des listes (bouton ▾/▸, état persisté).
- Liste "Waiting" ajoutée par défaut (kanban canonique).
- Undo/Redo (u/Ctrl+Y, 30 snapshots sessionStorage).
- Bugfixes : bleed-through clavier sur overlay recherche, Escape cheatsheet, éditeur bloqué si texte vide, import quota, _selCard orphelin, contextmenu DnD.

### 2026-06-19
- `DB.setCardsDoneAt(listId, doneAt)` : méthode batch pour les écritures groupées.
- `_save()` wrappé dans try/catch (QuotaExceededError → alert).
- `parseAITasks` : détection des guillemets dans le compteur de crochets.
- Prompt OpenAI : ajout d'un message `role: 'user'` (compatibilité élargie).
- `flashMessage(el, msg, cls, ms)` : fonction partagée.
- `close()` de la modale décomposition : nettoyage du timer de statut.
- `maxlength="200"` sur les inputs de nom (board et listes).

### 2026-06-17
- Timeout 60s sur fetch API via `AbortController`.
- `init()` wrappé dans try/catch avec bouton de réinitialisation en cas de crash.
- `createSuggestionsList` : écritures DB avant DOM.
- Chaînes vides filtrées dans `parseAITasks`, `readAIConfig()` unifiée.
- DB.KEY exposée (plus de littéral dupliqué).
- Blur sur éditeur de carte → sauvegarde automatique.
- Liste "terminé" : bouton ✓, cartes barrées, timestamp de complétion, mis à jour au DnD.
- Sélecteur de thème : 8 variantes (4 sombres, 4 clairs) via onglet dans Configuration.
- `initTheme` refactoré avec 3 clés (mode/dark/light) et mode système (`prefers-color-scheme`).
- Abstraction fournisseur IA : switch OpenAI/Anthropic/Google.
- DnD tactile : `mousedown` → `pointerdown` + `setPointerCapture` + `touch-action: none`.
- Responsive : media query ≤640px, header compact, modales scrollables.
- Export/Import JSON, FAB badge rouge si API non configurée.
- Tuto : liste avec cartes-exemples dans `DB._default`.
