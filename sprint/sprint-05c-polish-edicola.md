# Sprint 05c — Polish UI Edicola (modello OPS a 2 livelli) + brand FlipKit

> Spec per Claude Code. Trasformare l'edicola pubblica dal grid "1 card per testata" (Sprint 05b) al modello OPS Media a 2 livelli: rullino "ultime edizioni" in alto + una riga per testata con date picker. Footer fisso. Etichette testata blu. Densità più compatta. In più: sostituire la stringa di brand visibile da "EditorKit" a "FlipKit".

---

## 1. OBIETTIVO

Replicare la struttura dell'edicola OPS Media (https://edizionidigitali.netweek.it/dmedia/newsstand): non più un grid piatto, ma **due livelli**:

- **Livello 1 — Rullino "Ultime edizioni"**: striscia orizzontale scrollabile con le edizioni più recenti di tutte le testate, frecce di scorrimento sx/dx.
- **Livello 2 — Righe per testata**: una riga per ogni testata attiva. A sinistra l'etichetta blu col nome testata, a destra una striscia orizzontale delle sue edizioni (più recente prima) + un date picker per saltare a una data specifica.

Più: footer fisso in fondo alla pagina, densità delle copertine più compatta, brand visibile = **FlipKit**.

A fine sprint l'edicola assomiglia strutturalmente a quella OPS, ma con l'animazione pagina Heyzine (che è meglio della loro).

Niente Stripe, niente paywall a tempo, niente toolbar sfogliatore: quelli sono Sprint 05d e 09. Qui si lavora solo sulla pagina edicola e sul brand.

---

## 2. CONTESTO

Workspace: https://github.com/Paolocesareo/Paolo/blob/master/progetti/editorkit.md

Sprint precedenti chiusi: 03, 03b, 05, 05b. La home `/` oggi è il grid piatto di Sprint 05b (1 card = ultima edizione per testata) con `EditionCoverCard`.

**Dati DB attuali** (non hardcodare conteggi, fetchare sempre dinamicamente):

- Più publications attive (testate locali OPS Media: Lecco, Vimercate, Monza, Merate, Canavese e altre se presenti).
- Più edizioni per testata (non più una sola: c'è storico, va sfruttato per il date picker).
- Vista `public.public_editions` già pronta: espone `id, publication_id, edition_date, edition_number, title, num_pages, thumbnail_url, published_at`. Esclude `heyzine_flipbook_id` e `heyzine_url` (contenuti protetti da RLS sulla tabella `editions`). La vista è leggibile da `anon` e `authenticated`.
- Pattern thumbnail Heyzine: `https://cdn.heyzine.com/flip-book/<flipbook_id>/thumb-1.jpg`, già valorizzato in `editions.thumbnail_url`.

**Nota di riconciliazione (per orchestratore, non per l'agente):** il date picker per testata rende l'edicola stessa l'accesso allo storico. La pagina archivio dedicata `/testata/[slug]` (vecchio Sprint 07) sarà ridimensionata o eliminata: non è preoccupazione di Claude Code in questo sprint.

---

## 3. OWNER

**Claude Code**

---

## 4. PRE-REQUISITI

- Sprint 05b chiuso (edicola grid + `EditionCoverCard` + `/edizione/[slug]/[date]` funzionanti)
- Vista `public.public_editions` in DB (già fatto)
- Auto-deploy Netlify attivo da `main`

---

## 5. CARTELLA DI LAVORO

`C:\001-Sviluppo\editorkit\` — path assoluti, no `cd`.

---

## 6. DELIVERABLE

1. `app/page.tsx` rifatto: edicola a 2 livelli (rullino + righe per testata)
2. Nuovo componente client `components/edicola/edition-rail.tsx`: striscia orizzontale scrollabile con frecce sx/dx
3. Nuovo componente client `components/edicola/edition-date-picker.tsx`: date picker che naviga a `/edizione/[slug]/[date]`, abilita solo le date che hanno un'edizione
4. `EditionCoverCard` con variante compatta (prop `size?: "default" | "compact"`, default `compact` per l'edicola)
5. Nuovo componente `components/layout/site-footer.tsx`: footer fisso
6. `app/layout.tsx` aggiornato: monta il footer fisso + padding bottom per non coprire il contenuto
7. Brand visibile → **FlipKit**: header, footer, `<title>`/metadata, qualsiasi copy user-facing
8. Commit pushato su `Paolocesareo/editorkit` `main` → auto-deploy

---

## 7. SPECIFICHE DETTAGLIATE

### 7.1 Brand swap (fare per primo, è banale e tocca file che modifichi comunque)

Sostituire la stringa visibile `EditorKit` → `FlipKit` in TUTTI i punti user-facing:

- `components/layout/header.tsx` — il logo testuale (oggi `EditorKit`)
- `app/layout.tsx` — `metadata.title` / `metadata.description`
- Qualsiasi titolo, claim o copy che dica "EditorKit"

**NON toccare**: nomi repo, id progetto Supabase, nomi env var, path cartella, identificatori interni, nomi file/componenti. Solo testo che un utente legge a schermo.

### 7.2 EditionCoverCard — variante compatta

`components/edicola/edition-cover-card.tsx`: aggiungere prop opzionale `size?: "default" | "compact"` (default `"compact"`).

- `compact`: card più stretta, padding ridotto (`p-2`), testo `text-xs`, pensata per stare in una striscia orizzontale. Larghezza fissa contenuta (es. `w-[150px]` su desktop, `w-[120px]` su mobile) così il rail scorre.
- `default`: comportamento attuale di Sprint 05b (per retrocompatibilità se usato altrove).

Aspect ratio copertina resta 3:4. `<Image unoptimized>` come in 05b.

### 7.3 EditionRail — striscia orizzontale con frecce

`components/edicola/edition-rail.tsx` — **client component**.

Props:

```ts
type RailItem = {
  publicationName: string;
  publicationSlug: string;
  editionDate: string;        // YYYY-MM-DD
  editionTitle?: string | null;
  editionNumber?: string | null;
  thumbnailUrl?: string | null;
};

type Props = {
  items: RailItem[];
  ariaLabel: string;
};
```

Comportamento:

- Contenitore `flex` orizzontale con `overflow-x-auto`, scroll snap, scrollbar nascosta (`scrollbar-width: none` + webkit).
- Due bottoni freccia (sx/dx) sovrapposti ai bordi: click → `scrollBy` di circa una "pagina" di larghezza visibile (`ref.current.clientWidth * 0.8`). Usare `lucide-react` `ChevronLeft` / `ChevronRight` (già disponibile nel progetto).
- Le frecce si nascondono o si disabilitano agli estremi (opzionale ma gradito: `scrollLeft === 0` e fine scroll).
- Ogni item è una `EditionCoverCard size="compact"`.
- Accessibile: `role="region"` + `aria-label={ariaLabel}`, bottoni con `aria-label` "Scorri indietro"/"Scorri avanti".

### 7.4 EditionDatePicker — salta a una data

`components/edicola/edition-date-picker.tsx` — **client component**.

Props:

```ts
type Props = {
  publicationSlug: string;
  availableDates: string[];   // YYYY-MM-DD, solo date che hanno un'edizione
};
```

Comportamento:

- Input `<input type="date">` nativo (semplice, niente librerie).
- `min` = data più vecchia in `availableDates`, `max` = data più recente.
- Su `onChange`: se la data scelta è in `availableDates` → `router.push('/edizione/{slug}/{date}')`. Se NON è in `availableDates` → messaggio inline gentile ("Nessuna edizione per questa data") e nessuna navigazione.
- `useRouter` da `next/navigation`.
- Etichetta accanto: "Vai a una data".

> Nota: l'input date nativo non sa disabilitare singoli giorni. Va bene così: validiamo on-change contro `availableDates`. Niente librerie calendario in questo sprint.

### 7.5 Pagina edicola — 2 livelli

`app/page.tsx` — server component, sostituire integralmente. `export const revalidate = 60;`

Logica fetch:

1. Supabase server client.
2. Publications attive: `from("publications").select("id, name, slug").eq("is_active", true).order("name")`.
3. Per ogni publication, fetch edizioni dalla **vista**: `from("public_editions").select("edition_date, title, edition_number, thumbnail_url").eq("publication_id", p.id).order("edition_date", { ascending: false }).limit(30)`.
4. **Livello 1 (rullino)**: prendere, tra tutte le edizioni fetchate, le ~12 più recenti in assoluto (merge + sort per `edition_date` desc, slice 12). Passarle a `EditionRail` con `ariaLabel="Ultime edizioni"`.
5. **Livello 2**: per ogni publication con almeno un'edizione, una riga:
   - Etichetta blu a sinistra col nome testata (vedi 7.6).
   - `EditionRail` con le edizioni della testata, `ariaLabel="Edizioni di {nome}"`.
   - `EditionDatePicker` con `availableDates` = tutte le `edition_date` di quella testata.
6. Empty state se nessuna publication ha edizioni (riusare il pattern cortese di 05b).

Header pagina: titolo "Edicola" + sottotitolo breve. Mantenere `container mx-auto px-4 py-8`.

### 7.6 Etichetta blu testata

Il nome testata in testa a ogni riga del Livello 2 deve essere un'etichetta blu (pattern OPS): badge/pill con sfondo blu, testo bianco, font semibold, angoli arrotondati. Usare un blu coerente col tema (Tailwind `bg-blue-600 text-white`, o token tema se esiste un primary blu). Deve restare leggibile e non urlato.

### 7.7 Footer fisso

`components/layout/site-footer.tsx` — server component statico.

- `position: fixed; bottom: 0; left: 0; right: 0;` (Tailwind `fixed inset-x-0 bottom-0`), `z-index` sopra il contenuto ma sotto eventuali modali, bordo superiore, sfondo pieno (non trasparente, deve coprire).
- Contenuto minimale, una riga: a sinistra "FlipKit", a destra `© {anno} CSF Lab` + eventuali link placeholder ("Termini", "Privacy") inerti per ora (`<span>` o `href="#"`).
- Altezza contenuta (es. `h-12`).

`app/layout.tsx`: montare `<SiteFooter />` dopo il contenuto e aggiungere `pb-16` (o pari all'altezza footer + margine) al wrapper del contenuto perché il footer fisso non copra l'ultima riga di copertine.

### 7.8 Densità

L'edicola deve risultare più compatta del grid arioso di 05b: copertine più piccole (variante `compact`), gap ridotti (`gap-3`), righe ravvicinate ma respirabili. Obiettivo: vedere più testate e più edizioni senza scroll infinito, come fa OPS.

### 7.9 Commit

```
git -C C:\001-Sviluppo\editorkit add .
git -C C:\001-Sviluppo\editorkit commit -m "feat: edicola 2 livelli (rullino + righe testata + date picker) + footer fisso + brand FlipKit (sprint 05c)"
git -C C:\001-Sviluppo\editorkit push origin main
```

---

## 8. CRITERI DI ACCETTAZIONE

- [ ] `npm run dev` e `npm run build` senza errori
- [ ] Brand visibile = "FlipKit" su header, footer, titolo tab/metadata. Nessun "EditorKit" visibile a schermo
- [ ] Nessun nome interno cambiato (repo, supabase id, env, path, file): grep di sicurezza
- [ ] `/` da non loggato mostra: rullino "Ultime edizioni" in alto + una riga per ogni testata attiva con edizioni
- [ ] Le frecce sx/dx del rullino scorrono la striscia
- [ ] Ogni riga testata ha etichetta blu col nome + date picker
- [ ] Date picker: data valida → naviga a `/edizione/{slug}/{date}`; data senza edizione → messaggio gentile, nessuna navigazione
- [ ] Click su una copertina → pagina edizione corretta (paywall se non loggato, embed se entitled — invariato da 05/05b)
- [ ] Footer fisso visibile in fondo, non copre l'ultima riga di copertine (padding bottom corretto)
- [ ] Densità più compatta del grid 05b (verifica visiva: più copertine a parità di viewport)
- [ ] Empty state se nessuna edizione in DB
- [ ] Auto-deploy Netlify completato, tutto ok anche su staging
- [ ] `/dashboard` continua a funzionare (può restare il grid di 05b, NON è scope di 05c — non romperla)

---

## 9. NOTE PER L'AGENTE

- Usare **sempre** la vista `public.public_editions`, mai la tabella `editions` diretta (RLS la blocca per anon).
- Hostname immagini `cdn.heyzine.com`: mantenere la soluzione già adottata in 05b (`unoptimized` o `remotePatterns`). Non reintrodurre il problema.
- Lingua: italiano per tutto il testo user-facing.
- Niente librerie nuove: date picker = input nativo, frecce = `lucide-react` già presente.
- NON toccare middleware, schema DB, flusso auth, pagina edizione, paywall.
- `/dashboard` fuori scope: non modificarla, ma non romperla (se cambi `EditionCoverCard` aggiungendo `size`, default deve preservare il rendering attuale dove è già usata).
- In caso di errore bloccante: FERMARE e segnalare a Paolo, non improvvisare.

---

## 10. OUTPUT ATTESO PER PAOLO

```
[CLAUDE CODE FRONTEND SPRINT 05C — DONE]

Repo: https://github.com/Paolocesareo/editorkit (commit <sha>)
Auto-deploy Netlify: build state ready
URL staging: https://editorkit-staging.netlify.app

File creati/modificati:
- app/page.tsx (rifatto: edicola 2 livelli)
- app/layout.tsx (footer fisso + padding + metadata FlipKit)
- components/edicola/edition-rail.tsx (nuovo)
- components/edicola/edition-date-picker.tsx (nuovo)
- components/edicola/edition-cover-card.tsx (prop size compact)
- components/layout/site-footer.tsx (nuovo)
- components/layout/header.tsx (brand FlipKit)

Test in locale:
- Rullino ultime edizioni + frecce: ok
- Righe per testata con etichetta blu: ok
- Date picker valido/non valido: ok
- Footer fisso non copre contenuto: ok
- Brand FlipKit visibile, nessun nome interno toccato: ok
- /dashboard non rotta: ok

Pronto per ritest end-to-end su staging.
```

---

*Sprint 05c — Versione 1 — 16 maggio 2026*
