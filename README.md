# SaaS Starter — Next.js + Supabase + Stripe + Vercel

Un starter simple pour lancer un SaaS : authentification, base de données
avec sécurité au niveau des lignes (RLS), abonnements Stripe, et
déploiement Vercel.

## Stack

- **Next.js 15** (App Router, TypeScript, Tailwind CSS)
- **Supabase** — authentification + base Postgres
- **Stripe** — abonnements et facturation
- **Vercel** — hébergement

## Structure du projet

```
src/
  app/
    page.tsx                 -> Landing page
    login/, signup/           -> Pages d'authentification
    auth/actions.ts           -> Server Actions (login, signup, déconnexion)
    auth/callback/route.ts    -> Confirmation email / retour OAuth
    dashboard/                -> Espace connecté (protégé)
    pricing/                  -> Page tarifs + bouton d'abonnement
    api/stripe/checkout/      -> Crée une session de paiement Stripe
    api/stripe/portal/        -> Ouvre le portail client Stripe
    api/stripe/webhook/       -> Reçoit les événements Stripe
  lib/
    supabase/client.ts        -> Client Supabase (navigateur)
    supabase/server.ts        -> Client Supabase (serveur)
    supabase/service.ts       -> Client "service role" (webhook uniquement)
    supabase/middleware.ts    -> Rafraîchit la session + protège /dashboard
    stripe.ts                 -> Client Stripe (serveur)
  middleware.ts                -> Point d'entrée du middleware Next.js
supabase/schema.sql            -> Schéma SQL à exécuter dans Supabase
```

---

## 1. Créer les comptes nécessaires

Tu vas avoir besoin de 4 comptes gratuits (si ce n'est pas déjà fait) :

1. **GitHub** — https://github.com/signup
2. **Supabase** — https://supabase.com (connexion possible avec GitHub)
3. **Stripe** — https://dashboard.stripe.com/register
4. **Vercel** — https://vercel.com/signup (connexion possible avec GitHub)

---

## 2. Mettre le code sur GitHub

Depuis le dossier du projet (en local, une fois le zip décompressé) :

```bash
git init
git add .
git commit -m "Initial commit"
```

Puis sur GitHub : **New repository** (ne coche ni README, ni .gitignore,
ni licence — ils existent déjà), copie l'URL du repo, puis :

```bash
git remote add origin https://github.com/<ton-compte>/<ton-repo>.git
git branch -M main
git push -u origin main
```

> Si le projet a déjà un dossier `.git` (c'est le cas dans le zip fourni),
> saute `git init` et vérifie juste avec `git status`.

---

## 3. Configurer Supabase

1. Sur https://supabase.com/dashboard, clique **New project**. Choisis un
   nom, un mot de passe de base de données (garde-le de côté), une région
   proche de tes utilisateurs.
2. Une fois le projet créé, va dans **SQL Editor** > **New query**, colle
   le contenu du fichier `supabase/schema.sql` du projet, puis **Run**.
   Cela crée les tables `profiles` et `subscriptions`, leurs policies RLS,
   et les triggers automatiques.
3. Va dans **Project Settings > API** et note :
   - `Project URL` → `NEXT_PUBLIC_SUPABASE_URL`
   - `anon public` key → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `service_role` key (⚠️ secrète, ne jamais l'exposer) →
     `SUPABASE_SERVICE_ROLE_KEY`
4. (Optionnel mais recommandé) Dans **Authentication > URL Configuration**,
   ajoute l'URL de ton site en production (ex: `https://ton-site.vercel.app`)
   dans "Site URL" et "Redirect URLs" (avec `/auth/callback`), pour que la
   confirmation d'email fonctionne une fois déployé.

---

## 4. Configurer Stripe

1. Sur https://dashboard.stripe.com, reste en **mode test** pour commencer
   (interrupteur en haut à droite).
2. Va dans **Product catalog > Add product**, crée un ou deux produits
   (ex: "Starter" à 9€/mois, "Pro" à 29€/mois) en mode récurrent. Note
   l'ID de chaque prix (`price_...`) → `STRIPE_PRICE_STARTER` /
   `STRIPE_PRICE_PRO`.
3. Va dans **Developers > API keys** et note :
   - `Secret key` → `STRIPE_SECRET_KEY`
   - `Publishable key` → `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`
4. Le webhook (`STRIPE_WEBHOOK_SECRET`) sera créé à l'étape 6, une fois le
   site déployé (Stripe a besoin d'une URL publique).
5. Active le **Customer portal** dans **Settings > Billing > Customer
   portal** (pour que le bouton "Gérer mon abonnement" fonctionne).

---

## 5. Déployer sur Vercel

1. Sur https://vercel.com/new, importe le repo GitHub que tu viens de
   créer.
2. Dans **Environment Variables**, ajoute toutes les variables listées
   dans `.env.example` (copie les valeurs récupérées aux étapes 3 et 4).
   Pour `NEXT_PUBLIC_SITE_URL`, mets l'URL Vercel qui te sera attribuée
   (tu peux la corriger après le premier déploiement si besoin).
3. Clique **Deploy**. Après quelques minutes, ton site est en ligne.

---

## 6. Brancher le webhook Stripe (après le premier déploiement)

1. Sur https://dashboard.stripe.com/webhooks, clique **Add endpoint**.
2. URL de l'endpoint : `https://ton-site.vercel.app/api/stripe/webhook`
3. Événements à écouter :
   - `checkout.session.completed`
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
4. Une fois créé, copie le **Signing secret** (`whsec_...`) et ajoute-le
   dans Vercel comme variable `STRIPE_WEBHOOK_SECRET` (**Project Settings
   > Environment Variables**), puis redéploie (**Deployments > ... >
   Redeploy**).

---

## 7. Tester en local (optionnel)

```bash
npm install
cp .env.example .env.local   # puis remplis les valeurs
npm run dev
```

Ouvre http://localhost:3000. Pour tester les paiements en local, utilise
le [Stripe CLI](https://stripe.com/docs/stripe-cli) :

```bash
stripe listen --forward-to localhost:3000/api/stripe/webhook
```

Utilise une carte de test Stripe pour payer : `4242 4242 4242 4242`, une
date future, n'importe quel CVC.

---

## Aller plus loin

- Ajouter la connexion Google/GitHub : **Supabase > Authentication >
  Providers**, puis afficher un bouton OAuth sur `/login` et `/signup`
  (`supabase.auth.signInWithOAuth`).
- Ajouter des données propres à ton produit : crée une nouvelle table
  Supabase avec une colonne `user_id` et une policy RLS du même modèle que
  `profiles`/`subscriptions` (`auth.uid() = user_id`).
- Emails transactionnels personnalisés : **Supabase > Authentication >
  Email Templates**.
