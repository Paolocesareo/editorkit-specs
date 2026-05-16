# Sprint 05d — Pagina /account minimal + Toolbar sfogliatore "cambia edizione"

> Spec per Claude Code. Due deliverable indipendenti, stesso sprint perché entrambi piccoli e UI-only: (1) una pagina `/account` minimale protetta, (2) una toolbar custom sopra l'embed Heyzine nella pagina edizione, con selettore "cambia edizione" della stessa testata. Nessun Stripe, nessun paywall, nessuna estrazione multimedia (quella è Sprint 11).

---

## 1. OBIETTIVO

Replicare due elementi del modello OPS Media che oggi mancano:

- **Pagina `/account`**: l'utente loggato vede chi è, a quali testate ha accesso, e può uscire. Minimale: identità + entitlement + logout. NON billing (Stripe è parcheggiato), NON modifica dati.
- **Toolbar sfogliatore**: nella pagina edizione, sopra l'iframe Heyzine, una barra nostra con un selettore "Cambia edizione" che elenca le altre edizioni della **stessa testata** e ci naviga senza passare dall'edicola. Più un link di ritorno all'edicola.

A fine sprint l'utente entitled può saltare tra le edizioni di una testata restando nello sfogliatore, e ha una pagina account essenziale.

Fuori scope esplicito: estrazione articoli/multimedia (Sprint 11), controlli interni del viewer Heyzine (vista doppia, anteprima pagine: li dà già Heyzine), qualsiasi cosa Stripe.

---

## 2. CONTESTO

Workspace: https://github.com/Paolocesareo/Paolo/blob/master/progetti/editorkit.md

Sprint chiusi: 03, 03b, 05, 05b, 05c. Brand visibile = FlipKit (fatto in 05c, non rifarlo).

Stato rilevante:

- Pagina edizione: `app/edizione/[slug]/[date]/page.tsx`. Server component. Branca entitled → mostra iframe Heyzine; branca non-entitled → paywall placeholder (Sprint 05, NON toccare la logica di branching).
- Vista `public.public_editions`: espone `publication_id, edition_date, edition_number, title, num_pages, thumbnail_url`. Niente flipbook_id/heyzine_url (protetti da RLS). Leggibile da `anon` e `authenticated`. Usare SEMPRE questa per liste edizioni.
- Auth: `useUser()` (client) e `supabase.auth.getUser()` (server) già esistenti. `/dashboard` protetta esiste come pattern di riferimento per la protezione rotta. Componente `LogoutButton` esiste.
- Tabella `subscriptions`: `publications_included` (array di publication_id), `status`, `current_period_end`. Pattern di lettura entitlement già usato in `/dashboard` (Sprint 05b) — riusalo, non reinventarlo.
- `EditionCoverCard` esiste con prop `size` ("default" | "compact", default "default").

---

## 3. OWNER

**Claude Code**

---

## 4. PRE-REQUISITI

- Sprint 05c chiuso e live su staging
- Vista `public.public_editions` in DB (già)
- Auto-deploy Netlify attivo da `main`

---

## 5. CARTELLA DI LAVORO

`C:\001-Sviluppo\editorkit\` — path assoluti, no `cd`.

---

## 6. DELIVERABLE

1. `app/account/page.tsx` (nuovo): pagina protetta minimale
2. `components/sfogliatore/edition-toolbar.tsx` (nuovo): toolbar client con selettore cambia edizione + ritorno edicola
3. `app/edizione/[slug]/[date]/page.tsx` (modificato): monta la toolbar SOLO sulla branca entitled, sopra l'iframe Heyzine. La logica entitled/paywall di Sprint 05 resta identica.
4. `components/layout/header.tsx` (modificato): aggiungi link "Account" visibile solo se loggato
5. Commit su `Paolocesareo/editorkit` `main` → auto-deploy

---

## 7. SPECIFICHE DETTAGLIATE

### 7.1 Pagina /account

**File**: `app/account/page.tsx`. Server component. Protetta come `/dashboard`: se no user → `redirect("/login")`.

Contenuto, in italiano, sobrio:

- Titolo "Il tuo account"
- Email dell'utente loggato
- Sezione "Le tue testate": elenco dei nomi testata a cui l'utente ha accesso (deriva da `subscriptions` attive → `publications_included` → `publications.name`, stesso pattern di `/dashboard` 05b). Se nessuna: riga "Nessun abbonamento attivo" + link all'edicola.
- `LogoutButton` esistente
- NIENTE form, NIENTE campi modificabili, NIENTE billing. È una scheda in sola lettura.

Niente nuove dipendenze. Layout coerente col resto (`container mx-auto px-4 py-8`).

### 7.2 Toolbar sfogliatore

**File**: `components/sfogliatore/edition-toolbar.tsx` — **client component**.

Props:

```ts
type ToolbarEdition = { editionDate: string; label: string }; // label es. "30 aprile 2026" o "30 aprile 2026 · n. 5/2026"

type Props = {
  publicationName: string;
  publicationSlug: string;
  currentDate: string;            // YYYY-MM-DD dell'edizione aperta
  editions: ToolbarEdition[];     // tutte le edizioni della stessa testata, già ordinate desc
};
```

Layout: barra orizzontale sottile sopra l'iframe, sticky in cima al contenitore (`sticky top-0 z-10`), sfondo pieno, bordo inferiore.

- Sinistra: nome testata (in evidenza) + data edizione corrente.
- Centro/destra: `<select>` nativo "Cambia edizione" valorizzato con `editions`. L'opzione corrente è selezionata. `onChange` → `router.push('/edizione/{slug}/{date}')` con la data scelta. (Nativo, niente librerie.)
- Destra: link "Torna all'edicola" → `/`.
- Accessibile: `aria-label` sul select ("Cambia edizione di {publicationName}").

### 7.3 Aggancio nella pagina edizione

**File**: `app/edizione/[slug]/[date]/page.tsx`.

NON toccare la logica entitled vs paywall di Sprint 05. Solo:

- Sulla **branca entitled** (utente ha diritto, si renderizza l'iframe Heyzine): prima dell'iframe, fetch dalla vista `public_editions` tutte le edizioni della testata corrente (`select edition_date, edition_number, title where publication_id = <pub> order by edition_date desc`), costruisci l'array `editions` con `label` formattata in italiano, e monta `<EditionToolbar ... />` sopra l'iframe.
- Sulla **branca paywall**: NON montare la toolbar (l'utente non è entitled, non deve poter saltare tra edizioni).
- Il publication_id e lo slug sono già disponibili nella pagina (servono già per il branching). Riusa quei dati, non rifare query inutili.

### 7.4 Header

**File**: `components/layout/header.tsx`. Aggiungi voce "Account" → `/account`, visibile **solo se loggato** (stessa condizione con cui oggi mostri "Le tue testate"/dashboard). Non duplicare logica: stesso ramo `user ?`.

### 7.5 Commit

```
git -C C:\001-Sviluppo\editorkit add .
git -C C:\001-Sviluppo\editorkit commit -m "feat: pagina /account minimal + toolbar sfogliatore cambia edizione (sprint 05d)"
git -C C:\001-Sviluppo\editorkit push origin main
```

---

## 8. CRITERI DI ACCETTAZIONE

- [ ] `npm run dev` e `npm run build` senza errori
- [ ] `/account` da non loggato → redirect a `/login`
- [ ] `/account` da loggato → email + elenco testate accessibili + logout funzionante
- [ ] `/account` da loggato senza abbonamenti → messaggio + link edicola, nessun crash
- [ ] Pagina edizione, utente entitled: toolbar visibile sopra l'iframe, mostra testata + data corrente
- [ ] Il select "Cambia edizione" elenca le altre edizioni della stessa testata e ci naviga
- [ ] Pagina edizione, utente NON entitled (paywall): toolbar NON presente
- [ ] Logica entitled/paywall di Sprint 05 invariata (nessuna regressione su chi vede cosa)
- [ ] Header: voce "Account" solo se loggato
- [ ] `/`, `/dashboard`, edicola 05c: nessuna regressione
- [ ] Auto-deploy Netlify completato, verificato su staging

---

## 9. NOTE PER L'AGENTE

- Vista `public.public_editions` sempre, mai `editions` diretta.
- Niente librerie nuove: select e niente date-lib.
- NON toccare: middleware, schema DB, flusso auth, logica entitled/paywall di Sprint 05, brand (già FlipKit da 05c).
- Riusa i pattern esistenti (`/dashboard` per protezione rotta + lettura subscriptions, `LogoutButton`). Non reinventare l'entitlement.
- Solo stringhe visibili in italiano. Nessun nome interno/repo/env/path modificato.
- Errore bloccante → FERMATI e segnala, niente workaround improvvisati.
- Se una nota di questa spec sembra contraddirne un'altra, FERMATI e segnala a Paolo invece di scegliere in autonomia.

---

## 10. OUTPUT ATTESO PER PAOLO

```
[CLAUDE CODE FRONTEND SPRINT 05D — DONE]

Repo: https://github.com/Paolocesareo/editorkit (commit <sha>)
Auto-deploy Netlify: build state ready
URL staging: https://editorkit-staging.netlify.app

File creati/modificati:
- app/account/page.tsx (nuovo)
- components/sfogliatore/edition-toolbar.tsx (nuovo)
- app/edizione/[slug]/[date]/page.tsx (toolbar su branca entitled)
- components/layout/header.tsx (voce Account se loggato)

Test in locale:
- /account non loggato → /login: ok
- /account loggato: email + testate + logout: ok
- /account senza abbonamenti: messaggio + link: ok
- Toolbar su edizione entitled, select cambia edizione naviga: ok
- Toolbar assente su paywall: ok
- Nessuna regressione su edicola/dashboard/branching 05: ok
- npm run build: ok

Pronto per ritest end-to-end su staging.
```

---

*Sprint 05d — Versione 1 — 16 maggio 2026*
