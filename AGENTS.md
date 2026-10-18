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
  à chaque nouvelle saison (voir checklist ci-dessous).

## Nouvelle saison — checklist

Les règles Firestore **whitelistent joueurs + dates** : si on met à jour les
pages sans les règles (ou l'inverse), les écritures sont refusées. Faire les
deux dans le même mouvement :

1. **Code** :
   - `index.html` : `var P` (liste joueurs, ~l.134), `var D` (journées DD1.1,
     ~l.136 — les `id` = clés utilisées en base), sous-titre « saison 20XX-20XX ».
   - `tournois.html` : `var P` (~l.110) + liste des tournois (`id:"trN-ville"`,
     `jours` — les clés en base sont `id:index`, ex `tr2-gue:0`).
2. **Règles Firestore** (console → Firestore → Règles) :
   - `dispo`/`dispotr` : liste `player in [...]` = nouveau `P`.
   - `dispo` : `keys().hasOnly([...])` = nouveaux `id` de journées.
   - `dispotr` : `keys().hasOnly([...])` = nouvelles clés `trX:idx`.
3. **Purge des données de la saison passée** : le bouton reset de la page ne
   supprime que les docs des joueurs du `P` courant — si `P` change, purger les
   anciens docs via la console Firestore (Data → collections `dispo`, `dispotr`,
   `meta/reports`).

## Perspectives — partage à d'autres équipes

Bloqué tant que la page est figée sur Nantes BG 1. Prérequis identifiés :

- **Import PDF du calendrier officiel** pour extraire les journées de l'équipe
  au lieu de `D`/`jours` codés en dur (parser via pdf.js CDN, ou collage du
  texte du PDF — les dates/compétitions sont stables d'une année sur l'autre).
- **Saisie du nom d'équipe + joueurs concernés** au premier usage au lieu de
  `var P` codé en dur → les données deviendraient par équipe
  (`?equipe=…` ou doc de config).
- **⚠ Impact règles** : le modèle actuel whiteliste les noms de joueurs et les
  dates — avec des équipes dynamiques il faudra passer à une validation
  structurelle (forme des docs, tailles max) à la place des listes explicites.

## V2 envisagée — contraintes posées

- **Coût zéro garanti** : rester sur le plan **Spark** (pas de carte attachée —
  en cas de dépassement de quota, coupure du service plutôt que facture).
  Conséquences : pas de Cloud Functions (exigent Blaze) → 100 % client +
  règles ; quota largement suffisant même multi-équipes (~10 joueurs × ~10
  journées ≈ quelques Ko et quelques centaines d'écritures/jour vs 20k/50k
  quotidiens) ; le risque résiduel = spam d'écritures → mitigé par la
  validation structurelle des docs.
- **Didactique** : la page doit s'expliquer seule — assistant de premier usage
  (équipe → joueurs → import du calendrier), micro-aide inline, tuto repliable
  en pied de page (comme tournoi-blackball.html).
- **Calendrier mutualisé PAR DIVISION** : le PDF officiel est commun aux
  équipes d'une même division, mais il y a **3 divisions** (DD 1.1, DD 1.2,
  DD 1.3) et un seul PDF n'en couvre qu'une → docs `calendrier/{division}`
  séparés, parsing côté navigateur, stockage du résultat parsé en JSON
  (quelques Ko — pas le PDF, évite Firebase Storage).
  **Provisionnement distribué** : l'upload ne doit pas dépendre d'un seul
  admin — `allow create` libre si structure valide (un membre de la DD 1.2
  importe le PDF pour toute sa division), `allow update/delete` réservés à
  une clé secrète dans le doc (`request.resource.data.cle == "…"`) — un
  calendrier en place ne peut pas être écrasé par un robot.
  **Fallback** : si le PDF de la division n'est pas dispo → saisie manuelle
  des journées proposée.
  **Côté équipe** : saisie du nom exact (matching sur les équipes du
  calendrier de sa division, suggestion si proche) + joueurs.
- **Lien par équipe** : chaque équipe obtient un lien propre à elle
  (`?equipe=CODE` — le code fait identifiant + accès, comme le code live de
  tournoi-blackball.html).
- **Hébergement club** : migration envisagée vers l'hébergement web du club
  (projet site ASCBG en cours — WP multisite) au lieu de GitHub Pages.
  Les pages sont statiques → simple copie de fichiers ; le repo GitHub peut
  rester la source de vérité. ⚠ **ajouter le nouveau domaine aux referrers
  de la clé API Firebase** (Cloud Console → Identifiants) sinon la synchro
  sera refusée depuis le site du club.

## Test local

Ouvrir les fichiers en `file://` suffit pour l'UI (localStorage seul) — mais
**la synchro Firestore exige un referrer** : servir via
`python3 -m http.server` et ouvrir `http://localhost:8000` ou
`http://127.0.0.1:8000` (referrers autorisés sur la clé API).
Le partage de données en direct nécessite la console Firebase pour vérifier.
