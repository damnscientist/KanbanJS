# KanbanJS — Projet anti-procrastination

Fichier unique `kanban.html` (~2170 lignes). Aucune dépendance, pas de bundler. S'ouvre dans un navigateur moderne.

## Concept

Transformer une tâche lourde en micro-actions à coût cognitif nul via IA. L'utilisateur décrit une corvée → un agent d'IA la décompose en actions minuscules (30s-3min), créées dans une liste "Suggestions" dédiée. L'utilisateur les glisse ensuite dans "In Progress" → "Done".

## Fonctionnalités existantes

### Board
- Board unique avec listes et cartes
- Renommage du board (input dans le header)
- Header info : nombre de listes et cartes
- Thème jour/nuit persisté dans 3 clés (`kanbanjs:theme-mode`, `kanbanjs:theme-dark`, `kanbanjs:theme-light`)
- Sélecteur de thème dans la modale Configuration (onglet "Thèmes") : 4 sombres, 4 clairs
- Bouton reset (purge `kanbanjs:state`, rechargement)
- Export / Import JSON du board (boutons ↓ ↑ dans le header)

### Listes
- Création inline (bouton + formulaire "Ajouter une liste")
- Renommage (clic sur le titre)
- Suppression (confirmation)
- Défaut au premier lancement : "Tuto", "Backlog", "In Progress", "Done"
- Drag & drop souris pour réordonner horizontalement (DnD générique)
- **Liste "terminé"** : chaque liste a un bouton `✓` qui la marque comme "terminée".
  Les cartes d'une liste "terminée" s'affichent barrées/grissées.
  Par défaut, la liste "Done" est marquée terminée.
  Le statut se met à jour automatiquement au drag & drop entre listes.
  Les cartes d'une liste "terminée" affichent aussi la date de complétion.

### Cartes
- Création inline ("Ajouter une carte" dans chaque liste)
- Édition inline (textarea avec Sauver/Annuler, Enter valide, Escape annule)
- **Notes** : chaque carte a un bouton `≡` qui ouvre une textarea de note personnelle, sauvegardée au blur
- Suppression
- Drag & drop souris entre listes (ou au sein de la même liste)
- Drag ghost + placeholder visuel

### IA (Décomposition)
- FAB flottant `＋` en bas à droite
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
| `DB` | Adapter localStorage | `getBoard, renameBoard, getLists, createList, renameList, deleteList, reorderList, toggleListDone, getCards, createCard, updateCard, updateCardNotes, deleteCard, moveCard` |
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
- **Pas de déploiement** — fichier local, provoque des erreurs CORS si ouvert en `file://` (l'API fetch y est bloquée, à servir via un serveur local)
- **Fournisseur IA centralisé** : tout le branchement fournisseur est dans `queryAI` (switch provider → body/headers/parsing)

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

## Pour reprendre le développement

- L'ordre des modales et du debug suit le flow : config → décompose
- Le prompt système est dans `PROMPT_SYSTEM` (template literal, substitution `{{TASK}}`)
- La config API (provider, endpoint, apiKey, model) est stockée en localStorage, lue par `readAIConfig()` dans App
- Le fournisseur est branché dans `queryAI` via un switch (`'openai'` / `'anthropic'` / `'google'`), chaque branche construit body + headers + parsing de réponse
- Le board est initialisé avec 4 listes par défaut via `DB._default` (dont "Tuto" avec cartes-exemples)
- `buildList` est asynchrone (lit les cartes depuis DB)
- `buildCard` est synchrone (reçoit une carte déjà construite)
- Les listes ont un champ `done` (booléen) ; si vrai, leurs cartes affichent `.card-done` (barré + grisé)
- `buildCard(card, done)` accepte un second paramètre pour le style initial
- Les cartes ont un champ `notes` (chaîne) persisté via `updateCardNotes`, éditable inline (textarea toggleable)
- Les cartes ont un champ `doneAt` (ISO string ou null) stocké automatiquement quand glissées dans une liste "terminée"

## Convention de code
- `make(tag, className)` pour créer des éléments
- `autoResize(ta)` pour les textareas
- Tableaux de bord en `const`, fonctions helpers en closures
- Pas de commentaires dans le code (sauf en-têtes de sections)
- Noms en français (utilisateur francophone)
