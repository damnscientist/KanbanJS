# Changelog — KanbanJS

Historique des versions, de la plus récente à la plus ancienne. Le contexte de développement (architecture, conventions, roadmap) est dans [`AGENTS.md`](AGENTS.md).

## 2026-11-01 — Multiboard et suivi de projet (v1.2.0)

- **Plusieurs boards** : pour suivre les projets d'une équipe à côté du board personnel (1 carte = 1 projet, suivi dans la note). Menu `▾` à côté du nom du board et raccourci `b` (filtre insensible à la casse et aux accents, création « Créer « … » »). Configuration → onglet **Boards** : « compte pour le streak », seuil de suivi, suppression. Chaque board a ses listes, son historique d'annulation et son fichier de sauvegarde. Le presse-papier traverse les boards (couper sur l'un, coller sur l'autre).
- **Sans migration** : le board existant devient le board principal (`main`) et garde ses clés `localStorage` et son handle IndexedDB ; rien n'est écrit tant qu'aucun second board n'existe, un retour à la v1.1.2 reste possible tel quel. Les nouveaux boards utilisent des clés suffixées et un index `kanbanjs:boards`, auto-réparé par scan des clés.
- **Gamification par board** : streak, compteur du jour, confettis et citations seulement sur les boards qui « comptent pour le streak » (le principal par défaut). La barre de progression reste partout. Le streak reste unique.
- **Suivi de projet** : `i` ou bouton 📅 → entrée de journal datée en haut de la note (annulée si rien n'est écrit) ; badge `⏳ N j` et carte atténuée quand la note n'a pas bougé depuis le seuil du board (14 j par défaut sur les nouveaux boards, désactivé sur le principal).
- **Fichier compagnon par board** : `AutoSave.switchBoard` vide l'écriture en attente **et en vol** dans le fichier du board quitté avant de changer ; le handle est lié à son board, aucun chemin ne peut écrire un board dans le fichier d'un autre.
- **Undo par board** : piles `sessionStorage` par board, purgées quand un autre onglet modifie le board, filet si le board actif change hors App ; les valeurs de streak capturées sont neutralisées quand le streak a bougé ailleurs. Quota `sessionStorage` saturé : purge des autres boards puis troncature, l'action n'est jamais bloquée.
- **Multi-onglets** : un board supprimé dans un autre onglet n'est jamais recréé en clé orpheline ; l'onglet resté dessus revient au principal avec un message.
- **Correctifs hors chantier** :
  - **Duplication (`c`)** : la copie s'affichait sans note ni tags, puis les **effaçait en base** au premier aller-retour ≡ ou `+ tag` (même famille que C7 en v1.1.1, oubliée ici).
  - **`autoResize`** : l'épaisseur des bordures (`border-box`) était oubliée — barre de défilement parasite dans toutes les notes.
  - **`saveNotes`** n'écrit plus une note inchangée (instantanés undo parasites à chaque ouverture/fermeture).
- **Méthode** : livré en 5 étapes (`DB`, `AutoSave`, `App`, interface, suivi de projet), chacune **relue en contexte neuf** par un sous-agent en lecture seule avant commit. Les relectures ont remonté **39 constats**, tous corrigés (deux reportés d’une étape à la suivante, puis corrigés), dont plusieurs pertes de données possibles (écriture d'un board dans le fichier d'un autre, index corrompu faisant disparaître des boards, undo restaurant un état périmé, quota bloquant toutes les actions). Chaque correctif a été validé par **mutation inverse** ; plusieurs tests qui passaient sur du code faux ont été rendus discriminants. Rendu, focus et ARIA vérifiés sur la vraie page (Brave headless, profil jetable).
- **Audit d'ensemble avant publication** : une relecture du diff cumulé v1.1.2 → v1.2.0 (personne ne l'avait relu d'un seul tenant) et 25 scénarios en navigateur réel (mise à jour v1.1.2 → v1.2.0 puis retour arrière sur le même profil, démarrage à froid sur un board secondaire, deux vrais onglets, vrai glisser-déposer, thème sombre). Correctifs :
  - **Bandeau « Vos données ont disparu — Restaurer ? »** : la décision est désormais **persistée** avec le handle (IndexedDB). Avant, un aller-retour de board — ou, déjà en v1.1.2, une modification suivie d'un rechargement — faisait disparaître le bandeau, et la sauvegarde suivante **écrasait le fichier, seule copie restante**. La puce `💾 Réactiver` ne lève plus ce bandeau (même défaut, pré-existant).
  - **Décomposition IA** terminée après un changement de board : résultat ignoré avec un message, au lieu de remplacer la liste « IA … » du board d'arrivée.
  - **Changement de board** : l'écran bascule avant le rattachement du fichier compagnon (lecture IndexedDB jusqu'à 8 s), plus de fenêtre où l'ancien board restait affiché sur les données du nouveau.
  - **Faux « Recharger ? »** au retour sur un board dont le fichier a seulement été « touché » par un client de synchronisation : le contenu est comparé, comme en session (régression de la correction v1.1.2).
  - **« Ouvrir un fichier existant… »** pendant qu'un autre onglet supprime le board : rien n'est importé dans le board principal, son fichier n'est pas remplacé.
  - **Réglages modifiés dans un autre onglet** (streak, seuil) : appliqués sans recharger, piles undo neutralisées.
  - **Synchro en attente puis changement de board** : n'efface plus l'historique d'annulation du board d'arrivée.
  - Valeurs de streak neutralisées dans les piles undo : JSON valide (un retour à la v1.1.2 ne remet plus le streak à zéro) ; import sur un board secondaire : jamais de réécriture du streak global ; service worker : `index.html` mis en cache en contournant le cache HTTP (`cache: 'reload'`).
- **Tests** : 76 → 229 ; scénarios navigateur réel : 25/25.
- **Documentation** : historique déplacé d'`AGENTS.md` vers `CHANGELOG.md` ; roadmap réordonnée (traduction, mode focus, minuteur, son, LM Studio, checklist de distribution) ; référence technique **Multiboard** dans `AGENTS.md`.
- **Service worker** : cache `kanbanjs-v6` → `v7`. `VERSION` 1.2.0.

## 2026-09-20 — Détection en session et dette technique (v1.1.2)

- **Détection d'un fichier modifié pendant la session** : nouvelle méthode `AutoSave.checkExternal()`, appelée toutes les 60 s (`POLL_MS`, suspendue quand l'onglet est caché) et au retour de visibilité. Elle compare `getFile().lastModified` à `lastWriteAt` et, si le fichier est plus récent, affiche le bandeau « Recharger ? » — les écritures étant alors suspendues, la version externe n'est jamais écrasée. Lève l'ancienne limite « détection seulement au boot ou au clic Réactiver ».
- **Debounce de `refreshHeaderInfo()`** : la fonction n'est plus qu'un planificateur (50 ms), le rendu est fait par `_applyHeaderInfo()`. Les 16 points d'appel après mutation ne déclenchent plus autant de lectures de `getLists()` + `getAllCards()` + streak. Point d'attention signalé dans la roadmap depuis les débuts du projet. Limite assumée : un aller-retour `100 % → < 100 % → 100 %` en moins de 50 ms (auto-répétition clavier) peut faire manquer les confettis.
- **`DB.onSave` multi-abonnés** : le callback unique (`_onSave = cb`) devient un tableau, et `onSave(cb)` **renvoie une fonction de désabonnement**. Jusqu'ici un second abonné écrasait silencieusement le premier — le jour où un module autre qu'`AutoSave` s'abonnait, la sauvegarde automatique cessait de fonctionner sans le moindre signe.
- **Abandon de `companion.html`** (roadmap #7) : décision actée — le mobile n'est pas le lieu de l'exécution, KanbanJS reste une expérience desktop, et le besoin de capture rapide depuis le téléphone ne s'est pas fait sentir. `companion.md` est conservé comme trace de la réflexion.
- **Correctifs issus d'une relecture en contexte neuf (même jour)** :
  - **Régression visible** : `refreshHeaderInfo()` n'étant plus `async`, les `await` des appelants ne garantissaient plus le rendu. Conséquence mesurée : après une restauration, le message « Board restauré. » était effacé ~50 ms plus tard par le rendu différé de `_rebuild()`. La fonction renvoie désormais **une promesse résolue après le rendu** (file `_headerWaiters`), ce qui restaure le contrat des 16 appelants sans supprimer la coalescence.
  - **Course `checkExternal()` / `flush()`** : la garde `_flushing` n'était lue qu'à l'entrée, avant deux `await`. Un flush démarré pendant la lecture du fichier pouvait écraser la version externe *puis* voir le bandeau « a été modifié ailleurs » s'afficher à tort. Les gardes sont maintenant revérifiées après l'`await` et la comparaison refaite sur `_lastWriteAt` à jour.
  - **Faux positif bloquant** : la détection était purement temporelle. Un client de synchronisation qui « touche » le fichier sans en changer le contenu posait le bandeau et **suspendait toutes les écritures** jusqu'à décision. Le contenu est désormais comparé (`file.text()` vs `DB.exportJSON()`) avant de lever le bandeau.
  - **`setInterval` de polling** déplacé après la sonde `available()` : il ne tourne plus inutilement sur Safari/Firefox, où il ne pouvait rien faire.
- **Tests** : 68 → 76 assertions (en-tête rendu, notification **et désabonnement** des abonnés `onSave`, 4 cas de détection en session dont le fichier inchangé et le contenu identique à mtime plus récent, message de restauration survivant au rendu différé). Validation par **mutation inverse** : sur le code non corrigé, la suite échoue exactement aux assertions concernées (`compteur = 10`, `detected=false`, et le message remplacé par « 7 listes · 0/15 cartes »).
- **Service worker** : cache `kanbanjs-v5` → `v6`. `VERSION` 1.1.2.

## 2026-09-20 — Repli pour les navigateurs sans File System Access API (v1.1.1)

- **Sonde de capacité** : `AutoSave.capability()` remplace le booléen `available()` et renvoie `ok` | `insecure` | `no-picker` | `no-idb`. Toujours de la détection de capacité, **jamais de sniffing d'UA** — décisif ici, car **Brave se présente comme Chrome** : un test d'UA aurait annoncé « supporté » puis planté à l'appel réel. `available()` est conservé (`capability() === 'ok'`).
- **Messages ciblés** : fini « Indisponible sur ce navigateur ». Un message par cause. Sur Brave (drapeau `navigator.brave.isBrave`, utilisé **uniquement** pour le texte du message), il indique d'activer « File System Access API » dans `brave://flags`. `autosaveUnsupportedMessage(cap, brave)` est une fonction pure, testée.
- **Repli export manuel** : quand l'API est absente (`unsupported`) ou non configurée (`idle`), que le board a déjà été modifié (`DB.hasState()`) et que le dernier export — ou `firstSeen` — date de ≥ 7 jours, une puce `⬇ Sauvegarder` apparaît dans le header. Elle déclenche `exportBoard()` (le téléchargement extrait du handler `#exportBtn`) et disparaît une fois l'export fait. Rappel **persistant**, sans snooze.
- **Onglet Données** : ligne « Dernier export : il y a N jours » + bouton « Télécharger une sauvegarde (JSON) », quel que soit le navigateur.
- **`DB`** : `hasState()` (« `kanbanjs:state` a déjà été écrit »), `getBackupInfo()`, `setBackupInfo()`. Nouvelle clé `kanbanjs:backup` (`{ firstSeen, lastExport }`).
- **Pourquoi ça compte** : Safari et Firefox n'implémentent pas la File System Access API, et Brave la désactive par défaut. Ces utilisateurs n'avaient jusqu'ici aucun filet.
- **Tests** : 50 → 68 assertions (sonde de capacité ; 4 cas de déclenchement du rappel — seuil, export récent, installation récente, veille du seuil ; 3 messages ciblés ; persistance des infos de sauvegarde ; ordre des cartes après rebuild et après reload ; copie de secours réparée ; import invalide rejeté sans effet ; reset purgeant le rappel ; collage préservant notes et tags). Vérifications hors suite : chemin non supporté rejoué en masquant `showSaveFilePicker` **avant** le boot (message Brave rendu, boutons désactivés), rappel d'export 7/7 dans un vrai DOM, **suite complète sur un `index.html` où l'API est masquée** (preuve que la sonde ne dépend pas du navigateur), et **drag lâché hors zone** (le correctif C6 ci-dessous, non testable dans l'iframe masquée du harnais car il exige une mise en page réelle). Chaque correctif a été validé par **mutation inverse** : la suite échoue sur le code non corrigé, aux assertions concernées.
- **Correctifs hors périmètre du diff, trouvés par la même relecture** :
  - **Drag lâché hors de toute zone (C6)** : le DOM était réordonné — la carte était posée à l'emplacement du placeholder — sans aucune écriture en base, et l'ordre se « corrigeait » silencieusement au rechargement suivant. `_origParent`/`_origAfter` étaient capturés au départ du drag mais jamais utilisés ; ils le sont désormais via `restoreOrigin()`. En mode copie (Ctrl), un drop invalide ne déplace plus l'original (il était repositionné par `ph.replaceWith(ghostEl)` alors que, en copie, l'original n'a jamais quitté le DOM).
  - **Coller ou Ctrl+drag d'une carte (C7)** : la carte affichée était construite à partir de l'objet d'**avant** `updateCardNotes`/`updateCardTags` — donc sans pastille de notes ni chips de tags. Plus grave, la première interaction ensuite réécrivait la base depuis ce DOM périmé : `+ tag` remplaçait tous les tags par le seul nouveau, et ouvrir puis refermer l'éditeur de notes écrivait `""`. La carte rendue est maintenant le clone à jour.
- **Correctifs issus d'une relecture en contexte neuf (module `DB` + diff v1.1.1)** :
  - **Ordre des cartes perdu au rechargement — majeur, pré-existant** : `_rebuild()` rendait les cartes dans l'ordre du tableau `_state.cards`, pas trié par `position`, alors que `moveCard()` ne réordonne pas ce tableau (il ne réécrit que les `position`) et que `getCards()` trie correctement. Conséquence : après un glisser-déposer dans une même liste, l'ordre manuel revenait à l'ordre de création à chaque chargement de page, undo/redo, import, reset ou sync. Seul `_rebuild` était fautif — les trois autres appelants de `buildList()` passent par `DB.getCards()`.
  - **`_isValidState()` durci** : les éléments `null` ou non-objet de `lists`/`cards` sont désormais rejetés. Un import malformé (ex. `lists:[null]`) était persisté **avant** le rendu, puis faisait planter `_rebuild()` au boot suivant (écran de crash recovery, board inaccessible). Il est maintenant refusé avant toute écriture, et `_load()` bascule sur la copie de secours au lieu de planter.
  - **Self-heal complet** : quand la clé principale **et** la copie étaient illisibles, la copie restait cassée — le self-heal restait indisponible jusqu'à la première mutation réelle. Les deux clés sont maintenant réparées dans le même passage.
  - **`getBackupInfo()`** : une valeur non-objet (`"[]"`, `42`, `true`) renvoyait une primitive, et l'init de `firstSeen` écrivait alors dans le vide (rappel muet). Validation de forme ajoutée.
  - **`backupInfoText()`** : plus de « NaN jours » sur une date invalide.
  - **`reset()`** purge aussi `kanbanjs:backup` — le rappel ne réapparaît plus juste après une remise à zéro.
  - **`onChange`** : `refreshAutoSaveUI()` étant devenue asynchrone, l'abonnement est enveloppé d'un `.catch()` (pas de rejet non géré).
  - **Test de sonde corrigé** : l'assertion `cap === 'ok'` faisait échouer la suite sur Brave, Firefox et Safari — précisément les navigateurs que ce repli cible. Elle vérifie maintenant l'appartenance à l'ensemble des valeurs connues.
- **Service worker** : cache `kanbanjs-v4` → `v5`. `VERSION` 1.1.1.

## 2026-09-20 — Fichier compagnon (v1.1.0)

- **Sauvegarde automatique** : nouveau module `AutoSave` (File System Access API). Configuration → onglet **Données** → « Choisir un fichier… » ; le board est écrit dans ce fichier après chaque modification (debounce 800 ms), au format `exportJSON()` (`{state, streak}`), **sans la clé API**. Flush aussi quand l'onglet passe en arrière-plan.
- **Handle persistant** : `FileSystemFileHandle` conservé en IndexedDB (base `kanbanjs`, store `handles`) avec `lastWriteAt`. Helpers IndexedDB réécrits avec cycle de vie de transaction (`oncomplete`/`onabort`), fermeture de connexion et **timeout de 8 s** pour qu'IndexedDB ne puisse jamais bloquer l'app. Un `_store` swappable rend le module testable sans IndexedDB.
- **Restauration en un clic** : au boot, si les données locales ont disparu (`DB.loadSource() === 'default'`) et qu'un handle existe → bandeau « Vos données ont disparu — Restaurer ? ». Un clic → `requestPermission` → lecture → `DB.importJSON()` → `_rebuild()`.
- **Multi-machine** : si le fichier est plus récent que la dernière écriture locale (`file.lastModified > lastWriteAt`), bandeau « Recharger ? ». Jamais d'écrasement silencieux dans un sens comme dans l'autre. Coût assumé : un clic « 💾 Réactiver » par session Chromium, la permission ne survivant pas.
- **États et UI** : `unsupported` | `idle` | `active` | `paused` | `error`. Puce discrète dans le header (`💾 Réactiver`, `⚠ Sauvegarde en échec`), invisible quand tout va bien.
- **`DB`** : `onSave(cb)` (notifié par `_save()`), `loadSource()` (`state` | `backup` | `default`), `_isValidState()` réutilisée par `importJSON`/`restoreJSON`.
- **Relecture en contexte neuf — 4 correctifs majeurs** : (1) tant qu'un bandeau est affiché, **plus aucune écriture** n'a lieu ; sans cela, la première frappe après un « Ignorer » écrasait la seule sauvegarde existante — et « Ignorer » demande désormais confirmation ; (2) `attach()` valide le fichier **avant** d'armer le handle : un fichier choisi par erreur n'est plus écrasé ; (3) compteur de génération `_gen` : `disable()` pendant une écriture laisse l'état `idle`, jamais `active` ; (4) drapeau `_dirty` : une écriture déclenchée pendant un flush en vol est rejouée au lieu d'être perdue.
- **Correctifs mineurs de la même relecture** : messages honnêtes selon l'état (`enable`/`attach`/« Sauvegarder maintenant ») ; permission `denied` distinguée et expliquée ; `_lastWriteAt` initialisé à `file.lastModified` après attachement (plus de bandeau « plus récent » fantôme) ; `init()` protégé par la génération ; `_persist()` non silencieux (avertissement console) ; le bandeau ne rogne plus le bas des listes (`calc(100vh - 133px)`) ; « Configurer l'IA » ouvre l'onglet Système ; la confirmation de reset mentionne le fichier qui sera écrasé ; `AutoSave._test` gaté derrière `?test` ; `refreshPending()` après une sync (cas d'un `localStorage.clear()` distant).
- **Tests** : 34 → 50 assertions (écriture différée, format sans clé API, restauration après perte, permission requise, désactivation, écriture bloquée pendant une décision, fichier invalide non armé, écriture concurrente non perdue, désactivation pendant une écriture). Hooks : `autoSaveMock`, `autoSaveFile`, `autoSaveState`, `autoSaveFlush`, `autoSaveRestore`, `autoSaveResume`, `autoSaveSetFile`, `autoSaveAttachWith`, `autoSaveBadFile`, `autoSaveDismiss`, `autoSaveFlushAsync`, `autoSaveFileName`, `autoSaveDisable`, `loadSource`. Campagnes de vérification hors suite (fichiers temporaires, non committés) : sync multi-onglets à deux iframes 4/4, UI du fichier compagnon 12/12.
- **Limites documentées** : écriture non atomique (`createWritable` tronque avant d'écrire) ; repli Safari/Firefox non implémenté ; conflits = dernier écrivain gagne ; détection externe fiable à ~1 s près (granularité du système de fichiers et dérive d'horloge entre machines).
- **Service worker** : cache bumpé `kanbanjs-v3` → `v4`. `VERSION` 1.1.0.

## 2026-09-20 — Robustesse de la persistance (v1.0.2)

- **Self-heal** : `DB._load()` restaure la copie de secours `kanbanjs:state.bak` (état précédent, écrite par `_save()`) si `kanbanjs:state` est illisible ou invalide, puis répare la clé principale. Si la copie est absente **ou corrompue**, elle est re-semée depuis l'état principal au chargement. Si les deux sont illisibles, la clé est réparée et un avertissement console est émis. Évite de perdre un board sur un JSON cassé.
- **Sync multi-onglets** : `window.addEventListener('storage', …)` → debounce 200 ms → `DB.reload()` (nouvelle méthode publique) puis `_rebuild()`. Les **données** sont rechargées immédiatement ; seule la **reconstruction du DOM** est différée tant qu'une édition est en cours (carte, notes, aperçu, renommage, formulaire, modale, recherche, cheatsheet). Sans cette séparation, le `blur` d'une édition sauvait un état périmé et écrasait le changement distant. Les piles undo/redo sont purgées à la synchro pour éviter qu'un `u` ne révoque un travail distant. Les changements d'un autre onglet apparaissent sans rechargement ; les conflits simultanés restent en « dernier écrivain gagne ». L'événement `storage` avec `key === null` (clear) est aussi traité.
- **Reset** : `DB.reset()` purge désormais `kanbanjs:state.bak`, pour que la copie de secours ne ressuscite pas un board volontairement effacé (contrat « toutes les données seront perdues »).
- **Stockage persistant** : `navigator.storage.persist()` demandé au premier `updateStreak()` (`requestPersist()`), pour limiter l'éviction implicite par le navigateur.
- **Abstraction** : la validation de structure (`board` + `lists[]` + `cards[]`) est centralisée dans `_isValidState()`, réutilisée par `_load()`, `importJSON()` et `restoreJSON()`.
- **Surface de test** : `window.__testAPI` n'est plus enregistré que si l'URL contient `?test` (inerte sur le site publié). Ferme la lecture/écriture à distance du board via une iframe sandboxée (`origin 'null'`).
- **Tests** : 4 tests ajoutés dans `test.html` (self-heal après corruption, reset purge la copie, copie corrompue re-semée) — 34/34. Hooks : `renameBoard`, `reload`, `corruptState`, `getBackup`, `corruptBackup`.
- **Service worker** : cache bumpé `kanbanjs-v2` → `v3`.

## 2026-09-20 — Correctif streak UTC (v1.0.1)

- **Bug corrigé** : le streak, le compteur du jour et les comparaisons « terminé aujourd'hui » utilisaient la date UTC. Une tâche terminée après minuit (heure locale, fuseau UTC+) comptait pour le jour précédent — un utilisateur nocturne pouvait perdre son streak malgré un travail quotidien.
- **Helpers** : `dateKey(date)` (clé `YYYY-MM-DD` locale via `getFullYear`/`getMonth`/`getDate`) et `isToday(date)` dans le scope App. Remplacent 6 usages de `toISOString().slice(0, 10)` (`refreshHeaderInfo`, `updateStreak`, `adjustStreak`, 3 comparaisons `doneAt`).
- **Tests** : 3 tests ajoutés dans `test.html` (0h30, 23h30, 1er janvier) — 30/30. Le test à 0h30 échoue sur l'ancien code, il détecte donc bien la régression.
- **Service worker** : cache bumpé `kanbanjs-v1` → `v2`.

## 2026-09-20 — Publication GitHub Pages (v1.0.0)

- **Renommage** : `kanban.html` → `index.html` (via `git mv`), refs mises à jour dans `test.html`, `AGENTS.md`, `companion.md`.
- **Service worker** : `sw.js` créé (cache-first, chemins relatifs `'./'`, guard non-GET), enregistré dans `index.html` en `https:` uniquement.
- **Fichiers de publication** : `README.md` (français), `LICENSE` (MIT, Laurent Hofer), `VERSION` (1.0.0), `.nojekyll`, `.gitignore`.
- **Purge d'historique** : les 7 fichiers de process (`01`–`05`, `audit.md`, `github.md`) retirés du disque, de l'index et de tout l'historique (`filter-branch`, 81 → 80 commits). Sauvegardes hors dépôt : `kanbanjs-process-backup.tar.gz`, `kanbanjs-prepublic.bundle`.
- **Déploiement** : repo public `damnscientist/KanbanJS`, branche `main`, GitHub Pages actif — https://damnscientist.github.io/KanbanJS/. Auth push en SSH (clé ed25519).
- **Tests** : 27/27 après renommage. Vérification du site réel : chargement, 6 listes, thème, service worker `active`, zéro erreur console.

## 2026-07-13 — Ollama, confettis, service worker

- **Support Ollama** : nouveau fournisseur `ollama` dans le `<select>`, endpoint par défaut `http://localhost:11434/v1/chat/completions`. La clé API est optionnelle (masquée dans la modale), le timeout passe à 5 min pour les modèles locaux. Parsing identique à OpenAI (format compatible).
- **Confettis** : ne se déclenchent plus sur les opérations passives (chargement initial, undo, import, rename). La transition `< 100% → 100%` est désormais détectée via `_prevPct` (comparaison avant/après dans `refreshHeaderInfo`). `_rebuild()` pose `_prevPct = 100` pour que la reconstruction DOM ne soit jamais traitée comme une complétion.
- **Service worker** : spécifications ajoutées (cache-first, ~35 lignes) pour le mode hors-ligne après la première visite. Implémenté le 2026-09-20 (`sw.js`).

## 2026-07-01 — Tags, markdown enrichi, templates first, accessibilité, robustesse
- **Tags** : champ `tags: []` sur les cartes, chips colorés par hash du nom, bouton `+ tag`, input inline (Enter ajoute, Escape annule), suppression par ✕. Raccourci `t`. Tags préservés au copier/coller/dupliquer/paste de liste. Recherchables via `Ctrl+K`. DB: `updateCardTags(id, tags)`.
- **Markdown enrichi** : `#`, `##`, `###` → titres (h3/h4/h5), `- item` → listes à puces, `1. item` → listes numérotées, `[texte](url)` → liens cliquables, `---` → ligne horizontale.
- **Templates first** : quand l'API n'est pas configurée, la modale Décomposer affiche directement une grille des 12 templates + lien `⚙ Configurer l'IA`. Le FAB est toujours `+` (plus jamais `!` rouge). L'IA devient un bonus, pas un prérequis.
- **Accessibilité** : `label[for]` sur tous les champs de formulaire, `aria-label` sur les 20+ boutons (icônes header, actions carte/liste, FAB). Mise à jour dynamique des aria-labels (collapse, notes, FAB, `_toggleAllNotes`).
- **Robustesse** : `QuotaExceededError` affiche une bannière persistante dans le header (disparaît automatiquement quand l'espace se libère). `_snapshot()` avertit via `flashMessage` quand l'undo est désactivé (board > 200 ko). `parseAITasks` gère les code fences (\`\`\`json).
- **Correctifs** : `_rebuild()` préserve `scrollLeft`. `_confettiDone` se reset toujours après les confettis (permet plusieurs célébrations). `fuzzyMatch` dédupliqué en helper partagé. Le collapse bouton met à jour son `title`/`aria-label` au toggle.
- **Tests** : `test.html` — 30 tests DB via iframe + postMessage (reset, listes, cartes, export/import, streak, dates locales). Lancement : `python3 -m http.server`, puis `http://localhost:8000/test.html`.
- **Fichier** : ~3900 → ~4200 lignes (+300)

## 2026-07-01 — Onboarding, streak robuste, notes markdown, UX polie
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

## 2026-07-01 — Suppression de la couche UX mobile
- **Décision stratégique** : un outil anti-procrastination ne doit pas se faire sur mobile. Le kanban complet est une expérience desktop. Pour la capture rapide, voir `companion.md`.
- **CSS** : suppression du bloc `@media (max-width: 640px)` (~185 lignes), des classes `.mobile-toolbar`, `.header-menu-overlay`, `.header-menu-btn`, de `@keyframes slideUp`, de `touch-action: none` sur `.card`/`.list-header`/`.dnd-ghost-list`, de `-webkit-user-select` et `-webkit-touch-callout`.
- **HTML** : suppression de `#mobileToolbar`, `#headerMenuOverlay`, `#headerMenuBtn`, `#searchMobileBtn`, des classes `.header-desktop-only` et `.header-menu-btn`, des attributs `enterkeyhint`/`inputmode`.
- **JS** : suppression de `isMobile()`, des handlers mobile (recherche, menu ⋯, toolbar contextuelle, `MutationObserver`, `visualViewport.resize`), de la détection double-tap sur les titres, de la branche touch du DnD (tap-long 200ms, edge-scroll, `_touchDrag`, `webkitUserSelect`). `selectCard()` simplifié (plus de branche `isMobile`). `DnD.start()` simplifié (comportement souris uniquement).
- **Fichier** : 4024 → 3438 lignes (-586, -14.5%)

## 2026-06-29 — Correctifs mobile (post-refonte)
- **`touch-action: auto` annulé** : la tentative de passer les `.card` et `.list-header` en `touch-action: auto` sur mobile cassait le DnD. Le `e.preventDefault()` appelé dans `beginCapture()` arrive 200ms après le `pointerdown` — trop tard, le navigateur a déjà pris la main sur le scroll. Le DnD repose sur `touch-action: none` CSS (le navigateur ne scroll jamais sur les cartes, le JS gère tout). Le tap-long à 200ms est conservé pour laisser un délai entre le touch et l'activation du drag.
- **FAB recouvert par la toolbar** : le FAB (`z-index: 5000`) était masqué par la toolbar mobile (`z-index: 8000`) quand une carte était sélectionnée. FAB → `z-index: 8100` sur mobile + `showToolbar()`/`hideToolbar()` repositionnent le FAB à `calc(84px + safe-area)` au-dessus de la toolbar.
- **Sélection de texte iPhone pendant le DnD** : Safari iOS a besoin du préfixe `-webkit-user-select: none` — le `user-select` standard est ignoré. Ajout du préfixe sur `.card`, `.card-text`, `.list-header`, `.list-title`, `header h1`, `.board-name`. Ajout de `-webkit-touch-callout: none` sur `.card` et `.list-header`. Le JS pose aussi `webkitUserSelect` dans `startDrag()`/`cleanup()`.
- **Textarea de notes invisible jusqu'à la frappe** : `autoResize()` mesurait un `scrollHeight` potentiellement nul quand le textarea venait de passer de `display:none` à `display:block`. Hardening : guard `offsetParent`, `Math.max(scrollHeight, rows*20, 28)`, `rows=2` sur le textarea notes. `toggleNotes()` enrobe `autoResize()` dans un double `requestAnimationFrame`. Nouveau handler `visualViewport.resize` pour les notes sur mobile.
- **Délai tap-long réduit** : 320ms → 200ms. Le délai reste nécessaire pour distinguer scroll et drag sur touch, mais 200ms est suffisant et nettement plus réactif.
- **Recherche dans liste repliée** : le `colBtn.click()` pour déplier était async — la classe `.collapsed` n'était pas retirée avant `selectCard()`, donc la carte était sélectionnée mais invisible. Retrait synchrone de `.collapsed` avant `selectCard()`, puis `colBtn.click()` pour persister.
- **Bouton "Ajouter une liste"** : n'avait pas `scroll-snap-align`, causant un scroll non fluide en fin de board. Ajout de `scroll-snap-align: start` sur `.add-list-btn` et `.add-list-form` dans la media query mobile.
- **Zoom + click parasite + sélection liste pendant le DnD** : iOS zoomait et synthétisait un `click` au lâcher car `preventDefault()` n'était pas appelé au bon moment. Trois `e.preventDefault()` ajoutés : immédiat sur `pointerdown` touch (bloque la synthèse du `click`), dans `onMove` (bloque le zoom), dans `onUp` (ceinture+bretelles).
- **Edge-scroll pendant le DnD (touch)** : scroll automatique du board quand le doigt approche des bords. Horizontal pour atteindre les listes adjacentes, vertical pour les cartes hors vue dans la liste survolée. Vitesse proportionnelle à la distance du bord (`requestAnimationFrame`). Utilise `scrollBy({ behavior:'auto' })` et désactive temporairement `scroll-snap-type` pour contourner les limitations iOS. Invalidation des caches `_zoneCache`/`_cardCache` sur changement de `scrollLeft`/`scrollTop`. Activé uniquement pour `pointerType === 'touch'`, desktop souris inchangé.

## 2026-06-29 — Refonte UX mobile
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

## 2026-06-29 — Templates de décomposition
- **12 templates intégrés** : corvées universelles (vaisselle, lessive, rangement, courrier, facture, mail, dossier, réunion, RDV médical, départ, boîte mail, fichiers bureau) avec micro-tâches pré-générées.
- **Matching fuzzy** : au clavier dans le textarea de décomposition, détection automatique du template le plus proche et affichage d'un bandeau `💡 Template suggéré : X — Utiliser ce template`.
- **Zero API** : les templates créent une liste nommée d'après le template en position 0, sans aucun appel réseau.
- `createTemplateList(template)` : création atomique liste + cartes via `DB.batch()`.
- CSS : bandeau `.template-banner` avec `color-mix(var(--accent) …)`, hover accent.

## 2026-06-29 — Correctifs templates
- **Bug — Double-clic bandeau template créait des listes dupliquées** : le handler de clic du `.template-banner` n'avait pas de guard anti-double-clic (contrairement au bouton de décomposition IA qui fait `btn.disabled = true`). Ajout d'une classe `.disabled` (`pointer-events: none`, `opacity: .5`) + `finally` pour réactiver.
- **Bug — Aucune gestion d'erreur sur clic template** : si `createTemplateList` levait une exception (ex: quota localStorage), `close()` n'était jamais appelé, la modale restait ouverte sans feedback. Ajout d'un `try/catch` avec `flashMessage` sur le `statusEl`.
- `open()` : reset défensif de la classe `disabled` à l'ouverture de la modale.
- **Tab active le template** : dans le textarea de décomposition, `Tab` focus le template suggéré (tabindex dynamique), `Enter`/`Space` l'active. Un 2e `Tab` amène au bouton "Décomposer" pour l'IA.

## 2026-06-29 — Feedback de progression (streak)
- **Streak compteur** : `kanbanjs:streak` dans localStorage (`{ streak, lastDate, todayCount, todayDate }`). Incrémenté à chaque complétion (drag vers liste Done ou toggle liste terminée). Reset à 1 si un jour est sauté.
- **Badge streak** : badge `🔥 N` dans le header, visible quand streak ≥ 2. État "paused" (opacité réduite) quand le jour en cours n'a pas encore de complétion, "actif" sinon.
- **Compteur du jour** : `· X aujourd'hui` en couleur accent dans le header info.
- **Animation pulse** : la barre de progression pulse (brightness) à chaque nouvelle complétion.
- Survit au reset/import (clé séparée de `kanbanjs:state`), auto-correction naturelle en sautant un jour.

## 2026-06-29 — Robustesse undo/IA/paste
- **Snapshot atomique sur paste carte** : `_paste()` wrappe `createCard` + `updateCardNotes` dans un seul `mutate()`. Avant, l'undo après un collage avec notes perdait les notes.
- **Suppression de `location.reload()`** : undo, redo, reset et import reconstruisent le DOM via `_rebuild()` au lieu de recharger la page. Plus de flash visuel, scroll et sélection préservés.
- **`max_tokens` explicite** : ajout de `max_tokens: 2048` pour OpenAI et `maxOutputTokens: 2048` pour Google Gemini. Anthropic avait déjà `max_tokens: 4096`. Évite les troncatures de réponse JSON sur les petits modèles.
- Extraction de `_rebuild()` : la reconstruction DOM (board name, listes, cartes, widget) est extraite de `init()` et réutilisée par undo/redo/reset/import.

## 2026-06-21 — Bugs finaux
- **Bug 1 — `getAfter` cartes recevait `x` au lieu de `y`** : le moteur DnD appelle `cfg.getAfter(container, x, y, ghostEl)` mais le callback carte déclarait `(container, y)`, donc `y` recevait la valeur de `x`. Corrigé en `(container, x, y)`.
- **Bug 2 — Cache `_cardCache` jamais invalidé entre listes** : le cache des rects des cartes était construit pour une liste et réutilisé tel quel quand le drag survolait une autre liste. Corrigé en keyant le cache par container (`_cardCache._container`).
- **Bug 3 — Double snapshot undo sur « Sauver »** : blur + click déclenchaient `saveEdit()` deux fois. Corrigé avec un guard `if (!wrap.classList.contains('editing')) return;` en tête de `saveEdit`.
- **Bug 4 — « Annuler » sauvegardait quand même** : `exitEdit()` retirait `.editing`, puis le blur appelait `saveEdit()`. Le guard du Bug 3 corrige aussi ce bug.
- **Bug 7 — `importFile.value = ''` hors callback** : reset de l'input synchrone avant `onload`. Déplacé dans un `finally` du `onload` et dans `onerror`.
- **Bug 8 — `_selList` orphelin après suppression de liste** : le handler appelait `selectCard(null)` mais ne désélectionnait pas `_selList`. Remplacé par `clearSelection()`.
- **Bug 12 — Recherche sélectionnait carte dans liste repliée** : cliquer sur un résultat de recherche dans une liste collapsed sélectionnait la carte sans la rendre visible. Ajout d'un dépliage automatique.

## 2026-06-21 — Architecture
- **Frontières DB** : ajout de `DB.exportJSON()`, `DB.importJSON(json)`, `DB.reset()`, `DB.restoreJSON(json)`, `DB.getAllCards()`. Plus aucun accès à `localStorage` / `DB.KEY` hors du module DB.
- **DOM source de vérité** : `refreshHeaderInfo()` et `refreshCountBadge()` lisent les données via DB (pas `querySelectorAll`).
- **Init parallèle** : `init()` utilise `DB.getAllCards()` + `for...of` au lieu d'un `forEach` avec `await`.
- **buildList/buildCard synchrones** : les cartes sont pré-fetchées par l'appelant.
- **Presse-papier encapsulé** : `Clipboard` objet avec méthodes explicites, plus de variable globale `_clipboard`.
- **mutate helper** : `mutate(fn)` = `_snapshot()` + `fn()`. Remplace ~15 appels `_snapshot()` raw.
- **Theme API** : `const Theme = window.__themeAPI` capturé au début de App.
- **removeListDom(listId)** : fonction partagée pour retirer une liste du DOM par ID.
- `_rebuild()` reconstruit tout le DOM à partir de l'état DB. Utilisée par undo, redo, reset, import et `init()`. Remplace tous les anciens `location.reload()`.

## 2026-06-21 — Qualité/DRY
- `toggleForm(btn, form, input, open)` : helper pour le pattern open/close de formulaire.
- `registerOverlay(overlay, closeFn)` : helper pour fermeture overlay (clic extérieur + Escape).
- `formatDoneAt(iso)` : formatage de date unifié ("Terminé le JJ/MM/AAAA à HH:MM").
- `extractCardData(el)` / `extractListData(el)` : extraction des données depuis le DOM.

## 2026-06-21 — Perf/robustesse
- Cache `getBoundingClientRect()` DnD : rects listes et cartes capturés une fois au début du drag.
- `DB.batch(fn)` : suspend les écritures localStorage le temps de la batch, persiste en une fois.
- Ghost DnD liste simplifié : `ghostBuilder` crée un clone minimal (header sans boutons).
- Import quota pre-check : alerte si > 4.5 MB.
- `AbortController` cleanup : `clearTimeout` dans `finally` du fetch.
- Taille snapshot limitée à 200 KB.
- Bugfix : `await` dans `forEach` → `for...of`.
- Bugfix : double `DB.exportJSON()` dans `_snapshot()`.

## 2026-06-21 — Visuel (CSS)
- Couleurs hardcodées (`rgba(232,168,90,…)`) remplacées par `color-mix(in srgb, var(--accent) X%, transparent)` (10 règles CSS).
- `.add-list-btn` : suppression des 16 lignes de surcharge par thème, `:hover` unique avec `color-mix`.
- Debug panel : couleurs → variables de thème (`--surface-2`, `--text`, `--accent`, `--danger`).
- Ghost DnD : `box-shadow: 0 8px 32px rgba(0,0,0,.6)` → `box-shadow: var(--shadow)`.
- `.cfg-status.success` : `#2ecc71` → `var(--accent)`.
- Responsive : `.cheatsheet-intro` masqué sur petit écran
- États visuels : `:focus-visible`, `:active`, `:disabled` sur boutons, `cursor: default` sur `.cheatsheet .row`.
- Fallback `rgba(var(--accent-rgb), .X)` avant chaque `color-mix()` pour les WIP warnings.

## 2026-06-21 — Bugs
- Guards null DOM sur `querySelector` dans `cutCard`, `copyCard`, `cutList`, `copyList` et la recherche.
- `DB.toggleListDone(id, doneAt)` : second paramètre optionnel, supprime l'appel redondant à `setCardsDoneAt()`.
- `reader.onerror` ajouté sur l'import JSON.
- Centralisation Escape : le handler dédié de la cheatsheet est supprimé, géré par `BINDINGS['Escape']`.
- `_selCard` orphelin : guard dans `delListBtn` (si carte sélectionnée appartient à la liste supprimée).
- Ordre DnD DOM/DB : `wrap.dataset.listId = toListId` après `await mutate(() => DB.moveCard(...))`.

## 2026-06-20
- Raccourcis clavier vim-like (~160 LOC) : navigation h/l/j/k, actions n/r/e/x/y/p + majuscules, focus 1-9, cheatsheet `?`.
- Presse-papier unifié, sélection visuelle (bordure accent, actions toujours visibles).
- Cheatsheet toggleable (overlay, deux colonnes carte/liste, principes kanban, responsive).
- Indicateur de progression (barre + pourcentage dans le header).
- WIP warning (bordure + badge accent quand liste In Progress ≥ 3 cartes).
- Repli des listes (bouton ▾/▸, état persisté).
- Liste "Waiting" ajoutée par défaut (kanban canonique).
- Undo/Redo (u/Ctrl+Y, 30 snapshots sessionStorage).
- Bugfixes : bleed-through clavier sur overlay recherche, Escape cheatsheet, éditeur bloqué si texte vide, import quota, _selCard orphelin, contextmenu DnD.

## 2026-06-19
- `DB.setCardsDoneAt(listId, doneAt)` : méthode batch pour les écritures groupées.
- `_save()` wrappé dans try/catch (QuotaExceededError → alert).
- `parseAITasks` : détection des guillemets dans le compteur de crochets.
- Prompt OpenAI : ajout d'un message `role: 'user'` (compatibilité élargie).
- `flashMessage(el, msg, cls, ms)` : fonction partagée.
- `close()` de la modale décomposition : nettoyage du timer de statut.
- `maxlength="200"` sur les inputs de nom (board et listes).

## 2026-06-17
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
