# Sprint 06b — Onboarding fix: upload diretto resumable + conversione async su Edge Function

> Spec per Kimi Code. Sprint correttivo di 06. Riusa il core di 06, cambia solo trasporto e runtime. Verifica orchestratore.

---

## 1. OBIETTIVO

Lo Sprint 06 ha consegnato il core corretto ma è morto al primo upload reale. Causa diagnosticata sul DB live: un PDF di edizione reale pesa **113MB**, il bucket era vuoto e nessuna riga `editions` creata → la server action è morta durante l'upload, prima di scrivere.

06b risolve i quattro muri, riusando `runIngestion`/`heyzine` come logica, cambiando **solo** dove passano i byte e dove gira la conversione:

- Il PDF non passa più dentro Netlify. Va **diretto dal browser a Supabase Storage** in upload resumable (TUS).
- La conversione Heyzine non gira più su Netlify. Gira su una **Supabase Edge Function** asincrona, con la UI in polling.

Non è un redesign del prodotto: è cambiare transport + runtime di una pipeline la cui logica è già scritta e valida.

---

## 2. CONTESTO

Workspace: https://github.com/Paolocesareo/Paolo/blob/master/progetti/editorkit.md
Spec 06 (riferimento, NON ripeterla): https://raw.githubusercontent.com/Paolocesareo/editorkit-specs/main/sprint/sprint-06-onboarding-testate.md

I quattro muri (diagnosi orchestratore, confermata su DB e Storage live):

- **A** — PDF 113MB dentro la server action: oltre il tetto ~6MB delle function Netlify. La via "PDF attraverso Netlify" è chiusa, non recuperabile con tweak.
- **A'** — Limite dimensione file Storage. Risolto a metà: l'orchestratore ha già messo `editions-pdf.file_size_limit = 209715200` (200MB) e `allowed_mime_types = ['application/pdf']`. Resta da alzare il **limite globale di progetto** (vedi §7.1, azione Paolo).
- **A''** — Un upload singolo da 113MB è inaffidabile. Serve upload **resumable (TUS)**, non shot singolo.
- **B** — Conversione Heyzine sincrona (53s+) oltre il timeout function Netlify (era RISCHIO #1 della spec 06).

Cosa resta valido di Sprint 06 e NON si rifà:

- Migration `004` applicata e verificata sul DB live (ops_slug, processing_status, source_type, pdf_storage_path, vista `public_editions` con filtro `ready`).
- Gate admin (`app/admin/layout.tsx` + ricontrollo nelle action) — funzionante, verificato in staging.
- Bucket privato `editions-pdf` con limite 200MB e mime PDF.
- Logica conversione `lib/ingestion/heyzine.ts` (pura) e `lib/ingestion/run-ingestion.ts` (core source-agnostic): la **logica** è corretta e si riusa. Cambia il runtime (vedi §7.4 / §7.7).

Decisione #32: se trovi contraddizioni interne a questa spec, FERMATI e segnala. Non scegliere da solo.

---

## 3. OWNER

**Kimi Code.** Orchestratore verifica su staging con un PDF reale pesante.

---

## 4. PRE-REQUISITI

- Migration `004` già applicata su `editorkit-prod` (fatto). NON riapplicare.
- Bucket `editions-pdf`: `file_size_limit = 200MB`, mime `application/pdf` (fatto dall'orchestratore via SQL).
- **Limite globale Storage di progetto**: Paolo lo alza nel dashboard Supabase (vedi §7.1) a ≥ 200MB. Senza questo, l'upload diretto viene comunque rifiutato a livello progetto.
- Env già su Netlify (verificati live in 06): `INGESTION_TOKEN`, `ADMIN_EMAILS`, `HEYZINE_CLIENT_ID`, `SUPABASE_SERVICE_ROLE_KEY`, `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `NEXT_PUBLIC_SITE_URL`.
- **Secrets Edge Function** (Paolo li setta via `supabase secrets set` o dashboard, NON committati): `HEYZINE_CLIENT_ID`, `INGESTION_TOKEN`. NB: Supabase **riserva il prefisso `SUPABASE_`** per i secret custom delle Edge Function: NON si può creare `SUPABASE_SERVICE_ROLE_KEY` a mano. Ma Supabase **inietta già di default** `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` nelle Edge Function: leggerli da `Deno.env` senza settare nulla. (Kimi ha usato un secret custom `SERVICE_ROLE_KEY`: accettabile, ma il default iniettato `SUPABASE_SERVICE_ROLE_KEY` è più pulito.) Lato **Netlify** invece NON c'è restrizione di prefisso: le route trigger/status devono leggere l'esistente `SUPABASE_SERVICE_ROLE_KEY` già configurato, non introdurre un nuovo env `SERVICE_ROLE_KEY`. La route trigger/run per invocare l'Edge Function ha bisogno solo di `INGESTION_TOKEN`, non della service_role.
- Repo locale `C:\001-Sviluppo\editorkit\`, path assoluti, no cd.

---

## 5. CARTELLA DI LAVORO

- Edge Function: `C:\001-Sviluppo\editorkit\supabase\functions\ingest-edition\index.ts`
- Codice Next: sotto `C:\001-Sviluppo\editorkit\` rispettando la struttura esistente.

---

## 6. DELIVERABLE

1. Edge Function Supabase `ingest-edition` (Deno) — esegue la conversione in background.
2. Server action ridotta: crea riga `pending` + restituisce signed upload URL. Zero byte PDF attraverso Netlify.
3. Upload resumable lato browser (TUS) verso Supabase Storage.
4. Route trigger sottile + route status (service_role) + UI polling.
5. `/api/ingestion/run` rifattorizzata a delega verso l'Edge Function (resta il contratto pubblico, decisione #41).
6. Aggiornamento `.env.local.example` se servono nuove variabili client.
7. Output `[KIMI BACKEND SPRINT 06b — DONE]` (formato §10).

---

## 7. SPECIFICHE DETTAGLIATE

### 7.1 Limite Storage globale (azione Paolo, pre-requisito da verificare)

Dashboard Supabase → Storage → Settings → "Upload file size limit": portarlo a **≥ 200MB**. Il limite per-bucket è già 200MB ma il globale di progetto, se inferiore, vince e blocca. Kimi: nella spec assumi che Paolo lo abbia fatto; nei test, se l'upload viene rifiutato con errore size a livello progetto, FERMATI e segnala (è azione Paolo, non un bug tuo).

### 7.2 Upload diretto browser → Supabase Storage via SINGLE PUT (no TUS)

**Cambio di rotta (orchestratore, evidenza definitiva).** Il TUS resumable è abbandonato: richiede INSERT su `storage.s3_multipart_uploads`, ma su questo progetto i ruoli `authenticated`/`anon` non hanno il privilegio (grant di default Supabase rimossi da hardening precedente). Le tabelle sono di proprietà di `supabase_storage_admin`; il ruolo cliente `postgres` non può ripristinare il grant (non owner, non membro, non superuser — muro di piattaforma, verificato). Migration 005/006 (policy su `storage.objects`, `public`, scoped al solo bucket, **senza SELECT**) restano valide e NECESSARIE per questo path: il PUT diretto scrive in `storage.objects` come ruolo utente e quelle policy lo autorizzano (verificato via prova SQL: insert objects come authenticated E anon su bucket `editions-pdf` → OK).

- Niente `tus-js-client`, niente endpoint `/upload/resumable`. Rimuovere la dipendenza se aggiunta.
- Il server genera una **signed upload URL**: `admin.storage.from('editions-pdf').createSignedUploadUrl(storagePath)` (vedi §7.3). Ritorna `{ signedUrl, token, path }`.
- Il browser carica il file con **un singolo PUT** alla signed URL: `supabase.storage.from('editions-pdf').uploadToSignedUrl(path, token, file, { contentType: 'application/pdf', upsert: true })`. Questo path è già stato verificato funzionante da Kimi (PUT diretto → 200). Scrive solo in `storage.objects`, niente tabelle multipart, niente muro grant.
- Bypassa Netlify (browser→Supabase diretto, nessun tetto 6MB) e il muro multipart. Limite globale Storage + bucket entrambi a 200MB ≥ 113MB.
- UI: barra/spinner di avanzamento durante il PUT (il file è grande, l'upload dura; mostrare stato "Caricamento in corso, non chiudere"). `uploadToSignedUrl` non espone progress granulare nativo: accettabile uno spinner indeterminato con messaggio chiaro; NON usare librerie nuove.
- Tradeoff accettato (pilota): un singolo PUT da ~113MB non ha resume se la connessione cade. Accettabile per il pilota operato da Paolo su connessione stabile. L'evoluzione architetturale corretta (sorgente PDF su Cloudflare R2 con presigned URL, già nell'architettura parcheggiata) è rimandata, NON in scope qui.

### 7.3 Server action ridotta (driver manuale, zero byte PDF su Netlify)

Sostituisce l'upload-dentro-la-action di 06. Nuova `prepareEditionUploadAction(formData)`:

1. Ricontrolla il gate admin server-side (come 06, non fidarsi del solo layout).
2. Legge metadati: `publicationId`, `editionDate` (`^\d{4}-\d{2}-\d{2}$`), `title?`. **Nessun file.**
3. Recupera `publications.slug`, compone `storagePath = <slug>/<editionDate>.pdf`.
4. Con client service_role: **upsert** riga `editions` su conflitto `(publication_id, edition_date)` — `processing_status='pending'`, `is_published=false`, `source_type='manual'`, `pdf_storage_path=storagePath`. Recupera `editionId`.
   - **FIX bug 06**: la stringa `onConflict` deve essere senza spazi: `'publication_id,edition_date'`. (In 06 era `'publication_id, edition_date'` con spazio: PostgREST non risolve il target e al re-upload dà duplicate key. Correggere.)
5. Genera signed upload URL: `admin.storage.from('editions-pdf').createSignedUploadUrl(storagePath, { upsert: true })`. Ritorna al client `{ editionId, storagePath, token, signedUrl }`.
6. NON converte qui. NON triggera qui. Ritorna e basta. Tempo di esecuzione: millisecondi, nessun rischio timeout, nessun PDF.

### 7.4 Edge Function `ingest-edition` (Deno, conversione async)

`supabase/functions/ingest-edition/index.ts`. È la nuova sede di esecuzione della logica di `run-ingestion` + `heyzine`. Porta la logica esistente in Deno (è quasi identica: `fetch` per Heyzine è già standard, `@supabase/supabase-js` ha build Deno via `https://esm.sh/@supabase/supabase-js@2`).

- Input: `POST` con body `{ editionId: string }`.
- Auth: header `x-ingestion-token` confrontato con il secret `INGESTION_TOKEN`. Mismatch → 401. (Stesso contratto del REST 06: riusabile dal futuro webhook prepress.)
- Esecuzione **in background**: usa `EdgeRuntime.waitUntil(promise)` per ritornare **202 subito** e proseguire la conversione fuori dal ciclo richiesta/risposta. Il client non aspetta i 53s+; fa polling (§7.5).
- Logica del task in background (identica nella sostanza a `runIngestion`):
  1. service_role client (da env secret). Legge riga `editions` per `editionId`. Assente → fine, niente da fare.
  2. `processing_status='converting'`, `processing_error=null`.
  3. Signed URL del PDF: `createSignedUrl(pdf_storage_path, 7200)`.
  4. Titolo: `editions.title` o nome publication.
  5. Chiama Heyzine REST (`https://heyzine.com/api1/rest`, body `{pdf, client_id, title, prev_next:true, d:1}`, timeout 120s — i 113MB possono richiedere più dei 53s validati su 72 pagine; alza il margine). Thumbnail derivata dall'id (`https://cdnc.heyzine.com/flip-book/cover/<id>.jpg`), pattern #28.
  6. Successo → aggiorna riga: flipbook_id, heyzine_url, thumbnail_url, num_pages, aspect_ratio, `processing_status='ready'`, `is_published=true`, `published_at=now()`, `processing_error=null`.
  7. Errore → `processing_status='failed'`, `processing_error=<messaggio>`, `is_published=false`.
- Deploy: `supabase functions deploy ingest-edition`. NB: l'Edge Function ha timeout wall-clock proprio (≈150s), sufficiente; nessun tetto Netlify.
- Idempotente per retry: richiamarla sullo stesso `editionId` ri-tenta e sovrascrive. Niente righe nuove.

### 7.5 Trigger + polling + UI

- **Trigger**: a upload TUS completato, il client chiama `POST /api/ingestion/trigger` (route Next sottile) con `{ editionId }`. La route: ricontrolla gate admin, poi invoca l'Edge Function `ingest-edition` con header `x-ingestion-token` (server-side, il token non arriva mai al browser), **senza attendere** la conversione (l'Edge Function risponde 202 subito grazie a `waitUntil`). La route ritorna 202 al client. Payload minimo, nessun rischio timeout Netlify.
- **Status/polling**: il client fa polling ogni 4s su `GET /api/ingestion/status?editionId=...`. La route ricontrolla gate admin e legge `processing_status`/`processing_error`/`heyzine_url`/`thumbnail_url` via **service_role** (la RLS su `editions` non lascerebbe leggere una riga non pubblicata all'utente admin). Ritorna lo stato.
- **UI** `app/admin/onboarding/page.tsx`: stati = idle → uploading (% TUS) → converting (polling, messaggio "Conversione in corso, può richiedere oltre un minuto") → ready (thumbnail + link flipbook + link edicola) | failed (errore + bottone "Riprova"). "Riprova" ri-chiama `/api/ingestion/trigger` sullo stesso `editionId` (la riga e il PDF esistono già). Niente reload forzato della pagina come workaround.
- Testi user-facing in italiano. Solo componenti `components/ui/*` esistenti, nessuna libreria UI nuova oltre `tus-js-client`.

### 7.6 `/api/ingestion/run` resta contratto pubblico, delega all'Edge Function

La route Netlify `app/api/ingestion/run/route.ts` di 06 NON deve più eseguire `runIngestion` in-process (è il muro B). Rifattorizzala a: valida `x-ingestion-token`, poi invoca l'Edge Function `ingest-edition` con lo stesso `editionId` e ritorna 202. Resta l'URL stabile del contratto di ingestione per il futuro webhook prepress (decisione #41), ma l'esecuzione vera è sull'Edge Function. `maxDuration` torna irrilevante (la route non aspetta più la conversione).

### 7.7 Sorgente unica di verità

Per evitare due implementazioni divergenti della stessa logica: la **logica canonica di conversione vive nell'Edge Function**. `lib/ingestion/run-ingestion.ts` e `lib/ingestion/heyzine.ts` NON sono l'esecuzione del flusso manuale; restano nel repo come riferimento ma vanno marcati superati con un commento in testa (`// SUPERATO da supabase/functions/ingest-edition — non è il path di esecuzione, vedi Sprint 06b`). Non cancellarli (storia/diff), non importarli più nel flusso vivo.

### 7.8 Commit

```
git -C C:\001-Sviluppo\editorkit add .
git -C C:\001-Sviluppo\editorkit commit -m "fix: upload diretto resumable + conversione async edge function (sprint 06b)"
git -C C:\001-Sviluppo\editorkit push origin main
```

Auto-deploy Netlify attivo. L'Edge Function va deployata a parte con `supabase functions deploy ingest-edition` (Paolo fornisce login/ref CLI Supabase se Kimi non li ha; altrimenti consegna il comando a Paolo che lo esegue, e dichiaralo nell'output).

---

## 8. CRITERI DI ACCETTAZIONE

- [ ] Limite globale Storage progetto ≥ 200MB (azione Paolo, verificata).
- [ ] Edge Function `ingest-edition` deployata e raggiungibile.
- [ ] `/admin/onboarding` da admin: form visibile, upload di un PDF **reale pesante (≈100MB+)** completa via SINGLE PUT a signed upload URL (no TUS), con spinner/stato, **senza** "This page couldn't load" e **senza** 403.
- [ ] Nessun byte di PDF passa da Netlify (verifica a codice: la server action non riceve `File`; l'upload è client→Supabase via signed URL PUT).
- [ ] Dopo upload: riga `editions` `pending`→`converting`→`ready`, `heyzine_flipbook_id` valorizzato, thumbnail `cdnc.heyzine.com/...jpg` raggiungibile, edizione visibile in `public_editions` e sfogliabile da utente entitled (pagina Sprint 05 invariata).
- [ ] Nessun timeout: la conversione gira sull'Edge Function, la UI fa polling, il browser non resta appeso 53s+.
- [ ] Gate admin invariato: non-admin → `/dashboard`, non loggato → `/login`.
- [ ] Re-upload stessa testata+data: la riga si **aggiorna**, niente duplicate key (fix `onConflict` senza spazi verificato).
- [ ] Retry da `failed` a `ready` senza righe duplicate.
- [ ] `/api/ingestion/run` con token valido + editionId reale → 202 e l'Edge Function processa (contratto pubblico preservato).
- [ ] `npm run build` passa. Push su `main` → deploy Netlify ready. Edge Function deployata.
- [ ] Nessun segreto committato.

---

## 9. NOTE PER L'AGENTE

- **Non reintrodurre il PDF dentro Netlify in nessuna forma.** È il muro A, è definitivo. Qualunque soluzione che faccia transitare i byte del PDF da una function Netlify è sbagliata per definizione, anche se "funziona" con un PDF piccolo di test. Il test di accettazione usa un PDF reale pesante apposta.
- **Niente TUS/resumable.** Su questo progetto i grant Supabase sulle tabelle multipart sono assenti e non ripristinabili dal ruolo cliente (muro piattaforma verificato dall'orchestratore). L'unico upload diretto utilizzabile è il **single PUT a signed upload URL**. Migration 005/006 sono già applicate dall'orchestratore e bastano per il PUT su `storage.objects`. Migration 007 (grant multipart) è inefficace e superata: ignorarla.
- **`EdgeRuntime.waitUntil`**: è il meccanismo per cui l'Edge Function risponde subito e converte in background. Se non disponibile/instabile nel runtime corrente, FERMATI e segnala a Paolo invece di tornare a una conversione sincrona che ricrea il muro B.
- **Decisione #32**: contraddizione interna alla spec → fermati e segnala, non scegliere.
- **Slug**: la migration 004 ha già fatto il backfill `ops_slug`. Non rifarlo.
- **Lingua**: commenti brevi in italiano, testi user-facing in italiano.
- **In caso di errore bloccante** (TUS che non autentica, Edge Function che non deploya, `waitUntil` assente, limite globale Storage non alzato): FERMARE e segnalare. Niente pivot non concordati, niente PDF dentro Netlify come ripiego.

---

## 10. OUTPUT ATTESO PER PAOLO

```
[KIMI BACKEND SPRINT 06b — DONE]

Repo: https://github.com/Paolocesareo/editorkit (commit <sha>)
Auto-deploy Netlify: build state ready
Edge Function: ingest-edition deployata (<da Kimi | comando consegnato a Paolo>)
URL staging: https://editorkit-staging.netlify.app

File creati/modificati:
- supabase/functions/ingest-edition/index.ts (nuovo, logica canonica)
- lib/admin/upload-edition.ts (ridotta: prepareEditionUploadAction, no file, fix onConflict)
- app/admin/onboarding/page.tsx (TUS upload + polling)
- app/api/ingestion/trigger/route.ts (nuovo)
- app/api/ingestion/status/route.ts (nuovo)
- app/api/ingestion/run/route.ts (delega all'Edge Function)
- lib/ingestion/run-ingestion.ts, lib/ingestion/heyzine.ts (marcati superati)
- package.json (tus-js-client)

Test eseguiti:
- Upload PDF reale ~<dimensione>MB via TUS, barra avanzamento, nessun errore pagina: ✅
- Nessun byte PDF da Netlify (verificato a codice): ✅
- pending->converting->ready, flipbook+thumbnail, visibile in edicola, sfogliabile: ✅
- Nessun timeout, polling UI: ✅
- Gate admin invariato: ✅
- Re-upload stessa data: aggiorna, no duplicate key (fix onConflict): ✅
- Retry failed->ready, no duplicati: ✅
- /api/ingestion/run delega all'Edge Function: ✅

Azioni Paolo richieste/fatte: limite globale Storage ≥200MB <fatto/da fare>, secrets Edge Function <set/da settare>

Aperture/segnalazioni: <eventuali stop, oppure "nessuna">

Pronto per verifica orchestratore con PDF reale pesante.
```

---

*Sprint 06b — Versione 1 — 16 maggio 2026 (sera) — fix architetturale onboarding: upload diretto resumable + conversione async Edge Function*
