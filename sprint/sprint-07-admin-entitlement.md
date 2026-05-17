# Sprint 07 — Admin entitlement: login staff a password + assegnazione manuale abbonamenti

> Spec per Claude Code. Si appoggia sulla shell `/admin` di 06/06b. Verifica orchestratore su staging.

---

## 1. OBIETTIVO

Dare all'admin una UI per **assegnare manualmente l'accesso alle testate** a un utente lettore, senza Stripe, e introdurre il **login a password per lo staff operativo** (binario auth distinto dal magic link dei lettori).

Oggi l'entitlement manuale si fa via INSERT SQL a mano nella tabella `subscriptions`. Sprint 07 sostituisce quell'operazione manuale con una pagina admin. Niente di più.

Non è un sistema di billing, non è self-service, non tocca Stripe. È lo strumento operativo che serve a Paolo per far entrare i lettori del pilota mentre il paywall reale resta parcheggiato.

---

## 2. CONTESTO

Workspace canonico (NON ripeterlo, leggilo per orientarti): https://github.com/Paolocesareo/Paolo/blob/master/progetti/flipkit.md

Brand pubblico **FlipKit**; interno (repo, Supabase, path) resta `editorkit` — invariato (decisione #29).

### 2.1 Decisione auth a due binari (vincolo di prodotto, NON discuterlo)

- **Lettori → magic link.** Implementazione rimandata alla zona Sprint 09/10 (paywall + import anagrafica). **Fuori scope qui.** In Sprint 07 NON si costruisce né si tocca il login lettore.
- **Staff operativo → password**, impostata fuori dall'app (vedi §4), un account per persona, nessun self-service, nessuna email transazionale coinvolta. **Questo binario lo costruisce Sprint 07.**

Motivo del doppio binario: lo staff si passa l'utenza, serve un account per persona per l'accountability; i lettori sono un parco one-shot, la password lì genera solo attrito e churn.

### 2.2 Cosa esiste già e NON si rifà

- `app/admin/layout.tsx` — gate admin via env `ADMIN_EMAILS` (lista email, lowercase). Se loggato e email ∈ `ADMIN_EMAILS` → entra; loggato ma non in lista → redirect `/dashboard`; non loggato → redirect `/login?next=/admin/onboarding`. **Il modello "admin = env var" resta. NON introdurre una tabella ruoli** (decisione di sessione: niente sovrastruttura per il pilota).
- `middleware.ts` — protegge `/dashboard` e `/admin`: se non loggato → `/login` con `?next=`.
- `lib/supabase/server.ts` (SSR, anon, cookie), `lib/supabase/client.ts` (browser, anon), `lib/supabase/admin.ts` (`createAdminClient()`, service_role, **bypassa RLS**, solo server-side). Pattern già consolidato in 06/06b: le operazioni admin girano via `createAdminClient()` con ricontrollo gate server-side.
- `app/auth/callback/route.ts` — exchange PKCE del magic link (lettori). **Invariato, non toccarlo.**
- Schema DB live su `editorkit-prod`. Tabelle rilevanti (migration `001`, RLS `002`):
  - `subscriptions`: `user_id` (FK `auth.users`), `tier` enum `subscription_tier` (`tier_1|tier_2|tier_3`), `status` enum `subscription_status` (`active|past_due|canceled|paused|incomplete`), `publications_included UUID[]`, `publisher_id` (FK `publishers`, nullable), `current_period_start`, `current_period_end`, `cancel_at_period_end`, `stripe_subscription_id` (nullable, UNIQUE), `stripe_price_id` (nullable).
  - `user_profiles`: `id` (FK `auth.users`), `email`, `full_name`, `stripe_customer_id`. **Trigger automatico** `on_auth_user_created`: ogni `auth.users` creato genera in automatico la riga `user_profiles`. Non inserire `user_profiles` a mano.
  - `publications`: 5 testate pilota, `publisher_id` valorizzato, `is_active=true`, leggibili pubblicamente.
  - RLS entitlement (policy `editions_paid_users_see_published`): un utente vede un'edizione **se** ha una `subscriptions` con `status='active'` **AND** `current_period_end > now()` **AND** la `publication_id` è in `publications_included`. **Questa policy è il motore. Assegnare un entitlement = scrivere una riga `subscriptions` che la soddisfa. Revocarlo = portarla a `status='canceled'`.**

### 2.3 Disallineamento repo ↔ DB (LEGGI, incide sulla migration)

Nel repo `supabase/migrations/` esistono **solo** `001`, `002`, `003`, `004`. Sul DB live esistono **anche** migration `005`, `006` (policy RLS su `storage.objects` per l'onboarding) e `007` (inerte, superata): **applicate su Supabase ma mai committate nel repo**. Conseguenza operativa per questo sprint: la tua nuova migration si chiama **`008`**, non `005`. NON creare/ricreare/toccare `005/006/007`. (Il riallineamento repo↔DB delle 005/006/007 è un memo separato dell'orchestratore, fuori da questo sprint.)

Decisione #32: se trovi una contraddizione interna a questa spec, **FERMATI e segnala a Paolo**, non scegliere da solo.
Decisione #48: la verifica contro i criteri di accettazione è dell'orchestratore, non tua. Non auto-dichiarare "done" senza i test della §8 realmente eseguiti.

---

## 3. OWNER

**Claude Code.** Orchestratore verifica su staging.

---

## 4. PRE-REQUISITI (azioni Paolo, NON sono bug se mancano)

- **Primo account staff con password**: lo crea Paolo da Dashboard Supabase → Authentication → Add user → email + password + "Auto Confirm User". Questo è il bootstrap (l'app non gestisce la creazione staff in questo sprint, vedi §7.6). Claude Code: assumi che esista almeno un account staff con password la cui email è anche in `ADMIN_EMAILS`.
- **`ADMIN_EMAILS`** su Netlify contiene l'email di quell'account staff (già env esistente da 06; verificare che includa l'utente di test che Paolo userà).
- Env già presenti (06/06b, non reintrodurre): `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `NEXT_PUBLIC_SITE_URL`, `ADMIN_EMAILS`.
- Repo locale `C:\001-Sviluppo\editorkit\`, path assoluti.

Se al test il login staff fallisce perché l'account non esiste o l'email non è in `ADMIN_EMAILS`: **FERMATI e segnala** (è azione Paolo, non un tuo bug).

---

## 5. CARTELLA DI LAVORO

Sotto `C:\001-Sviluppo\editorkit\`, rispettando la struttura esistente. File nuovi previsti in §6. Migration in `supabase\migrations\008_*.sql`.

---

## 6. DELIVERABLE

1. Pagina `/admin/login` — form email + password, `signInWithPassword`.
2. `middleware.ts` modificato: `/admin/*` non loggato → `/admin/login` (non `/login`); `/admin/login` resta pubblica; `/dashboard` invariato → `/login`.
3. Sezione admin entitlement `/admin/entitlement`:
   - Lista utenti + ricerca per email.
   - Crea utente lettore per email se non esiste.
   - Per utente: lista subscription + crea assegnazione manuale + revoca.
4. Migration `008` — colonna `assigned_by` su `subscriptions` (accountability).
5. Voci di navigazione minime tra `/admin/onboarding` e `/admin/entitlement` nella shell admin.
6. `.env.local.example` aggiornato solo se introduci nuove variabili (non dovrebbero servirne).
7. Output `[CLAUDE CODE SPRINT 07 — DONE]` (formato §10).

---

## 7. SPECIFICHE DETTAGLIATE

### 7.1 Migration `008` — accountability

`supabase/migrations/008_entitlement_admin.sql`:

```sql
ALTER TABLE subscriptions
  ADD COLUMN IF NOT EXISTS assigned_by TEXT;
COMMENT ON COLUMN subscriptions.assigned_by IS
  'Email dello staff che ha creato/modificato manualmente questo entitlement. NULL = origine non manuale (es. Stripe futuro).';
```

Nessuna nuova policy RLS: `subscriptions` ha già RLS (utente legge solo le proprie); l'admin opera sempre via `createAdminClient()` (service_role, bypassa RLS). Non aggiungere policy che aprano `subscriptions` a `authenticated`.

Applicazione: committa il file nel repo. Se hai accesso CLI/SQL al DB applicala e dichiaralo; altrimenti consegna lo statement a Paolo nell'output e segnalalo. NON applicarla via una migration rinumerata 005/006/007.

### 7.2 Pagina `/admin/login` (binario staff a password)

`app/admin/login/page.tsx` — client component.

- Form: `email`, `password`, bottone "Accedi".
- Submit: `createClient()` (browser, `lib/supabase/client.ts`) → `await supabase.auth.signInWithPassword({ email, password })`.
- Successo → redirect a `/admin/entitlement` (usa `NEXT_PUBLIC_SITE_URL` come base se costruisci URL assoluti server-side; lato client `router.push('/admin/entitlement')` è sufficiente).
- Errore credenziali → messaggio in italiano sotto il form ("Credenziali non valide"), nessun dettaglio tecnico esposto. Non distinguere "email inesistente" da "password errata".
- Nessun link "password dimenticata", nessun "registrati", nessun magic link in questa pagina. È il binario staff, deliberatamente spoglio.
- Solo componenti `components/ui/*` esistenti (`Input`, `Label`, `Button`, `Card`). Nessuna libreria nuova.
- Testi user-facing in italiano.

### 7.3 `middleware.ts` — instradamento del gate

Oggi `/dashboard` e `/admin` non loggati vanno a `/login`. Modifica **mirata**:

- Path che inizia con `/admin` **e** non è `/admin/login` **e** utente assente → redirect a `/admin/login` (con `?next=` opzionale verso il path richiesto).
- `/admin/login` → sempre accessibile senza sessione (altrimenti loop).
- `/dashboard` e ogni altro path protetto → comportamento attuale invariato (→ `/login`).

Non cambiare il `matcher` né la logica di refresh sessione `@supabase/ssr`: solo il ramo di redirect per `/admin`. Lascia `app/admin/layout.tsx` com'è: continua a fare il gate `ADMIN_EMAILS` server-side (un account a password ma con email non in `ADMIN_EMAILS` deve comunque finire a `/dashboard` — comportamento voluto, non un bug).

### 7.4 `/admin/entitlement` — UI assegnazione manuale

Server component per il fetch iniziale, isole client per le azioni. Tutte le operazioni dati passano da **server action / route handler** con `createAdminClient()` e **ricontrollo gate admin server-side** (non fidarsi del solo layout, pattern 06/06b).

**Gate server-side riusabile**: replica la logica di `isAdmin(email)` di `app/admin/layout.tsx` in un helper condiviso server-side (es. `lib/admin/guard.ts`) e usalo all'inizio di ogni server action / route handler di questo sprint. Non duplicare la stringa di parsing in più punti.

**a) Lista utenti**
- Fonte: `user_profiles` via `createAdminClient()` (service_role). Colonne mostrate: `email`, `full_name`, numero di subscription con `status='active' AND current_period_end > now()`.
- Campo di ricerca per email (match case-insensitive, anche parziale). Lista limitata/paginata in modo ragionevole (es. 50 righe + ricerca); non serve paginazione sofisticata per il pilota (~500 utenti).

**b) Trova-o-crea utente lettore per email**
- Input email. Se esiste un `auth.users` con quella email → selezionalo.
- Se non esiste → crealo con `supabase.auth.admin.createUser({ email, email_confirm: true })` via service_role. **NON impostare una password significativa** per i lettori (è binario magic link, fuori scope: l'accesso effettivo del lettore è zona 09/10). **NON inserire a mano in `auth.users` via SQL** (il trigger crea `user_profiles` da solo; e l'INSERT SQL diretto su `auth.users` ha il muro dei token NOT NULL — usare SEMPRE l'API admin che lo gestisce).
- Mostra chiaramente che l'utente è stato creato e che il suo accesso (login) sarà abilitato in una fase successiva del prodotto — non promettere che può già entrare.

**c) Subscription di un utente + crea assegnazione manuale**
- Mostra le subscription dell'utente selezionato: `tier`, `status`, `current_period_start/end`, testate incluse (risolvi gli UUID di `publications_included` ai nomi `publications.name`), `assigned_by`.
- Form "Assegna accesso":
  - Multi-select testate tra le `publications` con `is_active=true` (le 5 pilota).
  - `tier`: select su `tier_1|tier_2|tier_3` (default `tier_2`, coerente con gli utenti pilota esistenti).
  - `current_period_start` (default oggi), `current_period_end` (default oggi + 12 mesi; libero ma obbligatorio e futuro).
  - On submit (server action, service_role, gate ricontrollato): INSERT in `subscriptions` con `user_id`, `tier`, `status='active'`, `publications_included = <uuid testate selezionate>`, `current_period_start`, `current_period_end`, `cancel_at_period_end=false`, `stripe_subscription_id = NULL`, `stripe_price_id = NULL`, `publisher_id` = **derivato** dal `publisher_id` delle publication selezionate (le 5 pilota appartengono tutte allo stesso publisher; se per assurdo le selezionate avessero publisher diversi → FERMATI e segnala, non indovinare), `assigned_by = <email staff loggato>`.
- **Revoca**: bottone su una subscription attiva → server action che porta `status='canceled'` e setta `assigned_by = <email staff loggato>` (traccia chi ha revocato). `canceled` è escluso dalla policy entitlement → l'accesso cade subito. Non cancellare la riga (storia).
- **Modifica**: per il pilota è sufficiente "revoca + crea nuova". NON implementare un editor inline complesso delle testate incluse a meno che non sia banale; se lo fai, resta su service_role + gate + `assigned_by` aggiornato. Non è obbligatorio per i criteri di accettazione.
- Conferma esplicita prima di revocare (dialog/conferma semplice). Tutti i testi in italiano.

### 7.5 Navigazione shell admin

Aggiungi nella shell admin (header di `app/admin/layout.tsx` o componente nav minimo) due link: "Onboarding testate" → `/admin/onboarding`, "Utenti & accessi" → `/admin/entitlement`. Minimale, coerente con lo stile esistente (Tailwind/shadcn già in uso). Niente redesign del layout.

### 7.6 Fuori scope (dichiarato, NON implementare)

- **Creazione/gestione account staff dalla UI.** Per il pilota gli account staff li crea Paolo da Dashboard Supabase (§4) e la loro email va in `ADMIN_EMAILS` (env Netlify, azione manuale Paolo, richiede redeploy). È un attrito accettato per pochi operatori. **Non "risolverlo" introducendo una tabella ruoli o un pannello di gestione staff**: violerebbe la decisione di sessione. Parcheggiato.
- **Login lettore / magic link / reset password lettore.** Zona Sprint 09/10. Non toccare `auth/callback`, non aggiungere flussi email.
- **Stripe, paywall, pricing.** Parcheggiati altrove.
- **Audit log dettagliato** oltre la colonna `assigned_by`. Sufficiente così per il pilota.

### 7.7 Commit

```
git -C C:\001-Sviluppo\editorkit add .
git -C C:\001-Sviluppo\editorkit commit -m "feat: admin entitlement + login staff password (sprint 07)"
git -C C:\001-Sviluppo\editorkit push origin main
```

Auto-deploy Netlify attivo su `main`.

---

## 8. CRITERI DI ACCETTAZIONE

- [ ] `/admin/login`: form email+password; credenziali valide di un account staff → sessione attiva; credenziali errate → messaggio "Credenziali non valide", nessuno stack/dettaglio.
- [ ] `middleware.ts`: visitando `/admin/entitlement` da non loggato → redirect a **`/admin/login`** (non `/login`). `/admin/login` raggiungibile senza sessione. `/dashboard` da non loggato → ancora `/login` (invariato).
- [ ] Account a password con email **non** in `ADMIN_EMAILS` → dopo login finisce a `/dashboard` (gate layout invariato).
- [ ] Migration `008` presente nel repo come `008_*.sql`; `subscriptions.assigned_by` esiste sul DB. Nessuna migration 005/006/007 creata o modificata.
- [ ] `/admin/entitlement` da admin: lista utenti con email/nome/n. sub attive; ricerca per email funziona.
- [ ] Trova-o-crea: inserendo un'email nuova viene creato l'utente (via `auth.admin.createUser`, `email_confirm:true`, no password); `user_profiles` popolato dal trigger; nessun INSERT SQL diretto su `auth.users`.
- [ ] Assegnazione: selezionate 1+ testate, tier, periodo futuro → riga `subscriptions` `status='active'`, `publications_included` corretto, `publisher_id` derivato, `assigned_by` = email staff, `stripe_*` NULL.
- [ ] **Effetto entitlement reale**: l'utente a cui è stato assegnato l'accesso, loggandosi, sfoglia l'edizione della testata assegnata (pagina Sprint 05 invariata); una testata NON assegnata resta paywall. (Verifica con un utente pilota esistente.)
- [ ] Revoca: porta `status='canceled'`, `assigned_by` aggiornato, l'accesso cade subito (l'edizione torna paywall per quell'utente). Riga non cancellata.
- [ ] Tutte le operazioni dati passano da server action/route con gate admin ricontrollato server-side + `createAdminClient()`. Nessuna apertura RLS di `subscriptions` ad `authenticated`.
- [ ] Nessuna tabella ruoli introdotta. Nessun pannello gestione staff. Nessun flusso email/magic link aggiunto.
- [ ] `npm run build` passa. Push `main` → deploy Netlify ready.
- [ ] Nessun segreto committato.

---

## 9. NOTE PER L'AGENTE

- **Due binari auth, non mescolarli.** `/admin/login` = password, spoglio, nessun magic link. Il login lettori NON esiste in questo sprint e non va simulato. Se ti accorgi che per "vedere l'effetto" ti servirebbe far loggare un lettore creato da te: usa un **utente pilota già esistente** a cui assegni l'accesso, non costruire un login lettore.
- **`admin = ADMIN_EMAILS` è una scelta deliberata per il pilota.** L'attrito (Paolo aggiunge l'email a mano su Netlify per ogni nuovo staff) è accettato e dichiarato. Non introdurre una tabella `roles`/`is_staff` per "migliorarlo": è esplicitamente vietato in questo sprint (§7.6).
- **service_role solo server-side.** Mai esporre `SUPABASE_SERVICE_ROLE_KEY` al client. Ogni mutazione su `subscriptions`/`auth` passa da server action o route handler con gate ricontrollato.
- **Migration = 008.** Repo fermo a 004, DB ha 005/006/007 non committate (§2.3). Non rinumerare, non ricreare le 005/006/007.
- **Trigger user_profiles**: non inserire `user_profiles` a mano, lo fa il trigger su `auth.users`. Crea utenti SOLO via `auth.admin.createUser` (gestisce i campi token che via SQL diretto darebbero "Database error finding user").
- **Decisione #32**: contraddizione interna alla spec → fermati e segnala. In particolare se publication selezionate avessero `publisher_id` diversi.
- **Decisione #48**: i criteri §8 li verifichi davvero prima di dichiarare done; l'orchestratore ri-verifica su staging.
- **In caso di blocco** (account staff inesistente, `ADMIN_EMAILS` non allineata, `signInWithPassword` che non apre sessione SSR, build rotta da un cambio middleware): FERMATI e segnala con la diagnosi. Niente pivot non concordati, niente login lettore come ripiego, niente tabella ruoli come scorciatoia.
- Lingua: commenti brevi in italiano, testi user-facing in italiano.

---

## 10. OUTPUT ATTESO PER PAOLO

```
[CLAUDE CODE SPRINT 07 — DONE]

Repo: https://github.com/Paolocesareo/editorkit (commit <sha>)
Auto-deploy Netlify: build state ready
URL staging: https://editorkit-staging.netlify.app
Migration 008: <applicata da me | statement consegnato a Paolo>

File creati/modificati:
- app/admin/login/page.tsx (nuovo, login staff password)
- middleware.ts (instradamento /admin -> /admin/login)
- app/admin/entitlement/page.tsx (+ isole client) (nuovo)
- lib/admin/guard.ts (gate admin server-side condiviso) (nuovo)
- lib/admin/entitlement-actions.ts (server actions) (nuovo)
- app/admin/layout.tsx (nav minima) (modificato)
- supabase/migrations/008_entitlement_admin.sql (nuovo)
- .env.local.example (solo se nuove variabili — atteso: nessuna)

Test eseguiti:
- Login staff password ok / credenziali errate messaggio: ✅
- Middleware /admin -> /admin/login, /dashboard -> /login invariato: ✅
- Account email non in ADMIN_EMAILS -> /dashboard: ✅
- Migration 008, subscriptions.assigned_by presente: ✅
- Lista/ricerca utenti: ✅
- Trova-o-crea utente (auth.admin.createUser, no SQL diretto, user_profiles via trigger): ✅
- Assegnazione: riga active, publications_included, publisher_id derivato, assigned_by, stripe NULL: ✅
- Effetto reale: utente entitled sfoglia la testata assegnata, le altre restano paywall: ✅
- Revoca: status canceled, accesso cade subito, riga non cancellata: ✅
- npm run build: ✅

Azioni Paolo richieste/fatte: account staff creato in Supabase <fatto/da fare>, email in ADMIN_EMAILS <ok/da fare>

Aperture/segnalazioni: <eventuali stop, oppure "nessuna">

Pronto per verifica orchestratore su staging.
```

---

*Sprint 07 — Versione 1 — 17 maggio 2026 — Admin entitlement: login staff a password (binario distinto dal magic link lettori) + assegnazione manuale abbonamenti, sopra la shell /admin di 06/06b*
