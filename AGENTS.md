# nantes-bg1-dispo — notes projet

Pages statiques pour l'équipe **Nantes BG 1** (club ASCBG) : disponibilités des joueurs
en championnat départemental DD1.1 + organisation des tournois régionaux.

- Repo : `github.com/Ludoo-73/nantes-bg1-dispo` (branche `main`)
- **Déploiement : GitHub Pages** depuis `main` → `https://ludoo-73.github.io/nantes-bg1-dispo/`
  Un simple `git push` suffit à publier. Pas de build, pas de bundler.

## Fichiers — où modifier quoi

- `index.html` : **page live**, source de vérité. Dispo joueurs DD1.1 avec synchro
  Firestore temps réel, export agenda (.ics / Google Calendar), lien vers tournois.
- `tournois.html` : dispo pour les tournois régionaux de la saison (Firestore aussi).
- `whatsapp.html` : **variante legacy sans Firebase** — les réponses circulent en
  messages `BG1:…` copiés/collés dans WhatsApp, import manuel côté capitaine.
  Gardée comme fallback ; ne pas lui ajouter de features.
- `firebase.html` : brouillon intermédiaire (index.html sans les features agenda/liens).
  Candidat à suppression — vérifier avant de s'en servir.

## Stack & conventions

- HTML/CSS/JS vanilla monolithique (style + script inline dans chaque page), français.
- Mobile-first : `env(safe-area-inset-*)`, boutons ≥ 44px, dark mode via
  `prefers-color-scheme` + `:root[data-theme]`, variables CSS `--bg/--card/--ink/…`.
- Fonte titres : Bricolage Grotesque (Google Fonts).
- Firebase compat **10.14.1** (app + firestore), projet `nantes-bg1-dispo`.
  Collections : `dispo` (un doc par joueur), `dispotr` (idem tournois),
  `meta/reports` (matchs reportés).
  La config `firebaseConfig` en clair est normale (clé client publique).
- **Règles Firestore déjà strictes** (console, onglet Règles) : `dispo`/`dispotr`
  n'acceptent que les joueurs whitelistés, clés de dates whitelistées et valeurs
  `ok|maybe|no` ; `meta` n'accepte que le doc `reports`. **Toute nouvelle
  journée/tournoi/joueur doit être ajoutée dans ces listes** sinon l'écriture
  est refusée.
- **Clé API restreinte** (Cloud Console → Identifiants) : referrers HTTP
  `ludoo-73.github.io/*`, `127.0.0.1/*`, `localhost/*` + 25 APIs.
  ⚠ la synchro Firebase ne marche PAS en `file://` (pas de referrer) —
  voir "Test local".
- localStorage : `bg1data` (réponses), `bg1reports` (reports), `bg1me` (joueur courant).

## Pièges

- **La synchro Firestore écrit en prod en direct** — tester les modifs avec prudence
  (les suppressions dans `dispo`/`meta` sont irréversibles).
- Les `firebase.html`/`whatsapp.html` dérivent d'index : une feature ajoutée à
  `index.html` ne va pas automatiquement dans les variantes.
- Saison courante codée en dur (« 2026-2027 », liste des journées) — à revoir
  à chaque nouvelle saison.

## Test local

Ouvrir les fichiers en `file://` suffit pour l'UI (localStorage seul) — mais
**la synchro Firestore exige un referrer** : servir via
`python3 -m http.server` et ouvrir `http://localhost:8000` ou
`http://127.0.0.1:8000` (referrers autorisés sur la clé API).
Le partage de données en direct nécessite la console Firebase pour vérifier.
