# FongoLab Mobile — Document de design

> **Version** 1.0 — **Statut** : proposition à valider
> **Repo** : `FongoGestion_mobile` (dossier local `FongoLab/`)
> **Cible V1** : espace **stagiaire** — **Identité** : palette reprise à l'identique du site web
> **Socle technique** : Expo SDK 58 / React Native 0.88 / TypeScript / Expo Router (`src/app/`)

---

## 0. Résumé exécutif

Le projet `fongo_gestion_V1` est une plateforme web complète de gestion du club de
formation informatique **FongoLab** (Burundi) : membres, finances (FBu / Lumicash),
devoirs, planning, annonces, présences par QR code, audit.

L'application mobile **FongoLab** (repo `FongoGestion_mobile`) doit porter d'abord
**l'expérience quotidienne du stagiaire** :

1. **Se pointer** en classe (scan QR) et suivre ses présences ;
2. **Consulter ses devoirs** et leurs échéances ;
3. **Suivre et déclarer ses paiements** (Lumicash) ;
4. **Lire les annonces** du club et les partager ;
5. **Gérer son profil**.

Tout repose sur l'**API REST/JSON JWT déjà existante** (`/api/…`, Swagger
`/api/docs/`) : aucune réécriture backend n'est nécessaire pour la V1.

---

## 1. Ce qui existe déjà (synthèse de l'analyse)

### 1.1 Le produit web

| Domaine | Écran web | Ce que le stagiaire y fait |
|---|---|---|
| Tableau de bord | `/dashboard` | Voit son total versé, les compteurs d'activité, les 3 dernières annonces, sa carte « Ma présence » |
| Présences | `/mes-presences` | Statut du jour, bouton **Scanner le QR**, compteurs présences/absences, historique des appels |
| Devoirs | `/devoirs` | Liste filtrée par son équipe (`frontend`/`backend`/`tous`), consignes, fichier joint, échéance `is_expired` |
| Finance | `/finance` | Ses **propres** versements, formulaire **Lumicash** (numéro du club + déclaration « en attente ») |
| Annonces | `/annonces` | Lecture des annonces (3 gabarits : `standard`, `fongo_expo`, `visual`), partage image + lien |
| Paramètres | `/parametres` | Profil + photo, changement de mot de passe |

### 1.2 L'API disponible (à réutiliser telle quelle)

- **Auth** : `POST /api/auth/token/` → `{access, refresh, user}` ; access 8 h,
  refresh 7 j (rotation) ; `Authorization: Bearer`.
- **Profil** : `GET|PATCH /api/auth/me/`, `POST /api/auth/change-password/`.
- **Tableau de bord** : `GET /api/dashboard/`.
- **Présences** : `GET /api/presence/ma-presence/`, `GET /api/presence/historique/`,
  `POST /api/presence/scan/` (`{code}` — accepte aussi `FONGO-PRESENCE:<code>`).
- **Devoirs** : `GET /api/devoirs/` (+ détail par id, serveur filtre la `cible`).
- **Finance** : `GET /api/finance/paiements/` (les siens), `GET /api/finance/overview/`
  (**total validé personnel** + nombre de paiements), `GET /api/finance/paiements/modules/`,
  `POST /api/finance/paiements/lumicash/`.
- **Annonces** : `GET /api/annonces/` (connecté) et `GET /api/annonces/public/` (sans auth).
- **Lecture libre** : `GET /api/planning/events/`, `GET /api/formation/modules/`,
  `GET /api/formation/formateurs/`.

### 1.3 Contraintes & opportunités pour le mobile

| ✅ Favorable | ⚠️ Point de vigilance |
|---|---|
| API REST/JSON documentée (OpenAPI/Redoc) | Réponses hétérogènes : listes paginées `{results}` **ou** tableaux nus → helper `unwrap()` |
| JWT standard Bearer + refresh rotatif | Tokens à stocker en **SecureStore** (pas de `localStorage`) |
| URLs de médias **absolues** (Supabase Storage) | En dev, prévoir un proxy / URL média configurable |
| Endpoint public `/annonces/public/` pour un accueil non connecté | Pas de throttling côté serveur → ne pas marteler l'API |
| Le web utilise déjà le polling (4 s/30 s/60 s) | Sur mobile : réduire les fréquences (batterie) |
| Règles métier **côté serveur** (week-end, 1 seul appel/jour) | Ne **jamais** les reproduire en local : afficher le `detail` du 400 |
| `PAGE_SIZE = 200` global | Prévoir une pagination ou des filtres côté app pour les listes qui grossiront |
| Formats : montants en **entiers FBu**, dates ISO 8601, fuseau `Africa/Bujumbura` | Formatter en `fr-FR … FBu` dans l'app |

### 1.4 Ce que la V1 ne peut PAS afficher (à savoir)

- **Le % de progression de paiement** : `GET /api/finance/progress/` est réservé aux
  admins et `GET /api/parametre/configuration/` (qui porte `frais_formation`) aussi.
  → La V1 affiche le **total validé** et le **nombre de versements** du stagiaire.
  (Si on veut une barre de % : ajouter en V2 un léger `GET /api/finance/ma-progress/`.)
- Les écrans d'administration (membres, audit, appel QR, CRUD) → **V2** (rôles
  `caissier`, `formateur`, `communication`, `admin`, `super_admin`).

---

## 2. Vision & principes du design

### 2.1 Objectif

> Une app **rapide, claire et joyeuse**, pensée pour un usage **quotidien en salle**
> (scan en 1 geste) et des **réseaux lents** : chaque écran doit être lisible en
> une seconde, chaque action principale atteindre le pouce.

### 2.2 Périmètre V1 (in / out)

**In — espace stagiaire**

- Connexion (JWT) + session persistante + déconnexion
- Accueil personnalisé (ma présence, prochaines échéances, annonces, formation)
- Scan QR de présence + historique personnel
- Devoirs : liste, détail, ouverture du fichier joint
- Paiements : mes versements, déclaration Lumicash
- Annonces : liste, détail, partage natif (image + lien)
- Lecture seule : planning, modules, formateurs
- Profil : informations, photo, mot de passe

**Out — reporté en V2/V3**

- CRUD membres, validation des paiements (caissier), CRUD devoirs (formateur),
  création d'annonces (communication), appel QR côté salle (admin), planning CRUD,
  paramétrage du club, journal d'audit, notifications push*.

> *Les push sont un objectif V2 (nécessite Expo Notifications + un service serveur).

### 2.3 Principes directeurs

1. **Mobile-first** : cibles tactiles ≥ 44 pt, actions principales en bas d'écran.
2. **Un geste pour l'essentiel** : le scan se lance depuis l'accueil ou l'onglet central.
3. **Hiérarchie forte** : 1 message clé par carte, chiffres en très grand.
4. **Système, pas d'écrans isolés** : composants réutilisables (carte, badge, bouton…).
5. **Honnêteté des états** : chargement (skeleton), vide, erreur + réessai, hors-ligne.
6. **Respect du serveur** : les règles métier vivent dans l'API, l'app affiche.

---

## 3. Identité visuelle — charte reprise du site web

> L'app mobile **reprend à l'identique** la palette du site FongoGestion :
> bleu nuit Fongo (`#03045e` → `#3f68d8`) + neutres slate, tels que définis
> dans `frontend/src/index.css` (thème `@theme` + variables `:root`).
>
> **Aucune couleur n'est inventée** : chaque valeur hex ci-dessous provient
> du projet web. C'est la consigne de conception — l'app et le site forment
> une seule identité.

### 3.1 Intention

*Continuité, sérieux, clarté* — le site et l'app partagent la même famille
de bleus froids. L'accent est porté par le **bleu nuit `#03045e`** (actions,
onglet actif) et par les **teintes de statut** (vert / ambre / rose), déjà
calibrées pour le contraste AA sur le web.

### 3.2 Palette — mode clair (source : `frontend/src/index.css`)

| Token | Hex | Usage | Origine web |
|---|---|---|---|
| `bg` | `#f8fafc` | Fond d'écran | `body` |
| `surface` | `#ffffff` | Cartes, feuilles | surfaces PrimeReact |
| `surfaceAlt` | `#f1f5f9` | Cartes secondaires, zones de saisie | slate-100 |
| `surfaceHighlight` | `#f1f5ff` | Fond de sélection, badge info | `--highlight-bg` |
| `border` | `#e2e8f0` | Bordures fines (1 px) | `--surface-border` |
| `borderInput` | `#cad5e2` | Bord de champ de saisie | slate-300 |
| `text` | `#0f172b` | Texte principal | `body` color |
| `textSoft` | `#45556c` | Texte secondaire | slate-600 |
| `textFaint` | `#90a1b9` | Texte tertiaire, placeholders | slate-400 |
| **`primary`** | **`#03045e`** | **Actions, onglet actif, liens** | `--primary-color` |
| `primaryHover` | `#1a3a8f` | Appui / survol | bouton primaire hover |
| `primarySoft` | `#f1f5ff` | Fonds d'états pressés, halo focus | `--highlight-bg` |
| `success` | `#009966` | Présent, validé | emerald-600 |
| `warning` | `#e17100` | En attente, échéance proche | amber-600 |
| `danger` | `#ec003f` | Absent, refusé, erreur | rose-600 |

**Teintes de statut** (badges/tags, `.p-tag` du web — fond + texte) :

| Statut | Fond | Texte |
|---|---|---|
| `success` (Validé, Présent) | `#dcfce7` | `#15803d` |
| `warning` (En attente) | `#ffedd5` | `#c2410c` |
| `danger` (Refusé, Absent) | `#fee2e2` | `#b91c1c` |
| `info` (Équipe, neutre) | `#e2eaff` | `#03045e` |

Les échelles complètes (`fongo` 50→950, `slate` 50→950, `emerald`, `rose`,
`amber`) sont transcrites dans `src/theme/colors.ts`, valeurs oklch de
Tailwind v4 converties en hex — identiques au rendu du site.

**Pas de dégradé signature** : le site n'en utilise pas. Les grands blocs
(hero, splash) utilisent `surface` + `primarySoft`, sans gradient.

### 3.3 Mode sombre

**Reporté.** Le site web force `color-scheme: light` et ne définit aucune
palette sombre : l'app suit, `app.json` déclare `userInterfaceStyle: "light"`.
Une variante sombre ne sera envisagée que si le site web l'adopte en premier
(évite une identité visuelle divergente).

> Une préférence **Système / Clair / Sombre** reste au catalogue pour une
> version ultérieure, à condition d'être adossée à une palette sombre
> réellement définie côté web.

### 3.4 Typographie

| Rôle | Police | Graisse | Usage |
|---|---|---|---|
| Display / chiffres | **Space Grotesk** (cible) | Bold 700 | Titres d'écran, montants FBu, compteurs |
| Corps | **Inter** (ou police système) | 400 / 500 / 600 | Textes, libellés, listes |
| Micro-label | Inter | 700, `letterSpacing 1.5` (≈ 0.14em), UPPERCASE | Étiquettes de carte (11 px) |

Échelle : `11 · 13 · 15 · 17 · 20 · 24 · 34` px.
**Chiffres tabulaires** pour les montants (`fontVariant: 'tabular-nums'`).

> **État actuel** : polices **système** (`src/theme/typography.ts`,
> `fontFamily` = System / sans-serif). `expo-font` est installé (peer
> d'`@expo/vector-icons`), mais Space Grotesk / Inter ne sont pas encore
> embarquées : étape de finition (§11, étape 9).

### 3.5 Formes, ombres, espacement

- **Rayons** : champs/boutons `16` · cartes `20` · grandes cartes `28` ·
  pastilles `999` · icône/badge `10–14` (`src/theme/spacing.ts`).
- **Ombres** (claires) : `0 8px 24px rgba(15,23,43,.06)` ; cartes pressées →
  translation `-2 px`.
- **Espacement** : base 4 → `4 / 8 / 12 / 16 / 20 / 24 / 32`.
- **Grille** : marges latérales 20 px, gouttière 12 px.
- **Cible tactile** minimale : 44 × 44 (`touchTarget`).

### 3.6 Iconographie

`@expo/vector-icons` → jeu **Ionicons** (`outline` au repos, `filled` à
l'onglet actif), jamais d'icône seule sans `accessibilityLabel`.

### 3.7 Mouvement

- Transitions d'écran : push natif (iOS) / material (Android) — aucun effet custom lourd.
- Feedback : skeleton *shimmer* (300 ms), barre de progression animée, **haptique**
  `expo-haptics` sur scan réussi/échoué *(non installée en étape 1)*.
- Durée cible : 150–250 ms, `easeOut`. Respecter **Reduce Motion**.

---

## 4. Design system — composants

| Composant | Spécification |
|---|---|
| **Button / Primary** | H 52, rayon 16, fond `primary`, texte blanc 15/600, pressé → `primaryHover`, `disabled` 40 % |
| **Button / Secondary** | H 48, bord 1.5 `primary`, texte `primary` |
| **Button / Ghost** | Texte `primary`, sans bordure (liens « Tout voir ») |
| **Button / Danger** | H 52, fond `danger`, texte blanc — suppression, refus |
| **IconButton** | 44 × 44, fond `surfaceAlt`, rayon 14 |
| **Input** | Label 13/600 au-dessus, champ H 52 rayon 16, bord `borderInput`, focus bord `primary` + halo `primarySoft`, erreur bord `danger` + message 12 px |
| **Card** | Fond `surface`, rayon 20, bord 1 `border`, padding 16, ombre douce |
| **StatTile** | Micro-label uppercase + nombre `display` 34 px + hint 12 px |
| **Badge / statut** | Pastille H 26, rayon 999, pastille-fond teintée + texte : `Validé`/`Présent` vert · `En attente` ambre · `Refusé`/`Absent` rose · `Équipe`/neutre bleu nuit (cf. §3.2) |
| **Tag / cible** | Pastille neutre (`surfaceAlt` + `textSoft`) : « Tous », « Équipe Frontend » |
| **ListRow** | H ≥ 64, icône 40 (rayon 12, fond teinté), titre 15/600, sous-titre 13 `textSoft`, chevron 16 `textFaint`, séparateur `border` |
| **Avatar** | 40/56/80 px, rond, image ou initiales sur `primarySoft` + `primary` |
| **Progress** | H 10, rayon 999, fond `surfaceAlt`, remplissage `primary` |
| **SegmentedControl** | H 40, fond `surfaceAlt`, rayon 12, onglet actif `surface` + ombre |
| **Sheet (feuille)** | Ancre bas, coins hauts 24, padding 20, poignée 36×4, fond `surface` |
| **Toast** | Snack bas, H 52, rayon 16, icône + titre + détail, auto 4 s |
| **Skeleton** | Blocs rayon 12/20, shimmer `surfaceAlt` |
| **EmptyState** | Icône 56 teintée, titre 17/700, texte 14 `textSoft`, CTA optionnel |
| **ErrorState** | Icône `danger`, message issu de l'API (français), bouton **Réessayer** |
| **ScanButton (FAB tab)** | 60 × 60, débordant au-dessus de la tab bar, fond `primary`, icône `qr-code` blanche 28 px |

**Feedback d'erreur** : réutiliser la logique du web (`detail` DRF → phrases
françaises, 401 → « session expirée » + reconnexion, 403 reformulé par rôle).

---

## 5. Navigation & arborescence Expo Router

### 5.1 Structure des routes (obligation AGENTS.md : tout routeur dans `src/app/`)

Le gate de session **n'est pas** une route `index.tsx` racine : créer un tel
fichier entrerait en conflit avec `(tabs)/index.tsx` sur l'URL `/`. Il est
implémenté par `Stack.Protected` dans `src/app/_layout.tsx` :

- `guard={isAuthenticated}` sur les groupes `(tabs)` et `scanner` →
  `redirectTo="/login"` si pas de session ;
- `guard={!isAuthenticated}` sur le groupe `login` → déjà connecté = renvoi `/`.

```
Fongolab/
├── src/
│   ├── app/                      # ← SEULEMENT des routes
│   │   ├── _layout.tsx           # Providers (Query, Auth) + splash JS + gate (Stack.Protected)
│   │   ├── login.tsx             # Connexion (groupe `login`, gardé hors session)
│   │   ├── scanner.tsx           # Scanner — modale plein écran (fullScreenModal)
│   │   ├── (tabs)/
│   │   │   ├── _layout.tsx       # Tab bar 4 entrées + FAB central
│   │   │   ├── index.tsx         # 1. Accueil
│   │   │   ├── devoirs.tsx       # 2. Devoirs (liste)
│   │   │   ├── annonces.tsx      # 4. Annonces (liste)
│   │   │   └── paiements.tsx     # 5. Mes paiements
│   │   ├── presences.tsx         # Ma présence + historique (depuis Accueil)   [V1]
│   │   ├── devoir/[id].tsx       # Détail devoir                              [V1]
│   │   ├── annonce/[id].tsx      # Détail annonce + partage                    [V1]
│   │   ├── planning.tsx          # Lecture seule                               [V1]
│   │   ├── formation.tsx         # Modules + Formateurs (segmented)            [V1]
│   │   └── profil.tsx            # Profil, mot de passe                        [V1]
│   ├── api/                      # client HTTP (axios + refresh) + auth
│   ├── components/               # UI générique (Button, Card, Badge, states…)
│   ├── contexts/                 # AuthContext (session, hydratation, logout)
│   ├── hooks/                    # useDashboard, useMaPresence, useDevoirs…    [V1]
│   ├── lib/                      # storage (SecureStore), session, errors, queryClient
│   ├── theme/                    # tokens : colors, typography, spacing
│   └── types/                    # miroir de frontend/src/types/index.ts
├── app.json                      # config Expo (scheme, typedRoutes, secure-store)
├── .env.example                  # EXPO_PUBLIC_API_URL
├── AGENTS.md                     # règles projet (à respecter)
└── DESIGN.md                     # ce document
```

> État à l'issue de l'**étape 1** (socle) : routes `login`, `(tabs)` (4
> placeholders) et `scanner` sont en place et se naviguent. Les routes marquées
> `[V1]` arrivent aux étapes 2 à 9.

### 5.2 Barre d'onglets (4 entrées + FAB central)

```
┌──────────────────────────────────────────┐
│  🏠        📋      ( 📷 )      💬    👤  │
│ Accueil  Devoirs [Scanner]  Annonces  Pay│
└──────────────────────────────────────────┘
          └─ FAB central (hors Tabs) ─┘
```

| Onglet | Route | Rôle |
|---|---|---|
| Accueil | `(tabs)/index` | Synthèse : ma présence, échéances, annonces, formation |
| Devoirs | `(tabs)/devoirs` | Liste + détail (push) |
| **Scanner** | `scanner` (modale) | **Action quotidienne** — FAB qui ouvre la caméra immédiatement |
| Annonces | `(tabs)/annonces` | Liste + détail (push) |
| Paiements | `(tabs)/paiements` | Mes versements + Lumicash |

- **FAB Scanner** : inséré dans la barre custom (`AppTabBar`) au milieu des
  routes ; l'appui émet `tabPress` puis appelle `router.push('/scanner')`
  (écran déclaré `fullScreenModal` dans `_layout.tsx` racine). Il ne fait donc
  **pas** partie des onglets `expo-router` : la barre affiche 4 onglets réels.
- **Profil** : avatar dans le header de l'accueil → `profil` (hors tabs).
- **Présences (historique)** : carte « Ma présence » de l'accueil → `presences`.
- **Planning / Formation** : raccourcis de l'accueil → `planning`, `formation`.
- Boutons d'onglets : label 11 px, icône 24, actif `primary`, inactif `textFaint`.

### 5.3 Règles de session

- `_layout.tsx` racine : restaure l'utilisateur (`SecureStore`) → tant que la
  session n'est pas résolue → **splash JS** (overlay, pas d'écran vide).
- Le gate est déclaré par `Stack.Protected` (voir §5.1) : sans token → `login`
  (retour arrière impossible sur le groupe protégé), connecté → renvoi `/`.
- 401 → refresh unique partagé (une seule requête en vol) → échec → toast
  « Session expirée » + bascule du gate → redirection automatique `login`.
- Redirections via `router.replace()` (pas de `push` sur les écrans d'auth).
- « Rester connecté » : tokens persistés (`SecureStore`) ; sinon session mémoire
  uniquement, perdue au redémarrage.

---

## 6. Wireframes écran par écran

> Représentation 390 × 844 (iPhone 14 classique / Android courant).

### 6.1 Connexion

```
┌───────────────────────────────────────────┐
│ 9:41              ●●●  4G   ▮▮▮        │
├───────────────────────────────────────────┤
│                                           │
│            ◆  F O N G O L A B             │
│         « Apprendre, coder, devenir. »    │
│                                           │
│  Identifiant                              │
│  ┌─────────────────────────────────────┐  │
│  │ josue01                             │  │
│  └─────────────────────────────────────┘  │
│  Mot de passe                             │
│  ┌───────────────────────────┐   👁       │
│  │ ••••••••••                │            │
│  └───────────────────────────┘            │
│                                           │
│  □ Rester connecté                        │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │         Se connecter   →            │  │
│  └─────────────────────────────────────┘  │
│                                           │
│       Identifiants oubliés ?              │
│       contactez le club de formation.     │
└───────────────────────────────────────────┘
```

- Champs pré-remplis si « Rester connecté » (tokens en SecureStore).
- Erreur 401 → toast « Identifiant ou mot de passe incorrect ».
- Pas de « mot de passe oublié » : **l'API n'a pas d'endpoint** → message de contact.

### 6.2 Accueil (onglet 1)

```
┌───────────────────────────────────────────┐
│ 9:41              ●●●  4G   ▮▮▮        │
├───────────────────────────────────────────┤
│  Bonjour, Josué                    (🔔3) │
│  Stagiaire · Équipe Frontend      (avatar)│
├───────────────────────────────────────────┤
│ ╭─ Ma présence ────────────── [Présent ✓] │
│ │ 08:47 · Appel du matin                  │
│ │                                         │
│ │   ┌─────────┐      ┌─────────┐          │
│ │   │   24    │      │    3    │          │
│ │   │ PRÉSENT│      │ ABSENTS │          │
│ │   └─────────┘      └─────────┘          │
│ │                                         │
│ │  [ 📷  Scanner le QR code ]  (primary)  │
│ ╰─────────────────────────────────────────╯
│                                           │
│  Prochaines échéances            Tout →  │
│  ⏰ Composant React réusable  ven. 12/12  │
│  📅 Atelier Git & GitHub         lun. 15/12│
│                                           │
│  Dernières annonces               Tout →  │
│  ▢ Inscriptions 2026 ouvertes…   il y a 2j│
│  ▢ Atelier Git (affiche FongoExpo) →      │
│                                           │
│  Ma formation                      Tout → │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │ HTML5    │ │ CSS3     │ │ Python   │   │
│  │ 20 h     │ │ 15 h     │ │ 40 h     │   │
│  └──────────┘ └──────────┘ └──────────┘   │
│                                           │
├───────────────────────────────────────────┤
│  🏠       📋      ( 📷 )     💬      💰   │
│ Accueil Devoirs [Scanner] Annonces Paiements│
└───────────────────────────────────────────┘
```

- **Statut de présence** issu de `GET /presence/ma-presence/` :
  `Présent · HH:mm` (vert) / `Absent` (rose, appel ouvert) / `Aucun appel` (neutre)
  / week-end → « Reprise lundi ».
- Le bouton **Scanner** n'apparaît que si `appel_ouvert && !present`.
- 3 compteurs (activité) viennent de `GET /dashboard/` : `activite.devoirs`,
  `.evenements`, `.annonces` — affichés ici en « Prochaines échéances » + annonces.

### 6.3 Scanner (modale plein écran)

```
┌───────────────────────────────────────────┐
│ ✕  Pointage                 ⚡ Lumière   │
├───────────────────────────────────────────┤
│ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ │
│ ▓▓   ┌───────────────────────────┐   ▓▓▓ │
│ ▓▓   │                           │   ▓▓▓ │
│ ▓▓   │       ┌─────────┐         │   ▓▓▓ │
│ ▓▓   │       │ QR CODE │         │   ▓▓▓ │
│ ▓▓   │       └─────────┘         │   ▓▓▓ │
│ ▓▓   │                           │   ▓▓▓ │
│ ▓▓   └───────────────────────────┘   ▓▓▓ │
│ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ │
│                                           │
│   Placez le QR code affiché en salle      │
│   dans le cadre.                          │
│                                           │
│   💡 Un formateur doit d'abord ouvrir     │
│      l'appel — pas de pointage le         │
│      week-end.                            │
│                                           │
│   [ ⌨️  Saisir le code à la main ]        │
└───────────────────────────────────────────┘
```

- Caméra arrière en direct, cadre à coins arrondis, détection continue.
- Payload accepté : `FONGO-PRESENCE:<code>` **ou** le `code` brut.
- **Succès** (haptic + plein écran vert, 1.2 s puis retour) :
  `✓ Présence enregistrée — 08:47 · « Appel du matin »`
- **Déjà pointé** → info : `Déjà pointé pour cet appel`.
- **Échec** (400) → panneau rouge avec le `detail` exact du serveur
  (« QR code inconnu ou appel clôturé… ») + bouton **Réessayer**.
- Fermeture : ✕ ou retour système (modal `presentation: 'fullScreenModal'`).

### 6.4 Ma présence + historique

```
┌───────────────────────────────────────────┐
│ ←  Ma présence                            │
├───────────────────────────────────────────┤
│ ╭─ Statut du jour ──────────── [Présent ✓]│
│ │ Pointé à 08:47 pour « Appel du matin ». │
│ │        [ 📷 Scanner le QR code ]        │
│ ╰─────────────────────────────────────────╯
│  ┌──────────────┐   ┌──────────────┐      │
│  │     24       │   │      3       │      │
│  │  PRÉSENTS    │   │   ABSENTS    │      │
│  └──────────────┘   └──────────────┘      │
│                                           │
│  Historique des appels                    │
│  ┌───────────────────────────────────┐    │
│  │ Appel du matin              Présent 08:47│
│  │ lundi 24 novembre 2026            │    │
│  ├───────────────────────────────────┤    │
│  │ Quiz Python — boucles       Absent     │
│  │ vendredi 21 novembre 2026         │    │
│  ├───────────────────────────────────┤    │
│  │ Appel du matin              Présent 08:52│
│  │ jeudi 20 novembre 2026           │    │
│  └───────────────────────────────────┘    │
└───────────────────────────────────────────┘
```

- Données : `GET /presence/ma-presence/` (statut + compteurs) et
  `GET /presence/historique/` (liste `present`/`horodatage`).
- Historique trié du plus récent au plus ancien, groupé par mois (en-tête sticky).

### 6.5 Devoirs (onglet 2)

```
┌───────────────────────────────────────────┐
│  Devoirs                          🔍      │
├───────────────────────────────────────────┤
│ [ Tous ] [ En cours ] [ Terminés ]        │  ← SegmentedControl
│                                           │
│ ╭───────────────────────────────────────╮ │
│ │ ● En cours          Équipe Frontend    │ │
│ │ Composant React réutilisable           │ │
│ │ ⏰ À rendre vendredi 12/12 · 18:00    │ │
│ │ 📎 projet-fin.zip · 2,4 Mo             │ │
│ │                          [ Ouvrir → ] │ │
│ ╰───────────────────────────────────────╯ │
│ ╭───────────────────────────────────────╮ │
│ │ ● Échéance dépassée       Tous         │ │
│ │ Quiz Python — boucles et comprehensions│ │
│ │ ⏰ 05/12 · 18:00 — depuis 4 jours      │ │
│ ╰───────────────────────────────────────╯ │
│ ╭───────────────────────────────────────╮ │
│ │ ● En cours            Équipe Backend   │ │
│ │ API REST avec Django                   │ │
│ │ ⏰ À rendre lun. 15/12 · 18:00        │ │
│ ╰───────────────────────────────────────╯ │
└───────────────────────────────────────────┘
```

- Tri : échéance la plus proche d'abord ; badge `Échéance dépassée` (rose).
- Filtres : `Tous / En cours / Terminés` (calcul local sur `is_expired`).
- Recherche (`?search=`) via la loupe.

### 6.6 Détail d'un devoir

```
┌───────────────────────────────────────────┐
│ ←  Devoir                                 │
├───────────────────────────────────────────┤
│ [En cours]  [Équipe Frontend]             │
│                                           │
│ Composant React réutilisable              │
│                                           │
│  Consignes                                │
│  Créer un composant Button réutilisable   │
│  avec variantes primary / secondary /     │
│  danger, props typées TypeScript…         │
│  (texte complet, préservation des sauts)  │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │ 📄  projet-fin.zip                 │  │
│  │     Archive · 2,4 Mo · formateur    │  │
│  │                  [ Ouvrir ] ↗       │  │
│  └─────────────────────────────────────┘  │
│                                           │
│  Échéance                                 │
│  Vendredi 12 décembre — 18:00            │
│  Il reste 3 jours                        │
│                                           │
│  [ 📤 Partager la consigne ]              │
│  [ 📥 Télécharger le document ]          │
└───────────────────────────────────────────┘
```

- Fichier joint : image → aperçu inline ; **PDF/Word** → ouverture via
  `expo-web-browser` / visualiseur natif ; téléchargement via `expo-file-system`
  + `expo-sharing` (à vérifier sur la doc SDK 58).
- Boutons masqués si `is_expired` (le serveur refuse de toute façon).

### 6.7 Paiements (onglet 5)

```
┌───────────────────────────────────────────┐
│  Mes paiements                            │
├───────────────────────────────────────────┤
│ ╭─ Déjà réglé ──────────────────────────╮ │
│ │  93 000 FBu                          │ │  ← display 34 px
│ │  4 versements · dont 1 en attente     │ │
│ │  ████████████░░░░░░                   │ │
│ │  Dernier : Module Django · 12 nov.    │ │
│ ╰───────────────────────────────────────╯ │
│                                           │
│  [ 💚  Payer par Lumicash ]  (primary)    │
│                                           │
│  Historique                    Filtrer ▾  │
│  ┌─────────────────────────────────────┐  │
│  │ Module Django          +50 000 FBu  │  │
│  │ Lumicash · 12 nov. 2026             │  │
│  │                    [En attente] 🟡  │  │
│  ├─────────────────────────────────────┤  │
│  │ Autre frais — inscription           │  │
│  │ Cash · 02 nov. 2026    +30 000 FBu  │  │
│  │                       [Validé] 🟢   │  │
│  ├─────────────────────────────────────┤  │
│  │ Module HTML5         +13 000 FBu    │  │
│  │ Cash · 15 oct. 2026                 │  │
│  │                       [Validé] 🟢   │  │
│  └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
```

- Total = `GET /finance/overview/` (le serveur ne renvoie au stagiaire que
  **ses** paiements validés) ; liste = `GET /finance/paiements/`.
- **Pas de barre de %** en V1 (voir §1.4).
- Filtres : statut (`valide` / `en_attente` / `refuse`), moyen, module.

### 6.8 Déclaration Lumicash (feuille)

```
╭─ Payer par Lumicash ──────────────── ✕ ─╮
│                                          │
│  1. Envoyez le montant                   │
│  ┌──────────────────────────────┬─────┐  │
│  │  070 000 00 00               │ Copier│ │
│  └──────────────────────────────┴─────┘  │
│  Numéro Lumicash du club                 │
│                                          │
│  2. Déclarez votre versement             │
│  Montant envoyé (FBu)                    │
│  ┌────────┬──────────────┬──────┐        │
│  │   −    │   50 000     │   +  │        │
│  └────────┴──────────────┴──────┘        │
│  Module / type de frais                  │
│  ┌───────────────────────────────┐  ▾    │
│  │ Module Django          40 h   │       │
│  └───────────────────────────────┘       │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │          ✓  J'ai payé              │  │
│  └────────────────────────────────────┘  │
│  Votre demande apparaîtra « En attente » │
│  jusqu'à la validation d'un caissier.    │
╰──────────────────────────────────────────╯
```

- Le numéro du club provient de `VITE_LUMICASH_NUMBER` côté web → côté mobile :
  variable d'env `EXPO_PUBLIC_LUMICASH_NUMBER` (même valeur).
- « Copier » → `expo-clipboard` + toast confirmation.
- Cibles : modules (`GET /finance/paiements/modules/`) + « Autre frais »
  (champ `motif` requis, ≥ 2 caractères) → `POST /finance/paiements/lumicash/`.
- Succès → toast + l'item apparaît « En attente » en tête de liste.

### 6.9 Annonces (onglet 4)

```
┌───────────────────────────────────────────┐
│  Annonces                            🔍   │
├───────────────────────────────────────────┤
│ ╭───────────────────────────────────────╮ │
│ │ ┌───────────────────────────────────┐ │ │
│ │ │        [ image1 / poster ]        │ │ │
│ │ └───────────────────────────────────┘ │ │
│ │ Affiche FongoExpo · Prévue 20/12      │ │
│ │ Atelier Git & GitHub                  │ │
│ │ Par communication · il y a 2 j        │ │
│ │  [📤 Partager]              [Lire →] │ │
│ ╰───────────────────────────────────────╯ │
│ ╭───────────────────────────────────────╮ │
│ │ ● Nouveau                 Tous        │ │
│ │ Inscriptions 2026 ouvertes            │ │
│ │ Prévue le 15/12 · 2026                │ │
│ │                          [ Lire → ]   │ │
│ ╰───────────────────────────────────────╯ │
│ ╭───────────────────────────────────────╮ │
│ │ ● Nouveau            Stagiaires       │ │
│ │ Changement de salle samedi            │ │
│ │ Prévue le 12/12 · 2026                │ │
│ ╰───────────────────────────────────────╯ │
└───────────────────────────────────────────┘
```

- Gabarits : `visual` → image seule ; `fongo_expo` → affiche (image1/image2) ;
  `standard` → carte texte.
- **Partage natif** (Web Share API équivalent `expo-sharing` / `Share` React Native)
  : image du poster + lien `https://<domaine>/a/<id>/` (page Open Graph existante).

### 6.10 Détail annonce + partage

```
┌───────────────────────────────────────────┐
│ ←  Annonce                                │
├───────────────────────────────────────────┤
│ ┌───────────────────────────────────────┐ │
│ │          [ poster / image1 ]          │ │
│ └───────────────────────────────────────┘ │
│                                           │
│ Atelier Git & GitHub                       │
│ Prévue le 20/12/2026 · 10:00              │
│ Par communication                          │
│                                           │
│  Bonjour à toutes et à tous…              │
│  (contenu complet, style éditorial)       │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │  📤  Partager (WhatsApp, Facebook…) │  │
│  └─────────────────────────────────────┘  │
│  [ 💾 Télécharger l'image ]               │
└───────────────────────────────────────────┘
```

### 6.11 Planning & Formation (lecture seule)

```
┌────────────────────────────┐  ┌────────────────────────────┐
│ ←  Planning               │  │ ←  Formation               │
├────────────────────────────┤  ├────────────────────────────┤
│  CE MOIS-CI                │  │ [Modules][Formateurs]      │
│ ┌────────────────────────┐ │  │                            │
│ │ 15  LUN · Atelier Git  │ │  │ ┌────────────────────────┐ │
│ │     dirigé par Patrick │ │  │ │ 1  HTML5      20 h     │ │
│ ├────────────────────────┤ │  │ │    sujets + formateurs │ │
│ │ 18  JEU · Clôture      │ │  │ ├────────────────────────┤ │
│ │     inscription        │ │  │ │ 2  CSS3       15 h     │ │
│ └────────────────────────┘ │  │ ├────────────────────────┤ │
│                            │  │ │ 3  Python     40 h     │ │
│  (trié par date, futur     │  │ └────────────────────────┘ │
│   d'abord)                 │  │                            │
└────────────────────────────┘  └────────────────────────────┘
```

### 6.12 Profil & paramètres

```
┌───────────────────────────────────────────┐
│ ←  Mon profil                             │
├───────────────────────────────────────────┤
│              (avatar 80)                  │
│            Josue ESANGA                   │
│      Stagiaire · Équipe Frontend          │
│         [ 📷 Changer la photo ]           │
│                                           │
│  Compte                                   │
│  👤  Informations personnelles      →     │
│  🔑  Changer le mot de passe        →     │
│                                           │
│  Préférences                              │
│  🌙  Apparence : Système ▾   (V1.5)        │
│      (Système / Clair / Sombre)           │
│  🔔  Notifications          (V2)          │
│                                           │
│  À propos                                 │
│  ℹ️  FongoLab Mobile · v1.0.0             │
│  🌐  Site du club                       ↗ │
│                                           │
│  [ ⏻  Se déconnecter ]   (danger)        │
└───────────────────────────────────────────┘
```

- Photo : `PATCH /auth/me/` en **multipart** (même champ que le web).
- Mot de passe : 3 champs (`old_password`, `new_password`, `confirm_password`)
  avec règles affichées en temps réel.
- Déconnexion → confirmation → purge SecureStore → `router.replace('/login')`.

---

## 7. Parcours utilisateurs clés

### 7.1 Pointage du jour (le geste n° 1)

```
Accueil → [Scanner le QR]  ──ou──  onglet 📷
   → caméra plein écran
   → détection du payload FONGO-PRESENCE:<code>
   → POST /presence/scan/ {code}
        ├─ 201 {present, horodatage} → haptic ✓ + écran vert → retour Accueil (badge « Présent »)
        ├─ 200 {deja_enregistre}      → info « Déjà pointé » → retour
        └─ 400 {detail}              → panneau erreur + Réessayer
```

Règles **côté serveur** affichées telles quelles : pas de scan le week-end,
un seul appel par jour, QR expiré si l'appel est fermé.

### 7.2 Paiement Lumicash

```
Paiements → [Payer par Lumicash] → feuille
   → 1. copier le numéro du club (toast « Numéro copié »)
   → 2. saisir montant + module/motif
   → POST /finance/paiements/lumicash/
   → 201 → toast « Demande envoyée » → liste avec badge « En attente »
   → ultérieurement : le caissier valide/refuse (V2) → badge mis à jour
```

### 7.3 Devoir & pièce jointe

```
Devoirs → carte → détail devoir
   → [Ouvrir] fichier → visualiseur / navigateur externe
   → [Télécharger] → expo-file-system → partage systeme
   → expiration masque les actions
```

### 7.4 Partage d'une annonce

```
Annonces → détail → [Partager]
   → image (poster ou image1) + lien /a/<id>/
   → Share sheet native (WhatsApp, Facebook, Gmail…)
   → hors-ligne → fallback : partage du lien seul
```

### 7.5 Session

```
Lancement → restauration SecureStore → GET /auth/me/
   → 200 : session OK → (tabs)
   → 401 : POST /auth/token/refresh/ (une seule requête partagée)
        ├─ 200 : retry de la requête initiale
        └─ échec : toast « Session expirée » → /login
```

---

## 8. Architecture technique

### 8.1 Couche API

```
src/api/
├── client.ts        # axios (ou fetch) + baseURL EXPO_PUBLIC_API_URL
│                    #  + intercepteur Authorization Bearer
│                    #  + refresh unique partagé (dedup) sur 401
│                    #  + déduplication des requêtes identiques
├── auth.ts          # token/, refresh/, me/, change-password/
├── dashboard.ts     # dashboard/
├── presences.ts     # ma-presence/, historique/, scan/
├── devoirs.ts       # devoirs/
├── finance.ts       # paiements/, overview/, paiements/modules/, lumicash/
├── annonces.ts      # annonces/, annonces/public/
├── planning.ts      # planning/events/
├── formation.ts     # formation/modules/, formation/formateurs/
└── errors.ts        # {detail} | {champ:[…]} → phrases françaises
```

- **`unwrap<T>(data)`** : `{results}` → tableau, ou tableau nu.
- Tokens : `expo-secure-store` (`fongo.access_token`, `fongo.refresh_token`),
  utilisateur en `AsyncStorage` (non sensible) pour l'affichage immédiat.
- Cache : **TanStack Query v5** (mêmes `queryKey` que le web pour la cohérence :
  `['ma-presence']`, `['devoirs']`, `['payments']`, `['dashboard']`…),
  `staleTime` 5 min, pas de retry sur 4xx.

### 8.2 Endpoints utilisés par écran (V1)

| Écran | Endpoints |
|---|---|
| Login | `POST /auth/token/` |
| Accueil | `GET /dashboard/`, `GET /presence/ma-presence/`, `GET /annonces/` |
| Scanner | `POST /presence/scan/` |
| Ma présence | `GET /presence/ma-presence/`, `GET /presence/historique/` |
| Devoirs | `GET /devoirs/` |
| Paiements | `GET /finance/overview/`, `GET /finance/paiements/`, `GET /finance/paiements/modules/`, `POST /finance/paiements/lumicash/` |
| Annonces | `GET /annonces/`, `GET /annonces/<id>/` |
| Planning | `GET /planning/events/` |
| Formation | `GET /formation/modules/`, `GET /formation/formateurs/` |
| Profil | `GET|PATCH /auth/me/` (multipart), `POST /auth/change-password/` |
| Hors-ligne publique (V1.5) | `GET /annonces/public/` |

### 8.3 Dépendances proposées (→ **vérifier impérativement sur
`https://docs.expo.dev/versions/v58.0.0/` avant chaque `expo install`**, conformément
à AGENTS.md)

| Besoin | Module | Commande | État étape 1 |
|---|---|---|---|
| Navigation | `expo-router` (+ `react-native-screens`, `react-native-safe-area-context`) | `npx expo install expo-router` | ✅ installé (58.0.32) |
| HTTP + état serveur | `axios` (portage du web), `@tanstack/react-query` | `npm i axios @tanstack/react-query` | ✅ installé (1.21.1 / 5.108.4) |
| Stockage sûr | `expo-secure-store` | `npx expo install expo-secure-store` | ✅ installé (58.0.3) + plugin |
| Icônes | `@expo/vector-icons` (+ peer `expo-font`) | `npx expo install expo-font` | ✅ installé (Ionicons) |
| Scan QR / caméra | `expo-camera` (`CameraView` + scan codes-barres) | `npx expo install expo-camera` | ⏳ étape 4 |
| Presse-papiers | `expo-clipboard` | `npx expo install expo-clipboard` | ⏳ étape 6 |
| Fichiers / partage | `expo-file-system`, `expo-sharing`, `expo-web-browser` | `npx expo install …` | ⏳ étapes 5–7 |
| Typographie | polices Space Grotesk / Inter via `expo-font` | `npx expo install expo-font` | ⏳ étape 9 (`expo-font` déjà là) |
| Retour haptique | `expo-haptics` | `npx expo install expo-haptics` | ⏳ étape 4 |

> `expo-linear-gradient` est **retiré** : la palette reprend le site web, qui
> n'utilise aucun dégradé (§3.2).
| État serveur | `@tanstack/react-query` | `npm i @tanstack/react-query` |
| HTTP | `axios` (portage direct du web) ou `fetch` natif | — |

> Permissions : `camera` (scan) et `mediaLibrary`/`photos` (sauvegarde d'image) à
> déclarer dans `app.json` avec les textes de justification iOS/Android.

### 8.4 Patterns d'écran (communs)

Chaque écran suit le même squelette :

```
Header (titre + actions)          ← SafeArea + Stack.Screen options
  ├─ état loading   → <Skeleton/>  (3 blocs)
  ├─ état erreur    → <ErrorState onRetry/>  (message API)
  ├─ état vide      → <EmptyState/>
  └─ état succès    → contenu + pull-to-refresh (RefreshControl)
Footer / action principale         ← thumb zone (bas)
```

#### Gestion du clavier (obligatoire sur tout écran à saisie)

**Réglage global (`app.json`) : `android.softwareKeyboardLayoutMode: "pan"`**

C'est le correctif du bug natif **react-native#33164** : en `adjustResize`
(le défaut), Android **referme le clavier dès l'ouverture au premier appui**.
`pan` glisse la fenêtre pour révéler le champ au lieu de la redimensionner,
ce qui supprime le reflow qui fait perdre le focus. `pan` est aussi le réglage
recommandé par Expo pour les navigateurs à onglets inférieurs (notre cas).

> ⚠️ La doc Expo interdit de cumuler `pan` avec `behavior="padding"` : le champ
> serait décalé deux fois. D'où `behavior={undefined}` sur Android.

```tsx
<KeyboardAvoidingView
  behavior={Platform.OS === 'ios' ? 'padding' : undefined}
  style={{ flex: 1 }}
>
  <ScrollView
    keyboardShouldPersistTaps="always"   // un appui hors champ ne ferme pas le clavier
    keyboardDismissMode="none"           // défaut iOS `on-drag` : ferme au moindre reflow
  >
    …
  </ScrollView>
</KeyboardAvoidingView>
```

| Règle | Raison |
|---|---|
| `softwareKeyboardLayoutMode: "pan"` (figé dans `app.json`) | **fixe** le clavier Android qui s'ouvre puis se referme au 1er appui (react-native#33164) |
| `behavior` = `padding` (iOS) / `undefined` (Android) | `pan` gère Android ; cumuler `pan` + `padding` décale le champ deux fois |
| `keyboardDismissMode="none"` | le défaut iOS (`on-drag`) referme le clavier au repositionnement du `ScrollView` |
| `keyboardShouldPersistTaps="always"` | empêche la fermeture par un appui accidentel |
| **pas** d'`autoFocus` au montage | le focus se déclencherait pendant l'animation de navigation → clavier ouvert/fermé |

> `react-native-keyboard-controller` est la solution robuste (UI thread), mais
> exige un **development build** — incompatible avec Expo Go. À réserver à V2.

### 8.5 Performance & réseaux lents (contexte Burundi)

- Images : `resizeMode="contain"`, chargement paresseux, dimensions déclarées.
- Listes : `FlatList` avec `initialNumToRender`, pas de map sur de gros tableaux.
- Polling **supprimé** sur mobile : invalidation au focus de l'écran
  (`refetchOnWindowFocus`) + pull-to-refresh ; le scan ne pollue pas.
- Retry exponentiel uniquement sur erreurs réseau (pas 4xx).
- Bundle : routes `lazy` via Expo Router (dynamique) ; splash jusqu'au premier rendu.

### 8.6 Hors-ligne

- Cache React Query = lecture hors-ligne des dernières données vues (badge « données
  du … »).
- Aucune écriture mise en file d'attente en V1 (scan et paiement exigent le réseau).
- Écran dédié si `NetInfo` signale l'absence de réseau (à confirmer SDK 58).

---

## 9. Accessibilité & inclusion

- Cibles tactiles ≥ 44 × 44 pt ; actions principales entre 560 et 760 px de hauteur.
- Contraste AA : texte `text` sur `surface` ≥ 7:1 ; libellés sur pastilles ≥ 4.5:1.
- `accessibilityLabel` sur **toutes** les icônes et boutons uniquement icône.
- Support **Dynamic Type** (texte non tronqué, cartes qui reflowent).
- `Reduce Motion` (le mode sombre est reporté, cf. §3.3).
- Langue : **français** (identique au web), chiffres `fr-FR`, montants `FBu`.
- Fuseau : `Africa/Bujumbura` — formatage local, jamais de calcul de date côté client
  pour les règles métier.

---

## 10. Roadmap

| Version | Contenu |
|---|---|
| **V1 — Stagiaire** | Socle (thème, auth, navigation), Accueil, Scanner + présences, Devoirs, Paiements + Lumicash, Annonces + partage, Planning/Formation lecture, Profil |
| **V1.5** | Mode hors-ligne élargi, annonces publiques non connectées, préférence d'apparence poussée, animations de finition |
| **V2 — Encadrement** | Caissier (validation des versements, stats), Formateur (CRUD devoirs), Communication (CRUD annonces + poster), Admin (appel QR en salle avec feuille d'appel live, planning CRUD), notifications push |
| **V3 — Direction** | Super admin : membres, configuration du club, audit |

---

## 11. Plan d'implémentation (étapes)

> ✅ = terminé et vérifié (`npx tsc --noEmit`, `npx eslint .`,
> `npx expo-doctor`, `npx expo export --platform android`).

1. ✅ **Socle + Auth** (étape 1, réalisée en une passe — les deux étaient
   indissociables : le gate de session a besoin du thème et des composants)
   - Expo SDK 58 / RN 0.88 / React 19.3 / TypeScript, `main:
     expo-router/entry`, `scheme: "fongolab"`, `typedRoutes`.
   - `src/theme/` : tokens couleurs (palette web) / typo / espacements.
   - Composants génériques : Button, Card, Badge, Input, Skeleton, EmptyState,
     ErrorState, Toast, AppTabBar (+ FAB).
   - `src/api/client.ts` (Bearer + refresh dédupliqué, émetteur de session) et
     `src/lib/errors.ts`.
   - `src/contexts/AuthContext.tsx` + `src/lib/storage.ts` (SecureStore, cache
     mémoire, `hydrate()`, option « Rester connecté »).
   - Gate de session par `Stack.Protected` dans `_layout.tsx` (pas de route
     `index.tsx` racine), splash JS pendant l'hydratation.
   - `login.tsx` pleinement fonctionnel, `(tabs)/` 4 onglets + FAB,
     `scanner.tsx` (modale placeholder).
   - Copie de `frontend/src/types/index.ts` → `src/types/`.
2. ✅ *fusionné dans l'étape 1* : `login.tsx`, gate, SecureStore.
3. **Accueil** + `useDashboard` + carte Ma présence.
4. **Présences** : `presences.tsx`, `scanner.tsx` (caméra réelle), haptics, états d'erreur.
5. **Devoirs** : liste + détail + ouverture/téléchargement de fichier.
6. **Paiements** : liste, overview, feuille Lumicash (copie numéro, formulaire).
7. **Annonces** : liste (3 gabarits) + détail + partage natif.
8. **Planning / Formation / Profil** : écrans de lecture + multipart photo + mot de passe.
9. **Finitions** : polices (Space Grotesk / Inter), accessibilité,
   pull-to-refresh, images, `expo-haptics`.
10. **Qualité** : `npx tsc --noEmit`, `npx expo lint`, `npx expo-doctor` avant chaque
    livraison ; puis `eas build` (jamais de dossier `ios/`/`android/` manuel).

**État global** : étape 1 ✅ — socle prêt, 0 erreur TypeScript, 0 problème
ESLint, expo-doctor 20/20, export Metro valide. Prochaine étape : **3 — Accueil**.

---

## Annexe A — Matrice des rôles (rappel, pour la V2)

| Rôle | Ce qu'il fera dans l'app |
|---|---|
| `stagiaire` | **V1 complète** (ci-dessus) |
| `admin` + `caissier` | + validation/refus des paiements, stats, liste stagiaires |
| `admin` + `formateur` | + création/édition/suppression des devoirs (avec pièce jointe) |
| `admin` + `communication` | + création/édition des annonces + upload du poster |
| `admin` (sans rôle) | lecture seule finance (% sans montants) |
| `super_admin` | + membres (CRUD), appel QR en salle, planning, configuration, audit |

## Annexe B — Champs API à ne pas oublier

- `roles_supplementaires` : **JSON string** en écriture multipart (`'["frontend"]'`).
- Champs `*_clear` (booléens write-only) pour purger une image/fichier existant.
- `stagiaire` (objet) en lecture vs `id_stagiaire` (write-only) côté paiement.
- Champs `*_display` fournis par l'API → **ne pas** recalculer les libellés côté app.
- Pagination DRF `{count, next, previous, results}` — helper `unwrap()` obligatoire.
