# Mobile Gaming Check

Diagnostic **100 % front-end** pour le gaming mobile : réseau, performances CPU/GPU, surchauffe et compatibilité avec les jeux (Free Fire en priorité), puis des conseils personnalisés.

Aucun compte, aucune installation, aucune donnée envoyée à un serveur. **Interface bilingue FR/EN**, thème sombre par défaut (et clair au choix).

---

## Démarrage

Le site utilise des **modules ES** et `fetch` : il ne fonctionne pas en `file://`. Lancez le petit serveur inclus (Node ≥ 18, zéro dépendance) :

```powershell
npm run dev
```

→ http://localhost:5173

Pour les tests unitaires :

```powershell
npm install   # une seule fois (Vitest)
npm test
```

Déploiement : copiez simplement le dossier sur n'importe quel hébergement statique (GitHub Pages, Netlify, Vercel). Aucun build n'est nécessaire.

---

## Ce que fait l'application

| Module | Mesures | Détails |
|---|---|---|
| **Réseau** | Ping, gigue, débit descendant et montant | Endpoints publics Cloudflare (`__down` / `__up`) avec CORS ouvert. Repli sur `navigator.connection` si les appels échouent. |
| **Performances** | Score CPU, score GPU, RAM, score global | Charge CPU calibrée (médiane de 3 passes), scène WebGL 2.0→1.0 sur 5 s, `navigator.deviceMemory`. |
| **Jeux** | Verdict : jouable / réglages bas / non recommandé | Comparaison RAM + CPU + GPU + version d'OS vs configuration min/recommandée. |
| **Bridage** | Écart de performance froid → chaud | Alerte au-delà de **20 %** de baisse, vigilance entre 10 et 20 %. |
| **Conseils** | 15+ règles déclaratives | Minimum garanti de 5 conseils, export presse-papier / partage natif. |

Le score global suit la pondération du cahier des charges — **CPU 40 % / GPU 40 % / RAM 20 %** — et renormalise automatiquement si une mesure est indisponible (ex. `deviceMemory` absent sur Safari iOS).

---

## Structure

```
index.html              Shell HTML (6 vues, tabbar, header)
css/
  tokens.css            Variables, reset, typographie, thèmes clair/sombre
  layout.css            Header, tabbar, toasts, cadre responsive
  components.css        Boutons, cartes, jauges, badges, icônes, alertes
  screens.css           Mise en page spécifique aux écrans
  style.css             Point d'entrée (@import)
js/
  main.js               Bootstrap : thème, langue, routes, hydratation
  core/
    dom.js              Helpers DOM, toasts, annonces lecteur d'écran
    format.js           Formatage localisé (unités, durées, dates)
    storage.js          Wrapper localStorage versionné et quota-safe
    state.js            Store central + pub/sub + persistance
    i18n.js             t(), interpolation, application au DOM
    router.js           Routage par hash + navigation clavier des onglets
    progress.js         Barre de progression annulable
    global-progress.js  Progression flottante « Tout tester »
    run-all.js          Orchestrateur « Tout tester »
  modules/
    network.js          Latence, gigue, débit (budgets de temps par palier)
    cpu-bench.js        Charge calibrée, warm-up, meilleure passe
    gpu-bench.js        Scène WebGL, FPS moyen sur 5 s
    device.js           OS, navigateur, écran, cœurs, RAM, batterie
    scoring.js          Pondération 40/40/20 et renormalisation
    thermal.js          Indice de performance CPU+GPU, seuils de bridage
    games.js            Chargement du catalogue, comparaison, verdicts
    advisor.js          Moteur de règles → conseils
  ui/
    icons.js            Jeu d'icônes SVG inline (aucune police, aucune requête)
    components.js       Jauges, compteurs, alertes, barres comparatives
    screen-*.js         Les 6 écrans
  i18n/
    fr.js               Dictionnaire de référence
    en.js               Dictionnaire anglais (parité vérifiée par les tests)
data/games.json         Catalogue de jeux éditable sans rebuild
assets/                 Logo et favicon SVG
tests/                  Tests Vitest (112 tests)
tools/serve.mjs         Serveur statique local
tools/smoke*.mjs        Scripts de vérification navigateur (dev)
```

---

## Fonctionnement des mesures

### Réseau

- **Ping** : 12 requêtes `__down?bytes=0`, médiane après écarte du premier échantillon (handshake TLS/TCP).
- **Gigue** : écart-type des RTT.
- **Débit** : lecture en streaming, chronométrée sur les octets réellement reçus. Chaque palier possède un **budget de 9 s** : sur une connexion lente, la mesure s'arrête et conserve ce qui est arrivé au lieu de bloquer. L'envoi adapte la taille des payloads au débit mesuré.
- Budget data annoncé : **~10 Mo**, avec avertissement avant lancement.

### CPU

Le nombre d'itérations est **calibré** à chaque exécution pour viser ~320 ms par passe, après une passe de chauffe. Sans cela, une passe de 20 ms sur desktop était dominée par la compilation JIT et deux exécutions consécutives pouvaient différer de 50 %.

Le score se base sur la **meilleure passe** : une interférence (autre onglet, tâche système) ne peut que ralentir une mesure, jamais l'accélérer.

**Référence** : `CPU_REFERENCE_ITER_PER_SEC = 35_000_000` (téléphone milieu de gamme de 2023). Un desktop récent dépasse 120 M it/s et sature donc à 100/100 — c'est volontaire, ce benchmark sert à classer des téléphones. Modifiez cette constante dans [cpu-bench.js](js/modules/cpu-bench.js) pour recalibrer.

### GPU

Scène de 9 000 particules avec shader procédural, contexte WebGL2 puis WebGL1. Si le contexte est refusé, le score GPU est simplement omis et le score global se renormalise sur CPU + RAM.

### Bridage

L'indice compare **CPU et GPU uniquement** (moyenne 50/50) : la RAM ne varie pas avec la chaleur et diluerait le signal. Seuils : `< 10 %` stable, `10–20 %` vigilance, `≥ 20 %` bridage.

---

## Compatibilité

| Navigateur | Support |
|---|---|
| Chrome / Edge Android | Complet |
| Samsung Internet | Complet |
| Firefox Android | Complet (`navigator.connection` partiel) |
| Safari iOS | Complet, avec `deviceMemory` absent → RAM affichée « inconnue » |

Résolution minimale testée : **360 × 640 px**. Toutes les cibles tactiles font au moins 48 px (hors liens secondaires du header).

Les résultats (réseau, perfs, bridage, historique) sont conservés en `localStorage` et restaurés au rechargement.

---

## Personnalisation

| Quoi | Où |
|---|---|
| Ajouter un jeu | [data/games.json](data/games.json) — ajoutez une entrée avec `id`, `name`, `icon`, `min`/`rec`, `presets`, `notes` |
| Ajuster les seuils réseau | `THRESHOLDS` dans [network.js](js/modules/network.js) |
| Recalibrer le CPU | `CPU_REFERENCE_ITER_PER_SEC` dans [cpu-bench.js](js/modules/cpu-bench.js) |
| Ajouter un conseil | `RULES` dans [advisor.js](js/modules/advisor.js) + clés dans les deux dictionnaires |
| Changer les couleurs | [tokens.css](css/tokens.css) |

---

## Tests

```powershell
npm test
```

112 tests couvrent la logique pure : seuils réseau aux bornes exactes, pondération du score, calibration CPU, seuils de bridage, verdicts de compatibilité, déclenchement des règles de conseil, et **parité des dictionnaires FR/EN** (aucune clé manquante, aucun texte vide).

---

## Limites connues

- Aucune API web ne donne la température interne : la surchauffe est déduite du bridage.
- Les scores CPU/GPU sont **relatifs** (ce benchmark), pas absolus.
- Cloudflare peut être bloqué par un pare-feu ou un bloqueur de publicité : un repli est prévu, jamais de crash.
- Le trafic réseau et le nombre de cœurs influencent la mesure ; l'interface le rappelle avant chaque test.
