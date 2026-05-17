# Sprint P1 — Ricerca full-text archivio edizioni

> Spec per agente (Claude Code / Kimi Code). Repo spec: `Paolocesareo/editorkit-specs`. Codice: `Paolocesareo/editorkit` (privato).
> Owner proposto: backend (estrazione+indice) Kimi Code, frontend (UI ricerca) Claude Code. Orchestratore verifica su staging.
> Origine: capability P1 della backend-surface VNP (`Paolocesareo/Paolo/progetti/flipkit-backend-surface.md`, v0.6).

---

## 1. PERCHÉ

VNP ha una ricerca full-text sull'intero archivio edizioni (dal 2016), alimentata dal testo che la sua pipeline di ingestione estrae dal PDF. È una capability ad alto valore d'uso per il lettore di un quotidiano/settimanale locale ("cerca il nome del mio paese", "cerca una persona") ed è **indipendente dal motore di rendering** (Heyzine): non dipende da come si sfoglia, dipende solo dal testo indicizzato.

FlipKit oggi NON ha questo. La pipeline `ingest-edition` (Sprint 06b) fa solo PDF → Heyzine → record `editions`. Non estrae il testo, non c'è indice, non c'è ricerca.

P1 colma questo divario. È stata scelta come prima mossa perché: alto valore, Heyzine-compatibile, **zero dipendenze** dalla risposta prepress di Roberto e dalle decisioni di bivio (assi 1/2/3).

---

## 2. SCOPE — COSA ENTRA E COSA NO

### Entra
1. **Estrazione testo per pagina** in fase di ingestione: ogni edizione processata produce, oltre al flipbook Heyzine, il testo grezzo di ogni pagina.
2. **Indice di ricerca** su Supabase (Postgres full-text search), per edizione e per pagina.
3. **API di ricerca** server-side che rispetta l'entitlement (RLS): un utente trova risultati solo nelle testate/edizioni a cui ha diritto, più ciò che è pubblicamente visibile in edicola secondo le regole già esistenti.
4. **UI di ricerca** nel frontend: una barra di ricerca che restituisce edizione + pagina + snippet, con link alla pagina dell'edizione.
5. **Backfill** delle edizioni già caricate (le 5 testate pilota già in staging) tramite ri-processo del solo testo, senza ri-convertire su Heyzine.

### NON entra (fuori scope, parcheggiato)
- Ricerca per articolo / hotspot (è P2, contingente alla risposta prepress).
- Evidenziazione del termine *dentro* il flipbook Heyzine (Heyzine non lo espone; al massimo si apre la pagina giusta).
- Filtri avanzati tipo VNP (frase esatta / tutti i termini / intervallo personalizzato): in questo sprint solo ricerca a termini con AND implicito. Gli operatori avanzati vanno nel parking lot.
- Ricerca su PDF scansionati/immagine senza testo (OCR): fuori scope, vedi §6 rischio.

---

## 3. BIVIO DA SEGNALARE — NON DECIDERE IN AUTONOMIA

> Regola progetto #32: se la spec presenta un nodo non risolto, l'agente FERMA e segnala all'orchestratore, non sceglie da solo.

**Nodo: dove e quando si estrae il testo.** Tre opzioni con implicazioni diverse. La spec NON sceglie. L'agente deve proporre la sua raccomandazione tecnica con motivazione e **attendere conferma orchestratore prima di implementare**.

- **Opzione A — dentro la Edge Function `ingest-edition` esistente.** Il testo si estrae nello stesso flusso async che già converte su Heyzine. Pro: un solo punto, niente nuova infrastruttura. Contro: il limite runtime/memoria della Edge Function su PDF da 100+MB è già stato un muro noto (memo #46/#47, Sprint 06b); aggiungere l'estrazione testo nello stesso processo può riproporlo.
- **Opzione B — step separato post-ingestione.** Una seconda funzione/job che parte quando l'edizione è `ready`, scarica il PDF dallo Storage, estrae il testo, popola l'indice. Pro: isola il rischio runtime, fallibile e ritentabile senza toccare il flipbook. Contro: secondo processo da orchestrare e monitorare.
- **Opzione C — estrazione lato client al momento dell'upload admin.** Il browser dell'admin estrae il testo prima dell'upload e lo manda insieme. Pro: zero carico server. Contro: fragile, dipende dal browser, non funziona per il backfill né per l'ingestione automatica futura (#40/#41) — la scarto in partenza ma la elenco per completezza.

**Raccomandazione attesa dall'agente:** valutare A vs B alla luce dei memo Sprint 06b sui limiti di piattaforma e proporre. Indizio non vincolante: B disaccoppia il rischio, coerente con la scelta source-agnostic #40. Ma decide l'orchestratore.

Un secondo nodo minore, sempre da segnalare se emerge: **quale libreria di estrazione testo** regge i PDF reali OPS (impaginati, multi-colonna, 100+MB). Da validare su un PDF reale pilota, non assumere.

---

## 4. MODELLO DATI

Nuova tabella (migration numerata progressiva, allineata alla sequenza esistente — verificare l'ultimo numero applicato, NON assumere 009):

```
edition_pages_text
├── id                uuid pk
├── edition_id        uuid fk → editions(id) on delete cascade
├── page_number       int            (1-based)
├── content           text           (testo grezzo della pagina)
├── tsv               tsvector        (generated, lingua 'italian', da content)
└── created_at        timestamptz default now()

indice GIN su tsv
indice btree su (edition_id, page_number)
unique (edition_id, page_number)
```

Note:
- `tsv` come colonna **generata** (`GENERATED ALWAYS AS (to_tsvector('italian', content)) STORED`) — niente trigger manuali.
- Configurazione full-text **italiano** (`'italian'`): stemming e stopword IT. Verificare che l'istanza Postgres Supabase abbia la config `italian` (default sì).
- `editions` resta invariato. Nessuna modifica a `subscriptions`/`publications`.

### RLS — punto critico, non negoziabile
`edition_pages_text` deve avere RLS che **riusa la stessa logica di entitlement già attiva su `editions`**. Un utente non deve poter cercare (e quindi inferire contenuti) dentro edizioni a cui non ha diritto. Pattern: policy che fa join su `editions` e applica lo stesso predicato della RLS esistente. NON reinventare la regola di accesso: agganciarsi a quella in essere (decisione architettura "RLS fa il filtro entitlement, niente codice custom"). Se la regola di visibilità pubblica edicola (vista `public_editions`, `is_published AND processing_status='ready'`) e quella entitlement divergono, **segnalare il conflitto** prima di scrivere la policy.

---

## 5. API DI RICERCA

Endpoint server-side (route handler Next.js, server-side, mai query diretta dal client al DB):

```
GET /api/search?q={termini}&publication={slug?}&from={YYYY-MM-DD?}&to={YYYY-MM-DD?}&page={n?}

Risposta:
{
  query: string,
  total: number,
  results: [
    {
      publication: { slug, name },
      edition: { id, date, slug },
      page_number: int,
      snippet: string,        // ts_headline su content, termine evidenziato
      url: string             // link alla pagina edizione già esistente
    }
  ]
}
```

Regole:
- Query con `plainto_tsquery('italian', q)` o `websearch_to_tsquery('italian', q)` — preferire `websearch_to_tsquery` (gestisce naturalmente più termini, virgolette base). Segnalare se la versione Postgres Supabase non la supporta.
- Ranking con `ts_rank` o `ts_rank_cd`, ordine per rilevanza poi data decrescente.
- Snippet con `ts_headline('italian', content, query, 'MaxFragments=2, MinWords=5, MaxWords=20')`.
- Paginazione server-side (limit/offset o keyset), default 20 risultati.
- L'entitlement NON si filtra in codice applicativo: passa per la RLS. La query gira con il contesto utente, la RLS taglia. Verificare con utente di test entitled vs non-entitled che i risultati cambino di conseguenza.
- Rate limiting minimo sulla route (anti-abuso scraping dell'archivio), riusare il pattern esistente se presente, altrimenti segnalare assenza e proporre.

---

## 6. ESTRAZIONE TESTO

- Input: il PDF sorgente già in Supabase Storage (bucket `editions-pdf`), lo stesso che alimenta Heyzine. Niente nuova sorgente.
- Output: una riga `edition_pages_text` per pagina con `content` = testo estratto.
- Pagine senza testo estraibile (PDF immagine/scansione): inserire comunque la riga con `content` vuoto/null, **non** fallire l'intera edizione. Loggare il conteggio pagine senza testo: se un'edizione ha 0 testo su tutte le pagine, marcarla per revisione (l'edizione resta pubblicata e sfogliabile, semplicemente non ricercabile) e segnalarlo in output.
- OCR: **fuori scope P1.** Annotare nel parking lot "OCR per edizioni scansione" come sprint futuro contingente (rilevante solo se i PDF reali OPS risultano essere immagine — da verificare sul pilota, non assumere).
- Idempotenza: ri-processare un'edizione già indicizzata deve fare upsert (cancella e re-inserisce le sue righe), non duplicare. Serve per il backfill e per le ribattute (#O MacroHandler-like).

---

## 7. BACKFILL

- Le 5 testate pilota già in staging hanno edizioni `ready` senza testo indicizzato.
- Fornire un comando/funzione di backfill che itera le edizioni `ready`, scarica il PDF, estrae, popola l'indice — **senza toccare Heyzine** (no ri-conversione, no nuovo flipbook_id).
- Deve essere ri-eseguibile (idempotente, §6).
- Verifica di accettazione: dopo backfill, una ricerca per un termine noto presente in una specifica edizione pilota (es. un titolo di testata o un toponimo certo) restituisce quella edizione/pagina.

---

## 8. UI FRONTEND

- Barra di ricerca accessibile dall'edicola (header o pagina dedicata `/cerca`).
- Input termini + opzionale: filtro testata (dropdown dalle testate esistenti) + intervallo date.
- Risultati: lista con testata, data edizione, numero pagina, snippet con termine evidenziato, click → pagina edizione esistente (`/edizione/[publication-slug]/[date]`, eventualmente con ancora alla pagina se la pagina edizione lo supporta; se non lo supporta, aprire l'edizione e basta — NON inventare deep-link nel flipbook).
- Stato vuoto ("nessun risultato") e stato non-loggato gestiti: un utente non loggato vede risultati solo per ciò che è pubblicamente visibile secondo le regole edicola esistenti; il resto è dietro il paywall/login già in essere (NON costruire nuovo paywall, riusare il flusso Sprint 05).
- Stile coerente con l'edicola FlipKit esistente (brand pubblico FlipKit, decisione #29). Niente nuovo design system.

---

## 9. CRITERI DI ACCETTAZIONE (verifica orchestratore su staging — #48)

1. Migration applicata, `edition_pages_text` esiste con RLS attiva e policy agganciata all'entitlement di `editions`.
2. Backfill eseguito sulle 5 testate pilota: ogni edizione `ready` con PDF testuale ha righe pagina; edizioni senza testo loggate, non fallite.
3. `GET /api/search?q=...` restituisce risultati con snippet e link corretti.
4. **Test entitlement**: utente `test1@studiocesareo.it` (sub su tutte) trova risultati su tutte le testate pilota; un utente senza sub NON ottiene risultati su testate non sue (solo ciò che è pubblicamente visibile). Verificato a DB + da UI.
5. UI di ricerca raggiungibile dall'edicola, risultato cliccabile porta alla pagina edizione esistente.
6. Ri-eseguire il backfill non duplica righe (idempotenza).
7. Nessuna regressione su edicola, pagina edizione, embed Heyzine, auth (Sprint 05/05x/07 invariati).
8. Migration committata in `editorkit` e indice spec aggiornato in `editorkit-specs` (riallineare anche il disallineamento repo↔DB noto: migration 005/006/007 — vedi parking lot workspace, da sistemare in questo giro se tocca le migration).

---

## 10. FUORI SCOPO / PARKING LOT (da annotare, non sviluppare)

- Operatori di ricerca avanzati (frase esatta, OR, tutti/almeno-un-termine) come VNP.
- Deep-link alla pagina dentro il flipbook Heyzine (Heyzine non lo espone in modo affidabile).
- Ricerca per articolo/hotspot → è P2, contingente risposta prepress (Roberto).
- OCR per edizioni PDF-immagine → sprint futuro contingente, solo se i PDF reali OPS lo richiedono.
- Ricerca cross-archivio storico OPS pre-FlipKit (le edizioni vecchie su VNP non sono nostre finché non migrate) → fuori scope, è zona Sprint 10+.
- Analytics sulle ricerche (cosa cercano i lettori) → potenziale valore editore, parking lot.

---

## 11. NOTA DI PROCESSO PER L'AGENTE

- Spec con nodo aperto al §3: **non scegliere A/B/C da solo. Proporre e attendere.** (#32)
- Non auto-dichiarare "done": la verifica è dell'orchestratore contro i §9 su staging. (#48)
- Numero migration: verificare l'ultimo applicato sul progetto Supabase `editorkit-prod`, non assumere. Esiste un disallineamento noto repo↔DB sulle migration 005/006/007: leggere il parking lot del workspace prima di numerare.
- Brand pubblico = FlipKit (UI), interno = editorkit (repo/Supabase/path). Decisione #29.
- PDF reali OPS sono grossi (testato 110MB): validare la libreria di estrazione su un PDF reale pilota prima di dare per buono l'approccio.

---

*Spec P1 v1 — 17 maggio 2026 — ricerca full-text archivio. Origine: backend-surface VNP v0.6 capability P1. Da committare in editorkit-specs e referenziare nell'indice.*
