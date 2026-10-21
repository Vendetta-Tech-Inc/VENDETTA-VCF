# VENDETTA VCF PRO V9 ULTIMATE

Gestion de contacts, de dossiers (avec mode campagne) et de fichiers VCF.
React + TypeScript + Vite · Netlify Functions ou Vercel Functions (Node.js) · Neon PostgreSQL · aucune dépendance à Supabase, Firebase ou Appwrite.

> **État honnête du projet** : le code est complet (voir « Fonctionnalités »), mais il a été écrit dans un environnement
> sans accès réseau. Les modules VCF/normalisation ont été exécutés pour de vrai ; **le reste (compilation Vite, SQL sur
> Neon, déploiement Netlify) n'a pas encore été exécuté**. Suivez la liste de contrôle « Première mise en route » et
> lancez `npm run typecheck` et `npm test` avant de mettre en production.

## Fonctionnalités

- Page d'accueil publique avec statistiques **réelles** (utilisateurs, contacts, dossiers, VCF générés), footer légal.
- Inscription, connexion, déconnexion, récupération du mot de passe (e-mail), profil, suppression du compte.
- Contacts : CRUD, recherche, filtre par dossier, tri A→Z, détection de doublons, sélection multiple, déplacement, export.
- Dossiers : CRUD, compteurs, export complet ; **mode campagne** (capacité 400 par défaut, modifiable par l'admin,
  refus quand c'est complet, ouverture automatique du dossier suivant, page publique `/c/<id>` avec compteur réel).
- Générateur VCF 3.0 (défaut) / 4.0, UTF-8, CRLF, pliage à 75 octets, aperçu, nom personnalisé, historique.
- Importateur VCF : analyse, aperçu, doublons, erreurs, choix du dossier, confirmation, rapport, limites de taille/nombre.
- Espace `/admin` : tableau de bord (stats issues de Neon), utilisateurs (suspendre, réactiver, rôle, supprimer),
  dossiers/campagnes, contacts (coordonnées masquées par défaut, affichage journalisé), historiques, paramètres du site,
  journal d'activité en ajout seul.
- Thème clair/sombre, navigation latérale (ordinateur) et barre inférieure (mobile).

## Première mise en route

```bash
npm install
cp .env.example .env        # puis renseigner DATABASE_URL, SESSION_SECRET, APP_BASE_URL
npm run migrate             # applique migrations/001_init.sql et 002_rate_limits.sql sur Neon
npm run typecheck           # vérifie les types (le build Vite ne les vérifie pas)
npm test                    # tests unitaires (+ intégration si TEST_DATABASE_URL est défini)
npm run dev:netlify         # site + fonctions sur http://localhost:8888
```

Créer le premier administrateur : inscrivez-vous sur le site, puis

```bash
npm run make-admin -- votre@email.com
```

(Un utilisateur ne peut jamais devenir admin depuis le navigateur ; la promotion se fait par script ou par un admin existant.)

## Variables d'environnement

| Variable | Rôle |
|---|---|
| `DATABASE_URL` | Chaîne Neon. Utilisez l'URL **pooled** (hôte contenant `-pooler`). Serveur uniquement. |
| `SESSION_SECRET` | ≥ 32 caractères aléatoires (`openssl rand -base64 48`). Sert à signer (HMAC) les jetons stockés. |
| `APP_BASE_URL` | URL publique, ex. `https://monsite.netlify.app` (liens de réinitialisation, contrôle d'origine). |
| `EMAIL_PROVIDER_API_KEY`, `EMAIL_FROM` | Clé API [Resend](https://resend.com) et expéditeur vérifié. Sans elles, aucun e-mail de réinitialisation n'est envoyé (en local, le lien s'affiche dans la console). |
| `DB_POOL_MAX` | Connexions par instance (défaut 3). |
| `TEST_DATABASE_URL` | Base Neon jetable pour les tests d'intégration. |

Aucune variable secrète n'est préfixée `VITE_` : rien de sensible n'est envoyé au navigateur.

## Déploiement Netlify

1. Poussez le dépôt sur GitHub/GitLab, puis « Add new site → Import an existing project » dans Netlify.
2. Build : `npm run build`, dossier de publication : `dist` (déjà dans `netlify.toml`).
3. Netlify → Site configuration → Environment variables : ajoutez `DATABASE_URL`, `SESSION_SECRET`, `APP_BASE_URL`
   (+ e-mail si besoin). Marquez-les comme secrètes.
4. Exécutez `npm run migrate` une fois (depuis votre poste, avec le `.env`) contre la base Neon de production.
5. Déployez, créez votre compte, puis `npm run make-admin -- votre@email.com`.
6. Vérifiez : inscription, connexion, création d'un dossier, ajout d'un contact, export VCF, accès `/admin`.

Si Netlify signale un souci d'empaquetage du module `pg` : dans `netlify.toml`, retirez `external_node_modules`
ou remplacez `pg` par `@neondatabase/serverless` dans `netlify/functions/_lib/db.ts`.

## Déploiement Vercel (alternative)

Le même backend tourne sur Vercel : `api/[...path].ts` appelle le gestionnaire de `netlify/functions/api.ts`, et `vercel.json`
fournit la redirection SPA (`/admin`, `/c/<id>`…), les en-têtes de sécurité et la durée maximale des fonctions.

1. Importez le dépôt dans Vercel (« Add New → Project ») ; le preset Vite est détecté (build `npm run build`, sortie `dist`).
2. Settings → Environment Variables : `DATABASE_URL`, `SESSION_SECRET`, `APP_BASE_URL` (ex. `https://monsite.vercel.app`)
   et, si besoin, `EMAIL_PROVIDER_API_KEY`, `EMAIL_FROM`.
3. Lancez `npm run migrate` une fois contre la base Neon, déployez, puis `npm run make-admin -- votre@email.com`.

Points d'attention Vercel :
- Corps de requête limité à 4,5 Mo : la limite d'import réglable dans l'admin est plafonnée à 3 400 Ko.
- Si `APP_BASE_URL` est absent, l'URL de production Vercel est utilisée pour les liens de réinitialisation.
- L'adresse IP des visiteurs (limitation des tentatives) est lue dans `x-vercel-forwarded-for`, posé par Vercel.
- Cette variante n'a pas encore été déployée : validez-la comme le reste (connexion, dossier, export, `/admin`).

## Sécurité (résumé)

- Mots de passe : bcrypt (coût 12), 10 à 72 octets ; comparaison factice si l'e-mail est inconnu.
- Sessions : jeton aléatoire 256 bits en cookie `HttpOnly`, `Secure` (HTTPS), `SameSite=Lax` ; seule l'empreinte HMAC est en base ; 14 jours.
- CSRF : contrôle de l'en-tête `Origin` + jeton `x-csrf-token` (double soumission, empreinte en base).
- Rôle toujours relu en base à chaque requête ; `/admin` n'est qu'une route d'interface, l'API refuse (403) les non-admins.
- Limitation de débit : connexion (5 échecs/15 min par e-mail, 30 par IP), inscription, mot de passe oublié, import/export, inscriptions publiques.
- SQL : uniquement des requêtes paramétrées ; clés dynamiques des `UPDATE` issues de schémas zod fermés.
- Propriété : chaque requête filtre par `user_id` côté SQL ; les ressources d'autrui répondent 404.
- Capacité : `SELECT … FOR UPDATE` sur le dossier puis comptage dans la même transaction (test de concurrence fourni).
- Import : le fichier est lu comme texte, ré-analysé côté serveur, limité en taille et en nombre ; jamais exécuté.
- Journal d'audit en ajout seul (triggers PostgreSQL) ; aucun mot de passe, jeton ni coordonnée complète n'y est écrit.
- En-têtes de sécurité et CSP dans `netlify.toml`.

## Écarts et ajouts par rapport au cahier des charges

- Tables ajoutées : `sessions`, `password_reset_tokens`, `login_attempts`, `rate_limit_events`, `import_history`, `schema_migrations`.
- La page publique de campagne (`/c/<id>`) est un ajout : sans elle, le mode campagne n'aurait aucune « inscription » à compter.
- La capacité n'est appliquée qu'aux dossiers en mode campagne ; les autres dossiers n'ont pas de limite.
- Les textes légaux (confidentialité, conditions, assistance) sont des modèles à faire relire.
- Les polices sont celles du système (aucune ressource externe, compatible avec la CSP).
- Le logo se règle par URL https (pas de téléversement de fichier).

## Structure

```
migrations/          001_init.sql, 002_rate_limits.sql
netlify/functions/   api.ts (point d'entrée /api/*), routes/, _lib/
api/                 [...path].ts : point d'entrée Vercel (même gestionnaire)
vercel.json          configuration Vercel
src/                 frontend React (pages/, pages/admin/, components/, lib/)
scripts/             migrate.mjs, make-admin.mjs
tests/               vcf.test.ts, security.test.ts, integration.test.ts
```
