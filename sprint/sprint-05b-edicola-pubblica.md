# Sprint 05b — Edicola Pubblica con Copertine

> Spec per Claude Code. Trasformare la home `/` nell'edicola pubblica: grid di copertine delle ultime edizioni di tutte le testate, visibile a chiunque (anche non loggato). Click sulla copertina porta alla pagina edizione (entitled vede l'embed, non entitled vede paywall — già fatto in Sprint 05).

---

## 1. OBIETTIVO

Replicare e migliorare il pattern di OPS Media (https://edizionidigitali.netweek.it/dmedia/newsstand): la home è un'**edicola pubblica** che mostra le copertine di tutte le testate attive, visibile da chiunque. Cliccando si va alla pagina edizione, dove scatta l'eventuale paywall.

Vantaggi del pattern: SEO, lead generation, conversione.

A fine sprint:

- `/` mostra grid di copertine, ognuna con thumbnail + nome testata + data ultima edizione
- Pagina visibile da non loggato (no auth required)
- Click su copertina → `/edizione/[slug]/[date]`
- Header aggiornato: link "Edicola" sempre visibile, "Le tue testate" visibile solo se loggato
- `/dashboard` aggiornata: usa le stesse copertine per coerenza visiva

Niente Stripe, niente paywall reale. La pagina edizione (Sprint 05) gestisce già il branching entitled/paywall.

---

## 2. CONTESTO

Workspace: https://github.com/Paolocesareo/Paolo/blob/master/progetti/editorkit.md (v0.6)

Sprint precedenti chiusi: 03, 03b, 05.

**Setup DB già pronto:**

- Vista `public.public_editions` creata: espone solo metadati copertina (id, publication_id, edition_date, edition_number, title, num_pages, thumbnail_url, published_at). **Esclude `heyzine_flipbook_id` e `heyzine_url`** (i contenuti restano protetti dalla RLS sulla tabella `editions`).
- La vista è leggibile da chiunque (`anon` e `authenticated`).
- Edition esistente in DB: Giornale di Monza n. 5 - 3 febbraio 2026 (publication_id `593aa772-ca25-483d-acef-8982a739d4b2`, edition_date `2026-02-03`).
- `editions.thumbnail_url` valorizzato a `https://cdn.heyzine.com/flip-book/a2ab342320/thumb-1.jpg`.

Pattern Heyzine thumbnail: `https://cdn.heyzine.com/flip-book/<flipbook_id>/thumb-1.jpg`. Per le edition future (Sprint 06 onboarding), questa URL può essere derivata automaticamente dal flipbook_id.

---

## 3. OWNER

**Claude Code**

---

## 4. PRE-REQUISITI

- Sprint 05 chiuso (pagina edizione + paywall placeholder funzionanti)
- Vista `public.public_editions` creata in DB (già fatto dall'orchestratore)
- Auto-deploy Netlify attivo

---

## 5. CARTELLA DI LAVORO

`C:\001-Sviluppo\editorkit\` — path assoluti, no `cd`.

---

## 6. DELIVERABLE

1. Nuova `app/page.tsx` (rifatta): edicola pubblica con grid copertine
2. Componente `components/edicola/edition-cover-card.tsx` riusabile
3. Header aggiornato con link "Edicola" sempre + "Le tue testate" se loggato
4. Dashboard rifatta per usare `EditionCoverCard` (coerenza visiva con edicola)
5. Stato vuoto sia per edicola (no edizioni in DB) sia per dashboard (no subscriptions)
6. Commit pushato su `Paolocesareo/editorkit` branch `main` → auto-deploy Netlify

---

## 7. SPECIFICHE DETTAGLIATE

### 7.1 Componente EditionCoverCard

**File**: `components/edicola/edition-cover-card.tsx`

Server component statico. Riusabile in edicola + dashboard.

Props:

```ts
type Props = {
  publicationName: string;
  publicationSlug: string;
  editionDate: string;       // YYYY-MM-DD
  editionTitle?: string | null;
  editionNumber?: string | null;
  thumbnailUrl?: string | null;
};
```

Layout:

- Card cliccabile (avvolta in `<Link>`) verso `/edizione/{publicationSlug}/{editionDate}`
- Top: immagine copertina (aspect ratio 3:4, verticale tipico giornale). Se `thumbnailUrl` null, fallback con icona/placeholder grigio.
- Bottom: nome testata in grassetto + data formattata in italiano + (se presente) numero edizione

Styling:

- Hover: leggera elevation o border highlight
- Width: card che riempie il grid item, l'immagine si adatta
- Overflow del titolo: line-clamp-1 sul nome testata, line-clamp-1 sulla data

```tsx
import Link from "next/link";
import Image from "next/image";

export function EditionCoverCard({
  publicationName,
  publicationSlug,
  editionDate,
  editionTitle,
  editionNumber,
  thumbnailUrl,
}: Props) {
  const formattedDate = new Date(editionDate).toLocaleDateString("it-IT", {
    day: "numeric",
    month: "long",
    year: "numeric",
  });

  return (
    <Link
      href={`/edizione/${publicationSlug}/${editionDate}`}
      className="group block overflow-hidden rounded-lg border bg-card transition hover:border-foreground/40 hover:shadow-md"
    >
      <div className="relative aspect-[3/4] w-full overflow-hidden bg-muted">
        {thumbnailUrl ? (
          <Image
            src={thumbnailUrl}
            alt={`Copertina ${publicationName} ${formattedDate}`}
            fill
            sizes="(max-width: 640px) 50vw, (max-width: 1024px) 33vw, 25vw"
            className="object-cover transition group-hover:scale-[1.02]"
            unoptimized
          />
        ) : (
          <div className="flex h-full w-full items-center justify-center text-xs text-muted-foreground">
            Copertina non disponibile
          </div>
        )}
      </div>
      <div className="space-y-1 p-3">
        <p className="line-clamp-1 font-semibold">{publicationName}</p>
        <p className="line-clamp-1 text-sm text-muted-foreground">
          {formattedDate}
          {editionNumber && ` · n. ${editionNumber}`}
        </p>
      </div>
    </Link>
  );
}
```

> ⚠️ `unoptimized` su `<Image>` evita di passare per il Next.js Image Optimizer (su Netlify costa risorse). Per ora va bene così — i thumbnail Heyzine sono già ottimizzati. Se in futuro serve, attiviamo `images.remotePatterns` in `next.config.ts`.

> Se `unoptimized` causa warnings o si preferisce, in alternativa configurare `next.config.ts`:
>
> ```ts
> images: { remotePatterns: [{ protocol: "https", hostname: "cdn.heyzine.com" }] }
> ```
>
> e togliere `unoptimized`. Decidere in base a quale sia più semplice da implementare senza rotture.

### 7.2 Pagina edicola (home)

**File**: `app/page.tsx` — sostituire integralmente.

Server component. Logica:

1. Crea server Supabase client
2. Fetch tutte le publications attive: `select id, name, slug from publications where is_active = true order by name`
3. Per ciascuna, fetch ultima edition dalla **vista** `public_editions`: `select * from public_editions where publication_id = ? order by edition_date desc limit 1`
4. Render grid

```tsx
import { createClient } from "@/lib/supabase/server";
import { EditionCoverCard } from "@/components/edicola/edition-cover-card";

export const revalidate = 60; // rigenera la pagina ogni minuto

export default async function HomePage() {
  const supabase = await createClient();

  const { data: publications } = await supabase
    .from("publications")
    .select("id, name, slug")
    .eq("is_active", true)
    .order("name");

  const cards = await Promise.all((publications ?? []).map(async (p) => {
    const { data: latest } = await supabase
      .from("public_editions")
      .select("edition_date, title, edition_number, thumbnail_url")
      .eq("publication_id", p.id)
      .order("edition_date", { ascending: false })
      .limit(1)
      .maybeSingle();
    return { publication: p, latest };
  }));

  const cardsWithEdition = cards.filter(c => c.latest !== null);

  return (
    <div className="container mx-auto px-4 py-8">
      <header className="mb-8">
        <h1 className="text-3xl font-bold tracking-tight">Edicola</h1>
        <p className="mt-1 text-muted-foreground">
          Le ultime edizioni delle testate locali. Clicca una copertina per sfogliare.
        </p>
      </header>

      {cardsWithEdition.length === 0 ? (
        <div className="rounded-lg border border-dashed p-12 text-center">
          <p className="text-muted-foreground">
            Nessuna edizione disponibile al momento. Torna a trovarci presto.
          </p>
        </div>
      ) : (
        <div className="grid grid-cols-2 gap-4 sm:grid-cols-3 lg:grid-cols-4 xl:grid-cols-5">
          {cardsWithEdition.map(({ publication, latest }) => (
            <EditionCoverCard
              key={publication.id}
              publicationName={publication.name}
              publicationSlug={publication.slug}
              editionDate={latest!.edition_date}
              editionTitle={latest!.title}
              editionNumber={latest!.edition_number}
              thumbnailUrl={latest!.thumbnail_url}
            />
          ))}
        </div>
      )}
    </div>
  );
}
```

### 7.3 Header aggiornato

**File**: `components/layout/header.tsx`

Mantieni client component (usa `useUser`). Aggiungi link "Edicola" sempre visibile, "Le tue testate" solo se loggato.

```tsx
"use client";

import Link from "next/link";
import { useUser } from "@/hooks/use-user";
import { LogoutButton } from "@/components/auth/logout-button";

export function Header() {
  const { user, loading } = useUser();

  return (
    <header className="border-b">
      <div className="container mx-auto flex h-14 items-center justify-between px-4">
        <Link href="/" className="font-semibold">
          EditorKit
        </Link>
        <nav className="flex items-center gap-4 text-sm">
          <Link href="/" className="text-muted-foreground hover:text-foreground">
            Edicola
          </Link>
          {loading ? null : user ? (
            <>
              <Link href="/dashboard" className="text-muted-foreground hover:text-foreground">
                Le tue testate
              </Link>
              <span className="hidden text-xs text-muted-foreground sm:inline">{user.email}</span>
              <LogoutButton />
            </>
          ) : (
            <Link href="/login">Accedi</Link>
          )}
        </nav>
      </div>
    </header>
  );
}
```

### 7.4 Dashboard rifatta con copertine

**File**: `app/dashboard/page.tsx` — sostituire la versione di Sprint 05.

Riusa `EditionCoverCard` per coerenza visiva. Layout simile all'edicola ma filtra solo le testate sottoscritte dall'utente.

```tsx
import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";
import { EditionCoverCard } from "@/components/edicola/edition-cover-card";

export default async function DashboardPage() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect("/login");

  const { data: subs } = await supabase
    .from("subscriptions")
    .select("publications_included, current_period_end")
    .eq("status", "active")
    .gt("current_period_end", new Date().toISOString());

  const publicationIds = Array.from(
    new Set((subs ?? []).flatMap(s => s.publications_included ?? []))
  );

  if (publicationIds.length === 0) {
    return (
      <div className="container mx-auto px-4 py-16">
        <h1 className="text-2xl font-bold">Le tue testate</h1>
        <p className="mt-4 text-muted-foreground">
          Non hai abbonamenti attivi. Vai all&apos;
          <a href="/" className="underline">edicola</a> per scoprire le testate disponibili.
        </p>
      </div>
    );
  }

  const { data: publications } = await supabase
    .from("publications")
    .select("id, name, slug")
    .in("id", publicationIds)
    .eq("is_active", true)
    .order("name");

  const cards = await Promise.all((publications ?? []).map(async (p) => {
    const { data: latest } = await supabase
      .from("public_editions")
      .select("edition_date, title, edition_number, thumbnail_url")
      .eq("publication_id", p.id)
      .order("edition_date", { ascending: false })
      .limit(1)
      .maybeSingle();
    return { publication: p, latest };
  }));

  const cardsWithEdition = cards.filter(c => c.latest !== null);

  return (
    <div className="container mx-auto px-4 py-8">
      <header className="mb-8">
        <h1 className="text-2xl font-bold">Le tue testate</h1>
        <p className="mt-1 text-muted-foreground">
          {cardsWithEdition.length === 1 ? "1 testata sottoscritta" : `${cardsWithEdition.length} testate sottoscritte`}.
          Clicca una copertina per sfogliare.
        </p>
      </header>

      <div className="grid grid-cols-2 gap-4 sm:grid-cols-3 lg:grid-cols-4 xl:grid-cols-5">
        {cardsWithEdition.map(({ publication, latest }) => (
          <EditionCoverCard
            key={publication.id}
            publicationName={publication.name}
            publicationSlug={publication.slug}
            editionDate={latest!.edition_date}
            editionTitle={latest!.title}
            editionNumber={latest!.edition_number}
            thumbnailUrl={latest!.thumbnail_url}
          />
        ))}
      </div>
    </div>
  );
}
```

> Il file `components/dashboard/publication-card.tsx` di Sprint 05 può essere **eliminato** (sostituito da `EditionCoverCard`). Pulizia.

### 7.5 Commit

```bash
git -C C:\001-Sviluppo\editorkit add .
git -C C:\001-Sviluppo\editorkit commit -m "feat: edicola pubblica con copertine + dashboard rifatta (sprint 05b)"
git -C C:\001-Sviluppo\editorkit push origin main
```

---

## 8. CRITERI DI ACCETTAZIONE

- [ ] `npm run dev` e `npm run build` partono senza errori
- [ ] `/` da non loggato mostra grid copertine (1 card visibile: Giornale di Monza)
- [ ] La copertina mostra l'immagine reale Heyzine (`cdn.heyzine.com/flip-book/a2ab342320/thumb-1.jpg`)
- [ ] Click su copertina → `/edizione/giornale-di-monza/2026-02-03`
- [ ] Da non loggato la pagina edizione mostra paywall placeholder
- [ ] Da loggato (con la subscription mock attiva) la pagina edizione mostra l'iframe Heyzine col PDF reale
- [ ] Header: "Edicola" sempre visibile, "Le tue testate" solo se loggato
- [ ] `/dashboard` da loggato mostra la stessa card-copertina di Giornale di Monza
- [ ] Empty state edicola: se non ci sono publications/editions, messaggio cortese (testabile in locale temporaneamente disattivando is_active)
- [ ] Empty state dashboard: se l'utente non ha subscription attive, messaggio + CTA verso `/`
- [ ] Auto-deploy Netlify completato → tutto funziona anche su staging
- [ ] File `components/dashboard/publication-card.tsx` rimosso (sostituito da EditionCoverCard)

---

## 9. NOTE PER L'AGENTE

- **Vista `public.public_editions`**: usare quella per fetchare i metadati copertina, NON la tabella `editions` direttamente. La vista non ha RLS (è pubblica) e quindi funziona anche per utenti non loggati.
- **Hostname Heyzine**: `cdn.heyzine.com`. Se Next.js/Image dà errori di hostname non whitelisted, scegliere una di queste due strade:
  - Soluzione A (più semplice): usare `unoptimized` sul componente `Image`
  - Soluzione B: aggiungere `cdn.heyzine.com` a `next.config.ts` `images.remotePatterns`
- **Lingua**: italiano per tutti i testi user-facing.
- **NO modifiche al middleware**: la home `/` è pubblica per default, non serve toccare niente.
- **NO modifiche allo schema DB**: tutto già pronto.
- **In caso di errore bloccante**: FERMARE e segnalare a Paolo.

---

## 10. OUTPUT ATTESO PER PAOLO

```
[CLAUDE CODE FRONTEND SPRINT 05B — DONE]

Repo: https://github.com/Paolocesareo/editorkit (commit <sha>)
Auto-deploy Netlify: ✅ build state ready
URL staging: https://editorkit-staging.netlify.app

File creati/modificati:
- app/page.tsx (rifatto: edicola pubblica)
- app/dashboard/page.tsx (rifatto: usa EditionCoverCard)
- components/edicola/edition-cover-card.tsx (nuovo)
- components/layout/header.tsx (aggiornato)
- components/dashboard/publication-card.tsx (rimosso)
- next.config.ts (eventuale aggiornamento images.remotePatterns)

Test eseguiti in locale:
- / da non loggato: grid con 1 copertina visibile (Giornale di Monza): ✅
- Thumbnail Heyzine renderizzato: ✅
- Click → /edizione/giornale-di-monza/2026-02-03: ✅
- Da non loggato la pagina edizione mostra paywall: ✅
- /dashboard da loggato mostra la stessa card: ✅
- Header con "Edicola" sempre, "Le tue testate" solo loggato: ✅

Pronto per ritest end-to-end su staging.
```

---

*Sprint 05b — Versione 1 — 30 aprile 2026*
