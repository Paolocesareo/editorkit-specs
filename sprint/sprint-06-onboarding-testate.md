# Sprint 06 — Onboarding testate (upload manuale admin, pipeline source-agnostic)

> Spec per Kimi Code. Backend + pipeline server-side + UI admin minimale. Verifica orchestratore.

---

## 1. OBIETTIVO

Strumento admin interno per caricare manualmente le edizioni PDF delle 5 testate pilota sul **nostro** Supabase (`editorkit-prod`, non il backend OPS).

Flusso per ogni edizione: PDF → Storage privato → conversione flipbook via Heyzine REST → record `editions` pubblicato e visibile in edicola.

Vincolo architetturale non negoziabile: la pipeline conversione + registrazione è scritta come **funzione indipendente dalla sorgente del PDF**. L'upload manuale è solo il primo driver. L'ingestione automatica (FTP/webhook dalla prepress OPS) si aggancerà in uno sprint successivo riusando la stessa funzione, senza riprogettare nulla. Vedi §7.4 — è anche un criterio di accettazione.

---

## 2. CONTESTO

Workspace progetto: https://github.com/Paolocesareo/Paolo/blob/master/progetti/editorkit.md

Decisioni che vincolano questo sprint:

- **#40 Opzione A confermata**: upload manuale admin per il pilota, pipeline disaccoppiata dalla sorgente PDF.
- **#41 Intel prepress**: il contratto di ingestione lo definiamo noi, la prepress si conforma. Conseguenza per questo sprint: NON serve modellare nulla dell'FTP di VNP. Basta che il core di ingestione (§7.4) sia chiamabile da un driver qualsiasi.
- **#35 Slug OPS come colonna in `publications`**: questo sprint aggiunge la colonna e fa il backfill delle 5 pilota.
- **#25 / #28 Pattern Heyzine validato**: `POST https://heyzine.com/api1/rest` con `pdf` (URL pubblico) + `client_id`. Tempi reali 40-53s per 60-72 pagine. Thumbnail derivabile dal solo flipbook_id: `https://cdnc.heyzine.com/flip-book/cover/<id>.jpg`. Niente oembed.
- **#32**: se trovi sezioni in contraddizione DENTRO questa spec, FERMATI e segnala. Non scegliere in autonomia.

Stato repo rilevante (`Paolocesareo/editorkit`, branch `main`):

- `app/edizione/[publication-slug]/[date]/page.tsx` — pagina edizione (Sprint 05) che legge `editions` e fa l'embed Heyzine. Questo sprint **alimenta** quella pagina, non la tocca.
- `components/edition/heyzine-embed.tsx`, `components/edicola/*` — già esistenti, non toccare.
- `lib/supabase/server.ts` — server client (anon, RLS attiva). Esiste.
- `supabase/migrations/` — nel repo solo `001`, `002`, `003`. **Il DB live è più avanti del repo**: la vista `public_editions` e l'hardening RLS sono live ma non versionati come migration. Per questo sprint: la migration `004` la versioni nel repo E la applichi sul DB live. Non assumere che il repo migrations sia la verità: la verità è il DB `editorkit-prod` live.
- 5 publications già a DB con `slug`: `giornale-di-lecco`, `giornale-di-merate`, `giornale-di-monza`, `giornale-di-vimercate`, `il-canavese` (verifica i valori reali con una SELECT prima di scrivere il backfill — vedi §7.1, NON inventare slug).

---

## 3. OWNER

**Kimi Code.** Orchestratore verifica i criteri di accettazione su staging.

---

## 4. PRE-REQUISITI

- Supabase `editorkit-prod` (id `mhqwhyvzmqmajifmewqe`) live e healthy.
- Credenziali Heyzine (Paolo le mette negli env Netlify + locale, NON committate):
  - `HEYZINE_CLIENT_ID` = `877c907b0a7d03e8` (già negli env Netlify secondo workspace — verificare presenza)
  - `HEYZINE_API_KEY` = (Paolo fornisce; per il pilota la conversione REST funziona col solo `client_id`, ma l'env va predisposto)
- `ADMIN_EMAILS` (nuovo env): lista email separate da virgola autorizzate all'area admin. Paolo fornisce i valori (almeno `paolo@csflab.it` + `test1@studiocesareo.it` per il test).
- Repo locale `C:\001-Sviluppo\editorkit\` esistente.
- 5 publications pilota già a DB (Sprint 05x).

---

## 5. CARTELLA DI LAVORO

Path assoluti, NON cambiare working directory.

- Migration: `C:\001-Sviluppo\editorkit\supabase\migrations\004_onboarding_ingestion.sql`
- Codice: sotto `C:\001-Sviluppo\editorkit\` rispettando la struttura esistente (`lib/`, `app/`, ecc.).

---

## 6. DELIVERABLE

1. Migration `004_onboarding_ingestion.sql` (versionata nel repo **e** applicata al DB live).
2. Storage bucket privato `editions-pdf` creato su `editorkit-prod`.
3. `lib/ingestion/heyzine.ts` — wrapper chiamata Heyzine REST.
4. `lib/ingestion/run-ingestion.ts` — **core source-agnostic** (il contratto riusabile).
5. `app/api/ingestion/run/route.ts` — wrapper HTTP del core (service_role, per retry e per il futuro webhook).
6. `lib/admin/upload-edition.ts` — server action: il driver manuale.
7. `app/admin/onboarding/page.tsx` + gate admin (layout o check server-side).
8. Aggiornamento `.env.local.example` con le nuove variabili (senza valori reali).
9. Output `[KIMI BACKEND SPRINT 06 — DONE]` nel formato §10.

---

## 7. SPECIFICHE DETTAGLIATE

### 7.1 Migration `004_onboarding_ingestion.sql`

Prima di scrivere il backfill: esegui `SELECT id, name, slug FROM publications ORDER BY name;` sul DB live e usa gli `slug` e gli `id` REALI. NON inventare. Se gli slug reali differiscono da quelli ipotizzati in §2, usa i reali e segnalalo nell'output.

```sql
-- ========================================
-- SPRINT 06 — ONBOARDING / INGESTION
-- ========================================

-- 1. ops_slug su publications (decisione #35)
--    mappa testata -> directory-slug OPS, per agganciare l'ingestione FTP futura
ALTER TABLE publications ADD COLUMN IF NOT EXISTS ops_slug TEXT;
COMMENT ON COLUMN publications.ops_slug IS 'Slug directory lato OPS/VNP, per ingestione automatica futura';

-- Backfill 5 testate pilota — SOSTITUIRE gli slug FlipKit con quelli REALI letti dal DB
UPDATE publications SET ops_slug = 'lecco'     WHERE slug = 'giornale-di-lecco';
UPDATE publications SET ops_slug = 'merate'    WHERE slug = 'giornale-di-merate';
UPDATE publications SET ops_slug = 'monza'     WHERE slug = 'giornale-di-monza';
UPDATE publications SET ops_slug = 'vimercate' WHERE slug = 'giornale-di-vimercate';
UPDATE publications SET ops_slug = 'canavese'  WHERE slug = 'il-canavese';

-- 2. Stato di lavorazione sulle editions (modella anche il futuro flusso async)
DO $$ BEGIN
  CREATE TYPE edition_processing_status AS ENUM ('pending', 'converting', 'ready', 'failed');
EXCEPTION WHEN duplicate_object THEN null; END $$;

ALTER TABLE editions
  ADD COLUMN IF NOT EXISTS processing_status edition_processing_status NOT NULL DEFAULT 'ready',
  ADD COLUMN IF NOT EXISTS processing_error TEXT,
  ADD COLUMN IF NOT EXISTS source_type TEXT NOT NULL DEFAULT 'manual',
  ADD COLUMN IF NOT EXISTS pdf_storage_path TEXT;

COMMENT ON COLUMN editions.processing_status IS 'pending->converting->ready|failed. ready = flipbook pronto';
COMMENT ON COLUMN editions.source_type IS 'manual | ftp | webhook — driver che ha originato il record';
COMMENT ON COLUMN editions.pdf_storage_path IS 'path nel bucket privato editions-pdf';

-- Nota: default 'ready' per non rompere le righe esistenti (già pubblicate in Sprint 05x).
-- I nuovi record da pipeline partono espliciti a 'pending' (vedi §7.4).

-- 3. La vista pubblica edicola NON deve esporre edizioni non pronte.
--    Verifica la definizione attuale di public_editions sul DB live.
--    DEVE filtrare is_published = true AND processing_status = 'ready'.
--    Se la vista non filtra processing_status, ridefinirla aggiungendo il filtro,
--    mantenendo invariate le colonne esposte (NON esporre flipbook_id/heyzine_url).
--    Se la definizione live è ambigua o diversa da quanto atteso: FERMARSI e segnalare (decisione #32).
```

Note migration:

- Il default `processing_status = 'ready'` serve solo a non rompere le righe già pubblicate. Tutti i record creati dalla pipeline (§7.4) settano esplicitamente lo stato; non affidarsi al default per i nuovi.
- Lo Storage bucket NON si crea in SQL. Vedi §7.2.
- Applica la migration al DB live e committa il file nel repo.

### 7.2 Storage bucket `editions-pdf`

- Bucket **privato** (non public) su `editorkit-prod`.
- Creazione: via Supabase dashboard Storage **oppure** via Management/Storage API con service_role. Documenta nell'output quale metodo hai usato.
- Convenzione path oggetto: `<publication_slug>/<YYYY-MM-DD>.pdf` (es. `giornale-di-lecco/2026-05-09.pdf`). Idempotente: ricaricare la stessa data sovrascrive.
- Heyzine deve poter scaricare il PDF: si usa una **signed URL** generata server-side con TTL ampio (`expiresIn = 7200`, 2h — molto sopra i 53s di conversione, con margine per retry). Mai rendere pubblico il bucket.
- Il PDF resta in Storage come sorgente immutabile di record anche dopo la conversione (il flipbook è ospitato da Heyzine, ma teniamo l'originale).

### 7.3 `lib/ingestion/heyzine.ts` — wrapper conversione

Funzione pura sulla chiamata HTTP, nessuna dipendenza da Supabase o Next.

```
convertPdfToFlipbook(input: {
  pdfPublicUrl: string   // signed URL leggibile da Heyzine
  title: string
}): Promise<{
  flipbookId: string
  heyzineUrl: string
  thumbnailUrl: string
  numPages: number | null
  aspectRatio: number | null
}>
```

Implementazione:

- `POST https://heyzine.com/api1/rest`, `Content-Type: application/json`.
- Body: `{ pdf: pdfPublicUrl, client_id: process.env.HEYZINE_CLIENT_ID, title, prev_next: true, d: 1 }`.
- Risposta attesa: `{ url, thumbnail, pdf, id, meta: { num_pages, aspect_ratio } }`.
  - `flipbookId` = `id` della risposta.
  - `heyzineUrl` = `url` della risposta (atteso `https://heyzine.com/flip-book/<id>.html`).
  - `thumbnailUrl` = costruito come `https://cdnc.heyzine.com/flip-book/cover/<flipbookId>.jpg` (derivato dall'id, NON usare per forza il campo `thumbnail` della risposta — l'id-derived è il pattern validato #28).
  - `numPages` = `meta.num_pages` se presente, altrimenti `null`.
  - `aspectRatio` = `meta.aspect_ratio` se presente, altrimenti `null`.
- Timeout fetch esplicito a 80s (conversione max osservata 53s + margine).
- Se la risposta non ha un `id` valido, o HTTP non-2xx, o timeout → lancia errore con messaggio leggibile (verrà salvato in `processing_error`). NON ritentare dentro questa funzione (il retry è responsabilità del chiamante).
- `HEYZINE_API_KEY` non è richiesta dalla conversione REST (memo workspace: la conversione funziona col solo `client_id`). Leggila dall'env se presente per uso futuro ma non bloccare se assente.

### 7.4 `lib/ingestion/run-ingestion.ts` — CORE SOURCE-AGNOSTIC (il contratto)

Questo è il cuore architetturale dello sprint. Vincolo: questo modulo **non deve sapere nulla** di come il PDF è arrivato. Niente riferimenti a "admin", "upload", `FormData`, request HTTP, sessione utente. Prende un `editionId` di una riga già esistente in `editions` (creata da un driver qualunque) e la porta a `ready` o `failed`.

```
runIngestion(editionId: string): Promise<{
  ok: boolean
  status: 'ready' | 'failed'
  error?: string
}>
```

Algoritmo:

1. Con client **service_role** (RLS non deve ostacolare scrittura backend), legge la riga `editions` per `editionId`. Se assente → ritorna `failed` con errore "edition not found".
2. Setta `processing_status = 'converting'`.
3. Legge `pdf_storage_path`. Genera una signed URL dal bucket `editions-pdf` (TTL 7200s).
4. Recupera il titolo: `editions.title` se presente, altrimenti `"<nome publication> — <edition_date>"` (join su `publications`).
5. Chiama `convertPdfToFlipbook({ pdfPublicUrl, title })`.
6. In caso di successo: aggiorna la riga con `heyzine_flipbook_id`, `heyzine_url`, `thumbnail_url`, `num_pages`, `aspect_ratio`, `processing_status = 'ready'`, `is_published = true`, `published_at = now()`, `processing_error = NULL`.
7. In caso di errore: `processing_status = 'failed'`, `processing_error = <messaggio>`, `is_published = false`. Ritorna `failed`.
8. È **idempotente per retry**: richiamare `runIngestion` su una riga `failed` o `ready` ri-tenta la conversione e sovrascrive i campi flipbook. Non crea righe nuove.

Il client service_role qui NON passa per `lib/supabase/server.ts` (quello è anon+RLS). Crea un client dedicato con `SUPABASE_SERVICE_ROLE_KEY` (es. `lib/supabase/admin.ts` se non esiste già — crealo, non riusare il server client utente).

### 7.5 `app/api/ingestion/run/route.ts` — wrapper HTTP del core

- Route handler `POST`. Body JSON `{ editionId: string }`.
- `export const maxDuration = 90` (Next.js / Netlify functions: la conversione Heyzine dura fino a 53s, serve margine).
- Protezione: header `x-ingestion-token` confrontato con `process.env.INGESTION_TOKEN` (nuovo env, Paolo lo fornisce — segreto condiviso interno). Se non combacia → 401. Questo rende la route riusabile dal futuro webhook prepress senza dipendere dalla sessione browser.
- Chiama `runIngestion(editionId)`, ritorna lo stato in JSON.
- Nessun accesso ai cookie/sessione: è una route service-to-service.

### 7.6 Gate admin

- Le pagine sotto `/admin/**` sono accessibili solo se: utente loggato (sessione Supabase) **E** `user.email` ∈ `ADMIN_EMAILS` (split su virgola, trim, case-insensitive).
- Implementazione server-side: check in un `app/admin/layout.tsx` (server component) che legge la sessione via `lib/supabase/server.ts` e fa `redirect('/login')` o `notFound()` se non autorizzato. NON affidarsi solo al middleware.
- NON aggiungere logica admin nel middleware esistente (Sprint 03) se non strettamente necessario; se serve aggiungere `/admin` al matcher protetto, fallo in modo additivo senza toccare la logica `/dashboard` esistente.

### 7.7 Driver manuale: `lib/admin/upload-edition.ts` + `app/admin/onboarding/page.tsx`

Server action `uploadEditionAction(formData)`:

1. Verifica gate admin server-side (riusa la stessa logica del §7.6; non fidarsi del solo layout).
2. Legge da `formData`: `publicationId` (select), `editionDate` (`YYYY-MM-DD`, valida con regex `^\d{4}-\d{2}-\d{2}$`), `title` (opzionale), `file` (PDF, obbligatorio, `application/pdf`, max ragionevole es. 100MB).
3. Con client service_role: carica il file su `editions-pdf` al path `<publication_slug>/<editionDate>.pdf` (upsert = sovrascrivi).
4. Upsert riga `editions` su conflitto `(publication_id, edition_date)`: setta `title`, `pdf_storage_path`, `source_type = 'manual'`, `processing_status = 'pending'`, `is_published = false`. Recupera l'`editionId`.
5. Chiama `runIngestion(editionId)` e **attende** il risultato (la action è server-side, `export const maxDuration = 90` dove serve).
6. Ritorna alla pagina lo stato finale (`ready` con link flipbook + thumbnail, oppure `failed` con `processing_error`).

Pagina `app/admin/onboarding/page.tsx` (UI minimale, utilitaria, non è uno sprint di design):

- Form: dropdown publication (query `publications` ordinata per nome), input date, input text titolo opzionale, input file PDF, bottone "Carica e converti".
- Durante la submit: stato di attesa esplicito — testo tipo "Conversione in corso, può richiedere ~60 secondi. Non chiudere la pagina." (la conversione è sincrona dentro la action).
- Esito ok: messaggio verde + thumbnail + link "Apri flipbook" + link alla pagina edicola `/edizione/<slug>/<data>`.
- Esito ko: messaggio rosso con `processing_error` + bottone "Riprova" che ri-chiama l'azione di ingestione su quell'`editionId` (può colpire `/api/ingestion/run` con l'`INGESTION_TOKEN`, oppure una action dedicata che richiama `runIngestion`).
- Testi user-facing in italiano. Niente librerie UI nuove: usa i componenti `components/ui/*` esistenti (button, input, label, card).

### 7.8 Env

Aggiorna `.env.local.example` (valori placeholder, NON reali):

```
HEYZINE_CLIENT_ID=
HEYZINE_API_KEY=
ADMIN_EMAILS=
INGESTION_TOKEN=
```

Paolo inserirà i valori reali in `.env.local` e nel pannello Netlify. Verifica che `HEYZINE_CLIENT_ID` sia già presente negli env Netlify (workspace lo dà per esistente); se manca, segnalalo, non bloccare lo sviluppo locale.

### 7.9 Commit

```
git -C C:\001-Sviluppo\editorkit add .
git -C C:\001-Sviluppo\editorkit commit -m "feat: onboarding testate + pipeline ingestion source-agnostic (sprint 06)"
git -C C:\001-Sviluppo\editorkit push origin main
```

Auto-deploy Netlify attivo: il push triggera il build.

---

## 8. CRITERI DI ACCETTAZIONE

- [ ] Migration `004` applicata sul DB live + file committato nel repo.
- [ ] `publications.ops_slug` valorizzato sulle 5 testate pilota con gli slug OPS reali (`lecco/merate/monza/vimercate/canavese` o equivalenti verificati).
- [ ] `editions` ha `processing_status`, `processing_error`, `source_type`, `pdf_storage_path`.
- [ ] `public_editions` non espone edizioni con `processing_status != 'ready'` o `is_published = false`, e continua a NON esporre `heyzine_flipbook_id`/`heyzine_url`.
- [ ] Bucket privato `editions-pdf` esiste; non è public.
- [ ] **Test pipeline reale**: caricato via `/admin/onboarding` un PDF reale di UNA testata pilota → record `editions` creato → `processing_status = 'ready'` → `heyzine_flipbook_id` valorizzato → thumbnail `cdnc.heyzine.com/flip-book/cover/<id>.jpg` raggiungibile → l'edizione compare in edicola e si sfoglia da utente entitled (pagina Sprint 05 invariata, ora alimentata).
- [ ] `/admin/onboarding` accessibile SOLO da email in `ADMIN_EMAILS` loggata; da non-admin loggato → redirect/404; da non loggato → redirect login.
- [ ] **Vincolo source-agnostic verificato**: `lib/ingestion/run-ingestion.ts` non importa nulla di Next request/FormData/sessione e non contiene le stringhe `admin`/`upload`/`formData`. È invocabile dato il solo `editionId`. (Criterio architetturale, non negoziabile — vedi §7.4.)
- [ ] Retry: forzato un fallimento (es. path PDF inesistente) → riga `failed` con `processing_error` → "Riprova" su PDF valido → riga torna `ready`. Nessuna riga duplicata.
- [ ] `npm run build` passa. Push su `main` → auto-deploy Netlify ready → tutto sopra funziona su staging.
- [ ] Nessun segreto committato (Heyzine key, service_role, ingestion token, admin emails reali).

---

## 9. NOTE PER L'AGENTE

- **Lingua**: codice/commenti come da convenzione repo; commenti brevi in italiano; testi user-facing in italiano.
- **RISCHIO #1 — timeout funzione Netlify vs conversione 53s.** La conversione Heyzine è sincrona e dura fino a 53s. `export const maxDuration = 90` va messo sulla route `/api/ingestion/run` e dove gira la server action di upload. **Se in test su staging la funzione va in timeout perché il piano Netlify del progetto ha un limite funzioni inferiore a 90s: FERMATI e segnala a Paolo.** NON improvvisare un workaround (no code creativo, no spostamenti di architettura non concordati). Il pattern async/polling è già modellato dai campi `processing_status` ed è il piano B, ma si adotta solo su decisione esplicita.
- **Service_role qui è corretto e necessario.** La spec Sprint 05 diceva "no service role": valeva per quello sprint (lettura entitlement utente via RLS). Qui scriviamo `editions` e Storage da backend: la RLS blocca gli insert utente, quindi il core ingestione e l'upload usano service_role con un client dedicato (`lib/supabase/admin.ts`), separato dal server client utente. Questo NON è una contraddizione su cui fermarsi: è risolto qui dall'autore della spec.
- **Decisione #32**: se trovi una contraddizione REALE interna a questa spec (due sezioni che si escludono), FERMATI e segnala a Paolo. Non scegliere il default da solo. Esempio tipico: definizione di `public_editions` sul DB live diversa da quella attesa in §7.1.
- **Confine architetturale (§7.4)**: è il senso dello sprint. `run-ingestion.ts` è il contratto che il futuro driver FTP/webhook riuserà identico. Tienilo puro. L'upload manuale è solo il primo chiamante.
- **NON toccare** la pagina edizione Sprint 05, i componenti edicola, l'auth/middleware (se non l'aggiunta additiva del gate `/admin`). Questo sprint *alimenta* l'edicola esistente, non la riscrive.
- **NO Stripe, NO paywall reale, NO checkout**: fuori scope, parcheggiati.
- **Slug**: leggi gli slug reali dal DB prima del backfill. Non inventare. Se diversi dalle ipotesi, usa i reali e segnalalo.
- **In caso di errore bloccante** (build rotto, conversione che non torna `id`, RLS che blocca il backend nonostante service_role, timeout di cui sopra): FERMARE e segnalare. Niente pivot non concordati.

---

## 10. OUTPUT ATTESO PER PAOLO

```
[KIMI BACKEND SPRINT 06 — DONE]

Repo: https://github.com/Paolocesareo/editorkit (commit <sha>)
Auto-deploy Netlify: build state ready
URL staging: https://editorkit-staging.netlify.app

Migration:
- supabase/migrations/004_onboarding_ingestion.sql (committata + applicata su editorkit-prod)
- ops_slug backfill: lecco/merate/monza/vimercate/canavese (slug reali letti dal DB: <conferma>)

Storage:
- bucket privato editions-pdf creato via <dashboard|API>

File creati:
- lib/ingestion/heyzine.ts
- lib/ingestion/run-ingestion.ts (core source-agnostic)
- lib/supabase/admin.ts (service_role client)
- app/api/ingestion/run/route.ts (maxDuration 90)
- lib/admin/upload-edition.ts (driver manuale)
- app/admin/onboarding/page.tsx
- app/admin/layout.tsx (gate ADMIN_EMAILS)
- .env.local.example aggiornato

Test eseguiti:
- Upload PDF reale testata pilota -> editions ready -> flipbook + thumbnail OK: ✅
- Edizione visibile in edicola e sfogliabile da entitled: ✅
- Gate admin (admin OK / non-admin bloccato / anon -> login): ✅
- public_editions non espone non-ready: ✅
- Vincolo source-agnostic run-ingestion (no admin/upload/formData): ✅
- Retry da failed a ready, nessun duplicato: ✅

Env da settare su Netlify (Paolo): ADMIN_EMAILS, INGESTION_TOKEN, HEYZINE_API_KEY (HEYZINE_CLIENT_ID verificato presente: <si/no>)

Aperture/segnalazioni: <eventuali stop o slug diversi dal previsto, oppure "nessuna">

Pronto per Sprint 07 (admin entitlement).
```

---

*Sprint 06 — Versione 1 — 16 maggio 2026 (sera)*
