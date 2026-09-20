# KanbanJS — L'anti-procrastination bienveillante

Transformer une tâche lourde en une série de micro-actions à coût cognitif nul.

## Le concept

Face à une corvée qui paraît insurmontable (« ranger mon bureau », « faire ma compta »), le blocage vient rarement du manque de volonté : il vient du coût cognitif de la première étape. KanbanJS casse ce mur en décomposant la corvée en actions minuscules (30 secondes à 3 minutes), assez petites pour être lancées sans réfléchir. L'outil cible en priorité les personnes avec un TDAH, les étudiants qui procrastinent et les freelances submergés. Philosophie : **fichier unique, pas de compte, données locales** — tout tourne dans le navigateur, rien n'est envoyé nulle part (sauf si vous configurez vous-même une IA).

## Fonctionnalités

- **Board kanban** : listes et cartes, renommage, réordonnancement par glisser-déposer, repli des listes.
- **Décomposition IA** : décrivez une corvée, l'IA la découpe en micro-actions dans une liste dédiée. Supporte OpenAI (et compatible), Anthropic, Google Gemini et Ollama (modèles locaux).
- **Templates intégrés** : 12 corvées universelles (vaisselle, lessive, rangement, courrier, facture, mail, dossier, réunion, RDV médical, départ, boîte mail, fichiers du bureau) prêtes à l'emploi, avec suggestion automatique par correspondance floue — **aucune clé API requise**.
- **Liste « terminée »** : marquez une liste comme Done, ses cartes se barrent et affichent la date de complétion.
- **Notes markdown** : chaque carte peut contenir une note en markdown (gras, italique, code, liens, titres, listes, ligne horizontale), avec un cycle édition → aperçu → fermé.
- **Tags colorés** : chips colorés par nom, recherchables.
- **Streak & gratification** : compteur de jours consécutifs, compteur du jour, paliers emoji, confettis à 100 %, citations bienveillantes.
- **8 thèmes** : 4 sombres (Warm Night, Deep Ocean, Forest, Tokyo Night) et 4 clairs (Soft Sand, Mint, Lavender, Tokyo Light), avec mode système.
- **Undo/redo** : 30 niveaux de snapshots.
- **Raccourcis vim-like** : pilotage complet au clavier.
- **Export / import JSON** : vos données restent à vous, sauvegardables en un fichier.
- **Fonctionne hors-ligne** après la première visite (service worker).

## Quick start

1. **Ouvrir l'app** : rendez-vous sur [https://damnscientist.github.io/KanbanJS/](https://damnscientist.github.io/KanbanJS/) (ou ouvrez `index.html` via un petit serveur local, voir *Développement*). Tout fonctionne sans serveur, sauf les appels IA qui nécessitent d'être servi en `http(s)`.
2. **Choisir une corvée** : cliquez sur le bouton `+` en bas à droite → sélectionnez une corvée dans la grille de templates. Aucune configuration n'est nécessaire.
3. **Avancer** : glissez les cartes de `Backlog` → `In Progress` → `Done`, une micro-action à la fois.

## Configurer l'IA (optionnel)

L'IA est un bonus, pas un prérequis : les templates couvrent déjà les corvées du quotidien. Pour utiliser un modèle :

1. Cliquez sur ⚙ dans l'en-tête (ou le lien « Configurer l'IA » dans la modale `+`).
2. Choisissez un fournisseur :
   - **OpenAI** (ou toute API compatible) — endpoint `https://api.openai.com/v1/chat/completions`.
   - **Anthropic** — `https://api.anthropic.com/v1/messages`.
   - **Google Gemini**.
   - **Ollama** — pour un modèle local, endpoint par défaut `http://localhost:11434/v1/chat/completions`, clé API optionnelle.
3. Renseignez l'endpoint, la clé API et le modèle, puis validez.

La clé API est stockée en clair dans le `localStorage` de votre navigateur — elle n'est jamais envoyée qu'au fournisseur que vous avez choisi.

## Raccourcis clavier

Minuscule = agir sur la **carte**, majuscule = agir sur la **liste**.

| Contexte | Touche | Action |
|---|---|---|
| Navigation | `h` / `l` | Liste précédente / suivante |
| Navigation | `j` / `k` | Carte suivante / précédente |
| Navigation | `1`–`9` | Focus sur la n-ième liste |
| Carte | `n` | Nouvelle carte |
| Carte | `r` | Renommer |
| Carte | `e` | Notes (édition) |
| Carte | `c` | Dupliquer |
| Carte | `t` | Ajouter un tag |
| Carte | `x` / `y` / `p` | Couper / copier / coller |
| Carte | `d` | Marquer terminée |
| Liste | `N` / `R` | Nouvelle liste / renommer |
| Liste | `D` / `C` | Toggle terminé / replier |
| Liste | `X` / `Y` / `P` | Couper / copier / coller |
| Global | `E` | Plier/déplier toutes les notes |
| Global | `u` / `Ctrl+Y` | Undo / redo |
| Global | `Ctrl+K` | Recherche |
| Global | `Esc` | Désélectionner / fermer |
| Global | `?` | Aide clavier |

`Ctrl` + glisser une carte la **copie** au lieu de la déplacer.

## Développement

Le projet est un fichier unique, sans dépendance ni bundler. Pour lancer les tests de non-régression :

```bash
python3 -m http.server 8000
```

Puis ouvrez [http://localhost:8000/test.html](http://localhost:8000/test.html) (26 tests DB via iframe + postMessage).

## À propos du développement

Ce logiciel a été intégralement développé avec l'assistance d'un agent d'IA (OpenCode / Claude). Toutes les décisions architecturales, le design, et les révisions ont été pilotés par un humain. Le code est libre, ouvert, et fait pour durer.

## Licence

MIT — voir le fichier [LICENSE](LICENSE).
