# KanbanJS — Projet anti-procrastination

Fichier unique `index.html` (~4216 lignes) + `sw.js` (service worker, 26 lignes). Aucune dépendance, pas de bundler. S'ouvre dans un navigateur moderne. Publié sur GitHub Pages : https://damnscientist.github.io/KanbanJS/

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
- Tests : 30 tests DB via iframe/postMessage (`test.html`)

**UX & design**
- Raccourcis clavier vim-like complets : navigation h/l/j/k, actions carte/liste, recherche `Ctrl+K`, undo/redo
- 8 thèmes (4 sombres, 4 clairs) avec mode système et 3 clés de persistance
- Streak + compteur du jour + paliers emoji (👍⚡🔥🚀💪) + confettis canvas à 100% — boucle motivationnelle
- 50 quotes bienveillantes en rotation, WIP warning, repli de listes, markdown riche dans les notes, tags colorés
- Accessibilité : `aria-label` sur 20+ boutons, `label[for]`, `prefers-reduced-motion`

### Faiblesses

**Persistance & sécurité**
- **localStorage = perte totale au `clear` navigateur** ou changement de machine. L'export JSON existe mais est manuel, sans rappel automatique
- **Pas de sync multi-onglets** — le dernier à écrire gagne, pas de `storage` event listener
- **Clé API en clair** dans `localStorage` — lisible par toute extension ou script cross-origin

**Fonctionnel**
- Pas de **companion mobile** implémenté — `companion.md` est une spec, pas du code. La capture rapide sur mobile n'existe pas
- Pas de **dates d'échéance** sur les cartes — utile pour les freelances avec deadlines
- Pas de **notifications** — le streak est silencieux hors de l'onglet

**Architecture — points d'attention (non bloquants)**
- `refreshHeaderInfo()` appelé après chaque mutation (création, suppression, move, cut/paste, toggle). Lit `getLists()` + `getAllCards()` + streak à chaque fois. Un debounce de 50ms réduirait le coût sans perte de réactivité
- `extractCardData()` lit le DOM (`textContent`, `value`) plutôt que la DB — en cas de désynchronisation, le clipboard contient des données périmées
- `_rebuild()` reconstruit tout le DOM à chaque undo/redo/reset/import. Correct pour la taille actuelle (~50-100 cartes), deviendrait coûteux au-delà

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
- Actions carte : `n` créer (fallback Backlog si aucune liste sélectionnée), `r` renommer (dblclick), `e` ouvrir notes (cycle édition→preview→fermé), `c` dupliquer, `t` ajouter un tag, `x` couper, `y` copier, `p` coller, `d` marquer terminée
- Actions liste : `N` créer, `R` renommer, `D` toggle terminé, `C` replier/déplier, `X` couper, `Y` copier, `P` coller
- `E` plier/déplier toutes les notes (preview), `Ctrl+drag` = copier une carte au lieu de déplacer
- `Esc` désélectionne la carte/liste courante ou ferme l'overlay actif, `?` affiche l'aide clavier
- Presse-papier unifié : couper/copier une liste copie titre + cartes
- Sélection visuelle : bordure accent, actions toujours visibles sur carte sélectionnée
- Ignore automatiquement quand un input/textarea a le focus ou une modale est ouverte
- `Ctrl+K` : recherche fuzzy sur les cartes du board (texte, notes, tags, titre de liste)
- `u` undo, `Ctrl+Y` redo : 30 niveaux de snapshots (session uniquement)

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
- `localStorage` avec 4 clés :
  - `kanbanjs:state` → board, listes, cartes (reset nettoie uniquement celle-ci)
  - `kanbanjs:config` → provider, endpoint, apiKey, model
  - `kanbanjs:streak` → `{ streak, lastDate, todayCount, todayDate }` (export/import inclus)
  - Thème (3 clés) :
    - `kanbanjs:theme-mode` → 'dark' | 'light' | 'system'
    - `kanbanjs:theme-dark` → 'warm-night' | 'deep-ocean' | 'forest' | 'tokyo-night'
    - `kanbanjs:theme-light` → 'soft-sand' | 'mint' | 'lavender' | 'tokyo-light'

## Architecture

### Modules (dans l'ordre dans le fichier)

| Module | Responsabilité | API publique |
|---|---|---|
| `initTheme` | IIFE, lit/applique/persiste le thème | lecture au load |
| `DB` | Adapter localStorage | `getBoard, renameBoard, getLists, createList, renameList, deleteList, reorderList, toggleListDone, toggleCollapsed, getCards, getAllCards, createCard, updateCard, updateCardNotes, updateCardTags, setCardDoneAt, setCardsDoneAt, deleteCard, moveCard, exportJSON, importJSON, reset, restoreJSON, batch` |
| `DnD` | Moteur de drag & drop générique | `start(dragEl, id, { ghostEl?, ghostClass?, phClass?, getZone, getAfter, getPos, skip? }, onDrop)` |
| `App` | UI Kanban | `init()` boot la session |

### Mécanique DnD
- `pointerdown`/`pointermove`/`pointerup` — API unifiée
- Seuil 4px avant démarrage (évite les faux positifs)
- Ghost : clone de l'élément ou `ghostBuilder` optionnel, fixed, `pointer-events:none`
- `getZone(x,y)` : itère sur `getBoundingClientRect()` des cibles (pas de `elementFromPoint`)
- Guard `lastPos` : mutation DOM uniquement si la position a changé
- Cache des `getBoundingClientRect()` construit au démarrage du drag, évite les reflows à chaque `pointermove`
- `cleanup()` après `dropCb()` (capture locale de la callback)
- Handler `pointercancel` + `cleanup()` défensif (libération capture persistante)

### Design
- Dark/light mode via `data-theme` sur `<html>` + CSS custom properties
- Modals : overlay `z-index: 20000`, centrées
- FAB : `z-index: 5000`, `position: fixed` bottom-right
- 4 thèmes sombres (Warm Night, Deep Ocean, Forest, Tokyo Night) et 4 clairs (Soft Sand, Mint, Lavender, Tokyo Light)
- Couleurs CSS : `color-mix(in srgb, var(--accent) X%, transparent)` + fallback `rgba(var(--accent-rgb), .X)` pour compatibilité

### Limites techniques
- **Ouverture en `file://`** : l'API IA est bloquée par CORS. L'app fonctionne en `file://` pour le board, les templates et la persistance, mais les appels IA nécessitent d'être servi en `http(s)` — d'où la publication GitHub Pages.
- **Service worker (`sw.js`)** : cache-first, chemins relatifs (`'./'`), guard non-GET (les POST IA ne doivent pas passer par la Cache API). Enregistré dans `index.html` uniquement en `https:`. **Toute modification publiée d'`index.html` ou `sw.js` doit bumper le nom de cache (`kanbanjs-v1` → `v2`)**, sinon les visiteurs gardent l'ancienne version.
- **Fournisseur IA centralisé** : tout le branchement fournisseur est dans `queryAI` (switch provider → body/headers/parsing)

### Déploiement
- **GitHub Pages** : https://damnscientist.github.io/KanbanJS/ (branche `main`, dossier `/ (root)`)
- Repo : https://github.com/damnscientist/KanbanJS (public, `origin` en SSH)
- Fichiers publiés : `index.html`, `sw.js`, `test.html`, `README.md`, `AGENTS.md`, `companion.md`, `LICENSE`, `VERSION`, `.nojekyll`
- Fichiers de process (prompts `01`–`05`, `audit.md`, `github.md`) : non publiés, purgés de l'historique, ignorés via `.gitignore`. **Ne pas les re-tracker.**
- Auth push : clé SSH ed25519 ajoutée au compte damnscientist. Le remote `origin` est en `git@github.com:...`

### Décisions non retenues
- **Multi-boards** : sur-ingénierie pour un outil mono-utilisateur. Les listes séparent déjà les contextes.

## Pour reprendre le développement

### Conventions et points d'entrée
- `mutate(fn)` est le seul point d'entrée pour les snapshots undo. Ne jamais appeler `_snapshot()` directement — utiliser `await mutate(() => DB.xxx(...))`.
- `DB.batch(fn)` suspend les écritures `localStorage` le temps de l'exécution, puis persiste en une fois. Utiliser pour les opérations groupées (création multiple de cartes, import de liste avec cartes).
- Les helpers `toggleForm`, `registerOverlay`, `formatDoneAt`, `extractCardData`, `extractListData`, `removeListDom` sont dans le scope App, juste après `removeListDom`.
- **Dates** : toujours passer par `dateKey(date)` (clé `YYYY-MM-DD` **locale**) et `isToday(date)` pour toute logique de jour (streak, compteur, comparaison `doneAt`). Ne jamais utiliser `toISOString().slice(0, 10)` pour une date logique — c'est la date UTC. Seul usage légitime restant : le nom de fichier d'export.
- Le thème est capturé au démarrage de App via `const Theme = window.__themeAPI`. Plus aucun accès à `window.__themeAPI` dans le code métier.
- `Clipboard` est un objet (plus un `let _clipboard`), méthodes : `copyCard/cutCard/copyList/cutList/paste/clear/isEmpty/type`.
- **Documents à tenir à jour** : toute évolution fonctionnelle ou visible du logiciel doit être répercutée dans le `README.md` (fonctionnalités, quick start, raccourcis, nombre de tests) en plus du changelog d'AGENTS.md. Le README est le seul document lu par les utilisateurs ; un README périmé est un bug de documentation.

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

1. **Hébergement statique** — Déployer sur GitHub Pages / Netlify / Vercel. Le fichier ne fonctionne pas en `file://` (CORS bloque fetch). Sans ça, aucun non-technicien ne peut utiliser l'outil. Alternative : un script shell/batch qui lance un serveur local (`python -m http.server`). ✅ **Déployé** — https://damnscientist.github.io/KanbanJS/ (branche `main`, racine, service worker actif).
2. **Supprimer le mur de la clé API** — Demander à un utilisateur lambda de créer un compte OpenAI et coller une clé API est exactement la friction que l'outil est censé éliminer. Options : backend léger qui proxy les appels IA, intégration d'un modèle local (WebLLM / Ollama), ou mode dégradé avec templates de décomposition pré-faits (pas d'IA requise). ✅ **Templates first** — 12 corvées universelles avec matching fuzzy, grille de templates dans la modale quand l'API n'est pas configurée, FAB toujours invitant (plus de `!` rouge).
3. **Onboarding** — La liste "Tuto" est un bon début mais ne montre pas pourquoi c'est différent d'un Trello. Ajouter une première décomposition guidée ("Essayez : ranger mon bureau") qui rend le concept tangible en 30 secondes. ✅ **Onboarding interactif** — Tuto reformulé en consignes actionnables, liste "Ranger une surface" avec 6 micro-tâches d'exemple (template réel) pour un premier drag en <10s.
4. **Corriger le bug de streak (dates UTC)** — Voir Faiblesses. Fix : un helper unique `dateKey(date)` retournant `YYYY-MM-DD` **local** (formatage manuel `getFullYear`/`getMonth`/`getDate`, ou `toLocaleDateString('sv-SE')`), utilisé partout où `toISOString().slice(0, 10)` apparaît : `updateStreak`, `adjustStreak`, `refreshHeaderInfo` (compteur du jour, badge paused), et les comparaisons `doneAt.slice(0, 10) === today` des handlers de complétion/décomplétion (conversion de l'ISO en `Date` avant extraction de la clé locale, sinon `doneAt` reste en UTC et la comparaison décale). ~20 lignes. Ajouter un test dans `test.html` : une date construite à 23h30 locale doit donner la clé du jour local, pas celle du jour UTC. ✅ **Corrigé (2026-09-20)** — helpers `dateKey(date)` et `isToday(date)` dans le scope App, tous les usages remplacés ; 3 tests ajoutés (0h30, 23h30, 1er janvier). Le seul `toISOString().slice(0, 10)` restant est le nom de fichier d'export (légitime).

### P1 — Utile au quotidien

5. **Persistance robuste — le "fichier compagnon"** — localStorage est fragile (clear navigateur, changement de machine = perte totale, streak inclus). Principe retenu : **l'app apporte le mécanisme, l'utilisateur apporte le stockage** (zéro compte, zéro backend, zéro dépendance).
   - **Fichier compagnon (File System Access API)** : action "💾 Sauvegarde automatique…" → `showSaveFilePicker()` ; le `FileSystemFileHandle` est conservé dans IndexedDB (survit aux redémarrages). `DB._save()` déclenche un `_scheduleFileSave()` debouncé (~800 ms). Même format que `exportJSON()` (`{state, streak}`) — zéro nouveau format.
   - **Restauration en un clic** : au boot, si `kanbanjs:state` est vide/corrompu mais que le handle existe et que le fichier contient des données → bannière inline "Vos données ont disparu — Restaurer ?" → un seul clic → `_rebuild()`. Le public cible ne complètera jamais un flow de restauration en 5 étapes.
   - **Multi-machine gratuit** : si l'utilisateur range le fichier dans un dossier synchronisé (Drive, Dropbox, Syncthing), il obtient la sync sans une ligne de code côté app.
   - **`navigator.storage.persist()`** au premier `updateStreak()` — empêche l'éviction implicite par le navigateur. Ne protège pas du "effacer les données de navigation", d'où le fichier compagnon.
   - **Sync multi-onglets** (absorbé depuis P2) : `window.addEventListener('storage', …)` → debounce → `_rebuild()`. ~15 lignes. Prérequis si le companion voit le jour (la spec `companion.md` prévoit aujourd'hui d'"accepter le race").
   - **Self-heal** : dans `DB._load()`, si le JSON est corrompu, tenter une copie de secours avant de retomber sur `_default` — perdre son board pour un JSON cassé est insupportable.
   - **Replis et pièges** : Safari/Firefox n'ont pas les pickers → feature detection + repli sur l'export manuel existant et un rappel périodique ("Dernier export : il y a N jours") ; permission à réactiver à chaque session Chromium (état discret dans le header, jamais d'`alert()`) ; conflits multi-machines = dernier écrivain gagne (assumé, à documenter dans le README).
   - **Ne PAS faire** : WebDAV, GitHub Gist, backend proxy, compte utilisateur — chacun brise la philosophie zéro-compte sans mieux protéger que le fichier chez l'utilisateur.
   - **Estimation** : ~175 lignes (~150 fichier compagnon + ~25 persist/storage event/self-heal).
6. **Feedback de progression** — La barre de progression existe mais il n'y a pas de gratification quand on termine une tâche. ✅ **Streak + pulse + compteur du jour** implémentés.
7. **UX mobile / Companion** — La spec `companion.md` décrit un entonnoir minimal mobile (champ texte → liste Inbox). Le fichier `companion.html` (~200 lignes) reste à implémenter. La capture se fait sur mobile, l'exécution sur desktop.

### P2 — Nice to have

8. **Notes markdown** — ✅ **Implémenté** — cycle édition/preview/fermé, rendu complet (gras, italique, code, liens, titres, listes, ligne horizontale).
9. **Tags** — ✅ **Implémenté** — chips par hash couleur, bouton `+ tag`, raccourci `t`, préservés au clipboard, recherchables.
10. **Mode offline complet** — ✅ **Implémenté** — service worker cache-first (`sw.js`), la page fonctionne hors-ligne après la première visite.
11. **Dates d'échéance** — Champ `dueDate` sur les cartes avec indication visuelle (couleur selon proximité). Utile pour les freelances avec deadlines.
12. **Notifications navigateur** — `Notification API` : rappel quotidien du streak ou notification quand une carte arrive dans Done. Renforce la boucle motivationnelle hors de l'onglet.
13. **Son au clic de complétion** — Feedback auditif optionnel (toggle), renforce le sentiment d'accomplissement pour le public TDAH.
14. **Réorganisation clavier** — Alt+J/K pour déplacer une carte vers le haut/bas sans souris.
15. **Dashboard statistiques** — Vues semaine/mois (cartes terminées, répartition par liste, évolution streak).
16. **Import/export CSV et Markdown** — Interopérabilité avec d'autres outils.
17. **Fournisseur LM Studio** — Ajouter `lmstudio` au switch `queryAI`, sur le modèle de `ollama` : serveur local compatible OpenAI (`http://localhost:1234/v1/chat/completions` par défaut), clé API optionnelle (LM Studio n'en exige pas), timeout long (~5 min) car les modèles locaux sont lents. Parsing identique à OpenAI. Complète l'offre « IA locale sans compte » à côté d'Ollama — utile pour les utilisateurs qui préfèrent l'interface graphique de LM Studio pour gérer/télécharger leurs modèles.

## Changelog

### 2026-09-20 — Correctif streak UTC (v1.0.1)

- **Bug corrigé** : le streak, le compteur du jour et les comparaisons « terminé aujourd'hui » utilisaient la date UTC. Une tâche terminée après minuit (heure locale, fuseau UTC+) comptait pour le jour précédent — un utilisateur nocturne pouvait perdre son streak malgré un travail quotidien.
- **Helpers** : `dateKey(date)` (clé `YYYY-MM-DD` locale via `getFullYear`/`getMonth`/`getDate`) et `isToday(date)` dans le scope App. Remplacent 6 usages de `toISOString().slice(0, 10)` (`refreshHeaderInfo`, `updateStreak`, `adjustStreak`, 3 comparaisons `doneAt`).
- **Tests** : 3 tests ajoutés dans `test.html` (0h30, 23h30, 1er janvier) — 30/30. Le test à 0h30 échoue sur l'ancien code, il détecte donc bien la régression.
- **Service worker** : cache bumpé `kanbanjs-v1` → `v2`.

### 2026-09-20 — Publication GitHub Pages (v1.0.0)

- **Renommage** : `kanban.html` → `index.html` (via `git mv`), refs mises à jour dans `test.html`, `AGENTS.md`, `companion.md`.
- **Service worker** : `sw.js` créé (cache-first, chemins relatifs `'./'`, guard non-GET), enregistré dans `index.html` en `https:` uniquement.
- **Fichiers de publication** : `README.md` (français), `LICENSE` (MIT, Laurent Hofer), `VERSION` (1.0.0), `.nojekyll`, `.gitignore`.
- **Purge d'historique** : les 7 fichiers de process (`01`–`05`, `audit.md`, `github.md`) retirés du disque, de l'index et de tout l'historique (`filter-branch`, 81 → 80 commits). Sauvegardes hors dépôt : `kanbanjs-process-backup.tar.gz`, `kanbanjs-prepublic.bundle`.
- **Déploiement** : repo public `damnscientist/KanbanJS`, branche `main`, GitHub Pages actif — https://damnscientist.github.io/KanbanJS/. Auth push en SSH (clé ed25519).
- **Tests** : 27/27 après renommage. Vérification du site réel : chargement, 6 listes, thème, service worker `active`, zéro erreur console.

### 2026-07-13 — Ollama, confettis, service worker

- **Support Ollama** : nouveau fournisseur `ollama` dans le `<select>`, endpoint par défaut `http://localhost:11434/v1/chat/completions`. La clé API est optionnelle (masquée dans la modale), le timeout passe à 5 min pour les modèles locaux. Parsing identique à OpenAI (format compatible).
- **Confettis** : ne se déclenchent plus sur les opérations passives (chargement initial, undo, import, rename). La transition `< 100% → 100%` est désormais détectée via `_prevPct` (comparaison avant/après dans `refreshHeaderInfo`). `_rebuild()` pose `_prevPct = 100` pour que la reconstruction DOM ne soit jamais traitée comme une complétion.
- **Service worker** : spécifications ajoutées (cache-first, ~35 lignes) pour le mode hors-ligne après la première visite. Implémenté le 2026-09-20 (`sw.js`).

### 2026-07-01 — Tags, markdown enrichi, templates first, accessibilité, robustesse
- **Tags** : champ `tags: []` sur les cartes, chips colorés par hash du nom, bouton `+ tag`, input inline (Enter ajoute, Escape annule), suppression par ✕. Raccourci `t`. Tags préservés au copier/coller/dupliquer/paste de liste. Recherchables via `Ctrl+K`. DB: `updateCardTags(id, tags)`.
- **Markdown enrichi** : `#`, `##`, `###` → titres (h3/h4/h5), `- item` → listes à puces, `1. item` → listes numérotées, `[texte](url)` → liens cliquables, `---` → ligne horizontale.
- **Templates first** : quand l'API n'est pas configurée, la modale Décomposer affiche directement une grille des 12 templates + lien `⚙ Configurer l'IA`. Le FAB est toujours `+` (plus jamais `!` rouge). L'IA devient un bonus, pas un prérequis.
- **Accessibilité** : `label[for]` sur tous les champs de formulaire, `aria-label` sur les 20+ boutons (icônes header, actions carte/liste, FAB). Mise à jour dynamique des aria-labels (collapse, notes, FAB, `_toggleAllNotes`).
- **Robustesse** : `QuotaExceededError` affiche une bannière persistante dans le header (disparaît automatiquement quand l'espace se libère). `_snapshot()` avertit via `flashMessage` quand l'undo est désactivé (board > 200 ko). `parseAITasks` gère les code fences (\`\`\`json).
- **Correctifs** : `_rebuild()` préserve `scrollLeft`. `_confettiDone` se reset toujours après les confettis (permet plusieurs célébrations). `fuzzyMatch` dédupliqué en helper partagé. Le collapse bouton met à jour son `title`/`aria-label` au toggle.
- **Tests** : `test.html` — 30 tests DB via iframe + postMessage (reset, listes, cartes, export/import, streak, dates locales). Lancement : `python3 -m http.server`, puis `http://localhost:8000/test.html`.
- **Fichier** : ~3900 → ~4200 lignes (+300)

### 2026-07-01 — Onboarding, streak robuste, notes markdown, UX polie
- **Onboarding interactif** : Tuto reformulé en 5 cartes actionnables (1, 2, 2a, 3, 4), liste "Ranger une surface" avec 6 micro-tâches d'exemple. Le Backlog reste vide pour l'utilisateur.
- **Streak robuste** : `adjustStreak(delta)` pour décrémenter quand une carte quitte Done (DnD, toggle liste, suppression). Undo/Redo snapshotte le streak (clés parallèles `*-streak`). Badge visible dès streak ≥ 1. Palier d'emoji journalier (👍⚡🔥🚀💪) avec animation pop.
- **Confettis à 100%** : canvas overlay avec 150 particules quand la barre atteint 100%.
- **50 quotes bienveillantes** : affichées en filigrane sur le board (opacité 0.45), rotation à chaque action.
- **Notes markdown** : cycle édition→preview→fermé (≡, `e`, `Esc`). Rendu `**gras**`, `*italique*`, `` `code` ``. Bouton crayon supprimé, double-clic sur carte pour renommer. Pastille colorée sur les cartes ayant des notes.
- **Export/Import inclut le streak** : `exportJSON()` wrapper `{state, streak}`, `importJSON()` rétrocompatible.
- **Raccourcis ajoutés** : `c` dupliquer carte, `C` replier/déplier liste, `Ctrl+drag` copier carte (original reste en place, copie au placeholder).
- **DnD** : auto-scroll horizontal + vertical (40px des bords), `ph.remove()` corrigé sur élément détaché, ghost en `left`/`top` (pas de conflit `transform`).
- **Audit bugs** : FAB dupliqué supprimé, `saveEdit` double snapshot corrigé, `setCardDoneAt` wrappé dans `mutate()`, CSS mort nettoyé, `exitRename` async, `document.createElement` → `make()`.
- **Fichier** : 3438 → ~3900 lignes (+462)

### 2026-07-01 — Suppression de la couche UX mobile
- **Décision stratégique** : un outil anti-procrastination ne doit pas se faire sur mobile. Le kanban complet est une expérience desktop. Pour la capture rapide, voir `companion.md`.
- **CSS** : suppression du bloc `@media (max-width: 640px)` (~185 lignes), des classes `.mobile-toolbar`, `.header-menu-overlay`, `.header-menu-btn`, de `@keyframes slideUp`, de `touch-action: none` sur `.card`/`.list-header`/`.dnd-ghost-list`, de `-webkit-user-select` et `-webkit-touch-callout`.
- **HTML** : suppression de `#mobileToolbar`, `#headerMenuOverlay`, `#headerMenuBtn`, `#searchMobileBtn`, des classes `.header-desktop-only` et `.header-menu-btn`, des attributs `enterkeyhint`/`inputmode`.
- **JS** : suppression de `isMobile()`, des handlers mobile (recherche, menu ⋯, toolbar contextuelle, `MutationObserver`, `visualViewport.resize`), de la détection double-tap sur les titres, de la branche touch du DnD (tap-long 200ms, edge-scroll, `_touchDrag`, `webkitUserSelect`). `selectCard()` simplifié (plus de branche `isMobile`). `DnD.start()` simplifié (comportement souris uniquement).
- **Fichier** : 4024 → 3438 lignes (-586, -14.5%)

### 2026-06-29 — Correctifs mobile (post-refonte)
- **`touch-action: auto` annulé** : la tentative de passer les `.card` et `.list-header` en `touch-action: auto` sur mobile cassait le DnD. Le `e.preventDefault()` appelé dans `beginCapture()` arrive 200ms après le `pointerdown` — trop tard, le navigateur a déjà pris la main sur le scroll. Le DnD repose sur `touch-action: none` CSS (le navigateur ne scroll jamais sur les cartes, le JS gère tout). Le tap-long à 200ms est conservé pour laisser un délai entre le touch et l'activation du drag.
- **FAB recouvert par la toolbar** : le FAB (`z-index: 5000`) était masqué par la toolbar mobile (`z-index: 8000`) quand une carte était sélectionnée. FAB → `z-index: 8100` sur mobile + `showToolbar()`/`hideToolbar()` repositionnent le FAB à `calc(84px + safe-area)` au-dessus de la toolbar.
- **Sélection de texte iPhone pendant le DnD** : Safari iOS a besoin du préfixe `-webkit-user-select: none` — le `user-select` standard est ignoré. Ajout du préfixe sur `.card`, `.card-text`, `.list-header`, `.list-title`, `header h1`, `.board-name`. Ajout de `-webkit-touch-callout: none` sur `.card` et `.list-header`. Le JS pose aussi `webkitUserSelect` dans `startDrag()`/`cleanup()`.
- **Textarea de notes invisible jusqu'à la frappe** : `autoResize()` mesurait un `scrollHeight` potentiellement nul quand le textarea venait de passer de `display:none` à `display:block`. Hardening : guard `offsetParent`, `Math.max(scrollHeight, rows*20, 28)`, `rows=2` sur le textarea notes. `toggleNotes()` enrobe `autoResize()` dans un double `requestAnimationFrame`. Nouveau handler `visualViewport.resize` pour les notes sur mobile.
- **Délai tap-long réduit** : 320ms → 200ms. Le délai reste nécessaire pour distinguer scroll et drag sur touch, mais 200ms est suffisant et nettement plus réactif.
- **Recherche dans liste repliée** : le `colBtn.click()` pour déplier était async — la classe `.collapsed` n'était pas retirée avant `selectCard()`, donc la carte était sélectionnée mais invisible. Retrait synchrone de `.collapsed` avant `selectCard()`, puis `colBtn.click()` pour persister.
- **Bouton "Ajouter une liste"** : n'avait pas `scroll-snap-align`, causant un scroll non fluide en fin de board. Ajout de `scroll-snap-align: start` sur `.add-list-btn` et `.add-list-form` dans la media query mobile.
- **Zoom + click parasite + sélection liste pendant le DnD** : iOS zoomait et synthétisait un `click` au lâcher car `preventDefault()` n'était pas appelé au bon moment. Trois `e.preventDefault()` ajoutés : immédiat sur `pointerdown` touch (bloque la synthèse du `click`), dans `onMove` (bloque le zoom), dans `onUp` (ceinture+bretelles).
- **Edge-scroll pendant le DnD (touch)** : scroll automatique du board quand le doigt approche des bords. Horizontal pour atteindre les listes adjacentes, vertical pour les cartes hors vue dans la liste survolée. Vitesse proportionnelle à la distance du bord (`requestAnimationFrame`). Utilise `scrollBy({ behavior:'auto' })` et désactive temporairement `scroll-snap-type` pour contourner les limitations iOS. Invalidation des caches `_zoneCache`/`_cardCache` sur changement de `scrollLeft`/`scrollTop`. Activé uniquement pour `pointerType === 'touch'`, desktop souris inchangé.

### 2026-06-29 — Refonte UX mobile
- **Toolbar contextuelle mobile** : barre fixe en bas d'écran (`#mobileToolbar`, z-index 8000) avec 4 boutons (Modifier / Notes / Terminer / Supprimer). Apparaît au tap sur une carte via `MutationObserver` sur `.card.card-selected:not(.editing)`. Se masque automatiquement à l'entrée en mode édition, à la fermeture de sélection, ou au tap ailleurs.
- **Menu ⋯ header mobile** : bottom-sheet (`#headerMenuOverlay`, z-index 19000) regroupant les 6 actions secondaires (Config IA, Thème, Aide, Export, Import, Reset). Les boutons desktop correspondants sont masqués via classe `.header-desktop-only`.
- **Bouton recherche mobile** : icône 🔍 (`#searchMobileBtn`) dans le header, visible uniquement sur mobile, ouvre directement la search overlay.
- **DnD tactile tap-long** : sur `pointerType === 'touch'`, le drag ne s'active qu'après 200ms de pression immobile. Si le doigt bouge de > 8px avant le délai, le scroll natif est libéré. Souris et stylet restent immédiats. Résout le conflit scroll horizontal / drag.
- **Modales bottom-sheet** : sur mobile, toutes les `.modal-overlay` s'affichent en bas de l'écran (`align-items: flex-end`, `border-radius` top-only, `max-height: 90dvh`).
- **Hauteur listes `dvh`** : `max-height: calc(100dvh - 88px)` sur mobile — tient compte du clavier virtuel sur iOS/Android.
- **Layout board mobile** : `--list-width: 82vw`, `scroll-snap-type: x mandatory`, `overscroll-behavior-x: contain` — navigation liste par liste au swipe.
- **Cibles tactiles 40×40px** : `.btn-icon` agrandit à 40×40px, `.card-actions .btn-icon` à 38×38px, `.btn` avec padding 10×16px sur mobile.
- **Actions carte toujours visibles** : `.card-actions { display: flex !important; position: static }` sur mobile — fin du hover-only, boutons toujours accessibles au doigt.
- **FAB** : `safe-area-inset-bottom`, padding compensatoire sur `.list-footer` (72px) pour éviter le recouvrement.
- **Police cartes** : 15px (card-text), 13px (notes), 14px (list-title) sur mobile.
- **`enterkeyhint="send"`** et **`inputmode="text"`** sur le textarea de décomposition. `visualViewport.resize` → `scrollIntoView` quand le clavier virtuel s'ouvre.
- **`prefers-reduced-motion`** : animations non essentielles (fadeIn cartes/listes, pulse, slideUp) désactivées.
- **Bugfixes post-refonte** : `_selCard` capturé avant `clearSelection()` dans `mtDone` ; MutationObserver ignore `.card.editing` ; stylet (`pen`) traité comme souris (pas de délai tap-long).

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
- Extraction de `_rebuild()` : la reconstruction DOM (board name, listes, cartes, widget) est extraite de `init()` et réutilisée par undo/redo/reset/import.

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
- Responsive : `.cheatsheet-intro` masqué sur petit écran
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
- DnD : `mousedown` → `pointerdown` + `setPointerCapture`
- Responsive : media query ≤640px, header compact
- Export/Import JSON, FAB badge si API non configurée.
- Tuto : liste avec cartes-exemples dans `DB._default`.
