# Sprint 03b — Deploy Netlify Staging

> Spec per Claude Code. Connettere il repo `Paolocesareo/editorkit` a Netlify e mettere online un ambiente di staging per testare il magic link e gli sprint successivi senza dipendere da localhost.

---

## 1. OBIETTIVO

Mettere online il frontend EditorKit su un dominio Netlify pubblico (`editorkit-staging.netlify.app`), collegato al branch `main` del repo per deploy automatico ad ogni push.

A fine sprint, Paolo deve poter aprire `https://editorkit-staging.netlify.app/login` da qualsiasi browser, ricevere il magic link a `test1@studiocesareo.it`, cliccarlo, e atterrare su `/dashboard` autenticato — esattamente come funzionava in localhost in Sprint 03.

---

## 2. CONTESTO

Workspace: https://github.com/Paolocesareo/Paolo/blob/master/progetti/editorkit.md

Sprint 03 chiuso (auth magic link end-to-end funzionante in localhost). Ora vogliamo:

- ambiente di staging URL pubblico
- auto-deploy ad ogni push su `main` (così Sprint 04, 05 testano direttamente da staging)
- redirect URL Supabase aggiornata per accettare il dominio Netlify

Hosting confermato in workspace: **Netlify**. Integrazione Next.js gestita dal plugin `@netlify/plugin-nextjs` (auto-detect).

---

## 3. OWNER

**Claude Code**

L'orchestratore (Paolo + Claude orch) si occupa solo dell'ultimo step: aggiungere il dominio Netlify alle redirect URL di Supabase Auth dopo che Claude Code comunica l'URL finale.

---

## 4. PRE-REQUISITI

- Sprint 03 chiuso e pushato su `Paolocesareo/editorkit` branch `main`
- `.env.local` in `C:\001-Sviluppo\editorkit\` popolato con credenziali Supabase reali (Claude Code le riusa per Netlify)
- Account Netlify di Paolo esistente (se manca, Paolo lo crea con un account Google/GitHub durante lo step 7.2)

---

## 5. CARTELLA DI LAVORO

`C:\001-Sviluppo\editorkit\` — usare path assoluti, no `cd`.

---

## 6. DELIVERABLE

1. Sito Netlify creato e linkato al repo `Paolocesareo/editorkit`
2. Slug sito: `editorkit-staging` → URL pubblico `https://editorkit-staging.netlify.app`
3. Build settings configurati (auto-detect Next.js)
4. Env vars Netlify popolate, identiche al `.env.local` locale ma con `NEXT_PUBLIC_SITE_URL` puntato al dominio Netlify
5. Primo deploy completato con successo, URL pubblico raggiungibile
6. Smoke test base: home `/` e `/login` rispondono 200
7. Comunicazione a Paolo del dominio finale (esatto, copiabile)

---

## 7. SPECIFICHE DETTAGLIATE

### 7.1 Installa netlify-cli globale

```bash
npm install -g netlify-cli
```

Verifica:

```bash
netlify --version
```

### 7.2 Login Netlify

```bash
netlify login
```

Si apre il browser. **Paolo si autentica** col suo account (Google/GitHub/email). Una volta autenticato il CLI riceve il token, Claude Code può proseguire da solo.

Se Paolo non ha account Netlify, lo crea sul momento dalla pagina aperta dal CLI (gratis, account starter ok).

### 7.3 Inizializza progetto e link

Dalla cartella `C:\001-Sviluppo\editorkit\` (path assoluto, usare `--cwd`):

```bash
netlify init --cwd C:\001-Sviluppo\editorkit
```

Risposte alle domande interattive:

- **What would you like to do?** → "Create & configure a new site"
- **Team** → quello di Paolo (default)
- **Site name** → `editorkit-staging`
- **Build command** → `npm run build` (auto-detect Next.js dovrebbe già proporlo)
- **Directory to deploy** → `.next` (auto-detect)
- **Netlify functions folder** → vuoto / default

Il comando crea il file `netlify.toml` se non esiste e aggiunge la connessione al repo GitHub via OAuth (Paolo conferma l'autorizzazione GitHub se richiesta).

### 7.4 Verifica `netlify.toml`

Se non esiste già, creare `netlify.toml` in `C:\001-Sviluppo\editorkit\netlify.toml`:

```toml
[build]
  command = "npm run build"
  publish = ".next"

[[plugins]]
  package = "@netlify/plugin-nextjs"
```

Il plugin Next.js viene installato automaticamente da Netlify al primo build.

### 7.5 Setta env vars su Netlify

Leggere i valori da `C:\001-Sviluppo\editorkit\.env.local` e replicarli su Netlify. Eseguire un comando per ciascuna variabile:

```bash
netlify env:set NEXT_PUBLIC_SUPABASE_URL "<valore da .env.local>" --cwd C:\001-Sviluppo\editorkit
netlify env:set NEXT_PUBLIC_SUPABASE_ANON_KEY "<valore da .env.local>" --cwd C:\001-Sviluppo\editorkit
netlify env:set SUPABASE_SERVICE_ROLE_KEY "<valore da .env.local>" --cwd C:\001-Sviluppo\editorkit
netlify env:set HEYZINE_CLIENT_ID "877c907b0a7d03e8" --cwd C:\001-Sviluppo\editorkit
netlify env:set NEXT_PUBLIC_SITE_URL "https://editorkit-staging.netlify.app" --cwd C:\001-Sviluppo\editorkit
```

**Importante**: `NEXT_PUBLIC_SITE_URL` su Netlify deve essere il dominio Netlify, **non** `http://localhost:3000`. Questa è la differenza chiave tra `.env.local` locale ed env vars Netlify.

Verifica:

```bash
netlify env:list --cwd C:\001-Sviluppo\editorkit
```

### 7.6 Trigger primo deploy production

```bash
netlify deploy --prod --cwd C:\001-Sviluppo\editorkit
```

Attendi il completamento del build. Output atteso: URL del sito + tempo di build.

In alternativa, dato che `netlify init` collega al repo, basta un push su `main` per triggerare il deploy automatico:

```bash
git -C C:\001-Sviluppo\editorkit commit --allow-empty -m "chore: trigger initial netlify deploy"
git -C C:\001-Sviluppo\editorkit push origin main
```

### 7.7 Smoke test

Dopo il completamento del deploy, verificare via curl:

```bash
curl -sI https://editorkit-staging.netlify.app/ | head -1
curl -sI https://editorkit-staging.netlify.app/login | head -1
```

Atteso: `HTTP/2 200` su entrambe.

**NON testare il magic link da staging in questo sprint** — la redirect URL Supabase non è ancora configurata, l'orchestratore lo fa dopo aver ricevuto il DONE.

---

## 8. CRITERI DI ACCETTAZIONE

- [ ] Sito Netlify creato, slug `editorkit-staging`
- [ ] Repo `Paolocesareo/editorkit` linkato (deploy automatico al push su `main` attivo)
- [ ] `netlify env:list` mostra 5 variabili: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `HEYZINE_CLIENT_ID`, `NEXT_PUBLIC_SITE_URL`
- [ ] `NEXT_PUBLIC_SITE_URL` su Netlify = `https://editorkit-staging.netlify.app` (non localhost)
- [ ] Build production completato senza errori
- [ ] `https://editorkit-staging.netlify.app/` risponde 200
- [ ] `https://editorkit-staging.netlify.app/login` risponde 200
- [ ] `netlify.toml` committato sul repo

---

## 9. NOTE PER L'AGENTE

- **Slug `editorkit-staging` può essere già preso** da un altro utente Netlify. Se la creazione fallisce per conflitto nome, ripiegare su `editorkit-staging-csflab` e segnalarlo nel DONE.
- **Non modificare il codice frontend** in questo sprint. Se il build fallisce per errori di codice, FERMARE e segnalare a Paolo prima di patchare.
- **Non toccare** `lib/supabase/`, `middleware.ts`, `app/auth/` — sono di Sprint 03.
- **Sicurezza env vars**: i valori Supabase letti da `.env.local` sono sensibili. Non stamparli nel terminal output. Non committarli da nessuna parte oltre Netlify.
- **In caso di errore deploy** (build failure, env mancanti, plugin not found), FERMARE e segnalare. Niente pivot non concordati.

---

## 10. OUTPUT ATTESO PER PAOLO

Al termine, Claude Code consegna:

```
[CLAUDE CODE FRONTEND SPRINT 03B — DONE]

Sito Netlify: https://editorkit-staging.netlify.app
Repo collegato: Paolocesareo/editorkit (auto-deploy su push main)

Env vars settate (5):
- NEXT_PUBLIC_SUPABASE_URL
- NEXT_PUBLIC_SUPABASE_ANON_KEY
- SUPABASE_SERVICE_ROLE_KEY
- HEYZINE_CLIENT_ID
- NEXT_PUBLIC_SITE_URL = https://editorkit-staging.netlify.app

Smoke test:
- /: ✅ 200
- /login: ✅ 200

In attesa: orchestratore aggiunge https://editorkit-staging.netlify.app/auth/callback alle redirect URL di Supabase Auth, poi Paolo testa il flow magic link da staging.
```

---

*Sprint 03b — Versione 1 — 30 aprile 2026*
