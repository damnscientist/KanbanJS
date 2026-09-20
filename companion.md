# Companion de poche KanbanJS

## Philosophie

> Le mobile est pour la **capture** (sortir une idée de sa tête avant qu'elle ne s'évapore).
> Le desktop est pour l'**exécution** (organiser, prioriser, cocher).

Un kanban complet sur téléphone est une friction cognitive. Mais ne pas pouvoir noter une tâche
qui surgit dans la rue ou dans le canapé est une perte. Le companion est un **entonnoir minimal** :
un champ texte, un bouton envoyer, et c'est tout.

## Spécifications fonctionnelles

- Un champ `<input>` ou `<textarea>` avec placeholder "Note rapide…"
- Un bouton "Envoyer"
- Une liste des 10 dernières cartes ajoutées sur le board (read-only), avec :
  - Le texte de la carte
  - Le nom de la liste contenant la carte
  - Un indicateur visuel du statut (barré si liste terminée)
- Un mini-header avec :
  - Le streak actif (`🔥 12`)
  - Le compteur du jour (`3 aujourd'hui`)
  - Le pourcentage de progression

## Spécifications techniques

### Fichier

`companion.html` — fichier unique, aucune dépendance. Lit et écrit dans le même
`localStorage` que `index.html` (clé `kanbanjs:state`). La clé streak
(`kanbanjs:streak`) est lue en lecture seule.

### Mécanique d'écriture

À l'envoi :
1. Créer une liste "Inbox" en position 0 si elle n'existe pas déjà
2. Créer la carte dans cette liste, avec `createdAt: new Date().toISOString()`
3. Persister dans `kanbanjs:state`

La carte arrive donc directement dans le board principal, visible au prochain
rafraîchissement de `index.html` (ou immédiatement s'il est ouvert dans un autre onglet
— l'utilisateur doit rafraîchir manuellement pour l'instant).

### Modifications nécessaires dans `index.html` (module DB)

Le companion a besoin de deux ajouts dans le module `DB` :

1. **Champ `createdAt` sur les cartes** :
   - Ajouter `createdAt` dans `DB.createCard(listId, text)` : valeur par défaut `null`
   - Ajouter `createdAt: null` dans le schema `_default` (cartes tuto)
   - Ajouter `createdAt` dans les cartes retournées par `getCards()` et `getAllCards()`

2. **Nouvelle méthode `DB.getRecentCards(n)`** :
   - Retourne les N cartes les plus récentes triées par `createdAt` décroissant
   - Filtre les cartes sans `createdAt` (anciennes cartes)
   - Chaque carte inclut le `title` de sa liste pour affichage contextuel

### Structure HTML cible

```html
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no" />
  <title>KanbanJS · Companion</title>
  <style>
    /* CSS minimal ~80 lignes : fond noir, input full-width,
       liste scrollable, couleurs héritées des tokens kanban */
  </style>
</head>
<body>
  <div id="app">
    <header>
      <span id="streakInfo"></span>
      <span id="progressInfo"></span>
    </header>
    <main>
      <form id="captureForm">
        <input id="captureInput" placeholder="Note rapide…" autocomplete="off" />
        <button type="submit">Envoyer</button>
      </form>
      <ul id="recentCards"></ul>
    </main>
  </div>
  <script>
    /* DB minimal (lecture/écriture localStorage, createCard,
       getRecentCards, streak read-only) */
    /* UI ~100 lignes : rendu de la liste, soumission, feedback */
    App.init();
  </script>
</body>
</html>
```

### DB partagée

Le companion duplique une version allégée du module `DB` de `index.html` :

- `DB.getLists()` — lecture seule
- `DB.getRecentCards(n)` — nouvelle méthode
- `DB.createCard(listId, text)` — création de carte
- `DB.batch(fn)` — pour la création atomique liste + carte si Inbox n'existe pas
- `DB.createList(title)` + `DB.reorderList(id, 0)` — si Inbox absente

Le companion n'a PAS besoin de : rename, delete, move, undo, export, DnD, IA.

### Streak (lecture seule)

Lecture de `kanbanjs:streak` (clé distincte) :
```js
function readStreak() {
  try {
    const raw = localStorage.getItem('kanbanjs:streak');
    return raw ? JSON.parse(raw) : { streak: 0, lastDate: null, todayCount: 0, todayDate: null };
  } catch (_) { return { streak: 0, lastDate: null, todayCount: 0, todayDate: null }; }
}
```

## Prompt d'implémentation

```
Crée un fichier `companion.html` — un companion mobile minimal pour KanbanJS.

Contexte : KanbanJS est un kanban anti-procrastination dans un fichier unique
`index.html` (~3400 lignes). Il stocke ses données dans localStorage sous les
clés `kanbanjs:state` et `kanbanjs:streak`.

Le companion est une version ultra-light pour mobile permettant uniquement de
capturer des idées rapides. Pas d'édition, pas de DnD, pas de clavier.

## Objectif

Permettre à l'utilisateur, depuis son mobile, d'envoyer une tâche vers son
board kanban principal. La tâche arrive automatiquement dans une liste "Inbox".

## Fonctionnalités

1. **Header :** affiche le streak (🔥 N) si ≥ 2, et le compteur du jour
   (X aujourd'hui). Lit `kanbanjs:streak` en lecture seule.

2. **Formulaire de capture :** un input text + bouton "Envoyer". À la
   soumission :
   - Si la liste "Inbox" n'existe pas, la créer en position 0
   - Créer une carte avec `createdAt: new Date().toISOString()` dans la liste Inbox
   - Persister dans `kanbanjs:state`
   - Vider l'input, afficher un feedback visuel fugace
   - Rafraîchir la liste des cartes récentes

3. **Liste des cartes récentes :** les 10 dernières cartes triées par
   `createdAt` décroissant. Chaque item affiche :
   - Le texte de la carte
   - Le nom de la liste (en plus petit/grisé)
   - Si la liste est marquée `done: true`, le texte est barré

4. **Persistance partagée :** lecture/écriture dans `kanbanjs:state` via
   localStorage. La structure exacte est :
   ```json
   {
     "board": { "id": "board-1", "name": "Mon Board" },
     "lists": [{ "id": "_xxx", "title": "Inbox", "position": 0, "done": false, "collapsed": false }],
     "cards": [{ "id": "_xxx", "listId": "_xxx", "text": "texte", "position": 0, "notes": "", "doneAt": null, "createdAt": "2026-..." }]
   }
   ```

## Contraintes techniques

- Fichier unique, zéro dépendance externe
- Pas de `createCard` qui écrase le localStorage entre les deux onglets (utiliser
  un lock simple ou juste accepter le race)
- L'input doit marcher avec `Enter` ET avec le bouton
- Aucune gestion d'état complexe : pas de undo, pas d'édition, pas de suppression
- CSS minimal adapté au viewport mobile (max-width: 480px cible, pas de media query)
- La liste des cartes récentes se met à jour automatiquement après chaque envoi
  (relire `kanbanjs:state` depuis localStorage pour éviter les conflits inter-onglets)

## Style visuel

Reprendre les tokens de couleur de index.html (fonds sombres par défaut) :
- Fond : #1a1410, surface : #241e1a, texte : #e0d8d0, accent : #e8a85a
- Police système, 16px minimum sur input (anti-zoom iOS)
- ~120 lignes de CSS max

## Ne PAS faire

- Pas de thème (toujours sombre)
- Pas de drag & drop
- Pas de modales
- Pas de raccourcis clavier
- Pas d'IA
- Pas d'undo/redo
- Pas d'export/import
- Pas de barre de progression
- Pas de WIP warning
```

## Complexité estimée

- `companion.html` : ~200 lignes (HTML + CSS + JS)
- Modifications `index.html` (module DB) : ~30 lignes
- Total : ~230 lignes
