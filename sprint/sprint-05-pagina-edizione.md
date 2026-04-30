# Sprint 05 — Pagina Edizione + Embed Heyzine + Dashboard Testate

> Spec per Claude Code. Pagina pubblica `/edizione/[publication-slug]/[date]` con embed Heyzine protetto da RLS Supabase, paywall placeholder per chi non è entitled, dashboard utente con lista testate sottoscritte.

---

## 1. OBIETTIVO

Mettere live il **cuore funzionale del prodotto**: l'utente loggato e con subscription attiva clicca una testata dalla dashboard, atterra sulla pagina di un'edizione, vede il PDF sfogliabile via embed Heyzine. L'utente senza subscription vede un placeholder "Abbonati per leggere".

A fine sprint:

- `/edizione/giornale-di-monza/2026-04-28` mostra l'iframe Heyzine se l'utente ha la subscription
- Stesso URL mostra placeholder paywall statico se l'utente non è entitled (o non è loggato)
- `/dashboard` mostra la lista delle testate sottoscritte dall'utente con link all'ultima edizione di ciascuna

Stripe e checkout NON sono in scope (parcheggiati). Il "click su Abbonati" del paywall placeholder è un bottone disabilitato/finto.

---

## 2. CONTESTO

Workspace: https://github.com/Paolocesareo/Paolo/blob/master/progetti/editorkit.md (v0.6)

Sprint precedenti chiusi:
- 03 (auth magic link end-to-end)
- 03b (deploy Netlify staging)

**Le policy RLS sono già configurate in modo perfetto per questo sprint:**

```
publications: chiunque legge le publications con is_active = true
editions: l'utente legge una edition SOLO se is_published = true
         AND ha una subscription attiva con publication_id nel publications_included
subscriptions: l'utente legge solo le proprie
```

Quindi tutta la logica di entitlement è server-side via SQL. Dal codice basta fare `SELECT * FROM editions WHERE ...` con il client Supabase server, e RLS fa il filtro: se l'utente non è entitled, la query torna 0 righe → render paywall.

**Seed data già creato in DB:**

- Publisher: OPS Media (id già in DB)
- Publication: "Giornale di Monza", slug `giornale-di-monza` (id `593aa772-ca25-483d-acef-8982a739d4b2`)
- Edition: 2026-04-28, "Edizione del 28 aprile 2026", num_pages 9, is_published true, heyzine_flipbook_id placeholder
- Subscription: utente test1@studiocesareo.it (`66bca43e-dcff-4a3e-9e53-56aa8a524122`), tier_2, attiva, include la publication

> ⚠️ Il `heyzine_flipbook_id` nel seed è un placeholder (`PLACEHOLDER_FLIPBOOK_ID`). L'orchestratore lo aggiorna con il valore reale di Heyzine prima del test finale. La pagina deve gestire qualsiasi flipbook_id, non hardcodarlo.

---

## 3. OWNER

**Claude Code**

---

## 4. PRE-REQUISITI

- Sprint 03 + 03b chiusi e funzionanti in staging
- `lib/supabase/server.ts` e `lib/supabase/client.ts` invariati da Sprint 02
- Seed data Supabase già pronto (vedi sopra)

---

## 5. CARTELLA DI LAVORO

`C:\001-Sviluppo\editorkit\` — path assoluti, no `cd`.

---

## 6. DELIVERABLE

1. Pagina dinamica `app/edizione/[publication-slug]/[date]/page.tsx` (server component)
2. Componente `components/edition/heyzine-embed.tsx` (iframe responsive)
3. Componente `components/edition/paywall-placeholder.tsx` (UI statica)
4. Pagina dashboard `app/dashboard/page.tsx` aggiornata: lista testate sottoscritte con card e link all'ultima edizione
5. Componente `components/dashboard/publication-card.tsx`
6. Stato vuoto dashboard se l'utente non ha subscription attive
7. Commit pushato su `Paolocesareo/editorkit` branch `main` → auto-deploy Netlify

---

## 7. SPECIFICHE DETTAGLIATE

### 7.1 Pagina edizione

**File**: `app/edizione/[publication-slug]/[date]/page.tsx`

Server component. Riceve i parametri di routing da Next.js 15 (props.params è Promise).

Comportamento:

1. Estrai `publicationSlug` e `date` dai params (date in formato `YYYY-MM-DD`)
2. Crea il server Supabase client
3. Fetch della **publication** via slug. Se non esiste o non è attiva → `notFound()`
4. Fetch della **edition** via `(publication_id, edition_date)`. La query usa il server client autenticato: RLS fa il filtro per entitlement.
5. Branching:
   - Edition trovata → render header (titolo testata + data + numero) + `<HeyzineEmbed flipbookId={...} url={...} />`
   - Edition NON trovata → controlla se l'edition esiste **per qualcun altro** con un client di servizio (vedi sotto). Se esiste ma RLS l'ha nascosta → `<PaywallPlaceholder publicationName={...} editionDate={date} />`. Se proprio non esiste → `notFound()`.

Per distinguere "edizione inesistente" da "edizione esistente ma non sei entitled" senza usare il service role (che non vogliamo esporre lato server route per motivi di sicurezza), si può:

**Strategia A (semplice, raccomandata)**: facciamo 2 query col client utente:

```ts
// 1. Esiste un'edition pubblicata per (publication_id, date)? (può essere RLS-filtered)
const { data: edition } = await supabase
  .from("editions")
  .select("id, title, edition_number, num_pages, heyzine_flipbook_id, heyzine_url")
  .eq("publication_id", publication.id)
  .eq("edition_date", date)
  .eq("is_published", true)
  .maybeSingle();

if (edition) {
  // Entitled: mostra embed
  return <ArticleView edition={edition} publication={publication} />;
}

// Non visto: o non esiste per niente, o esiste ma RLS l'ha bloccata.
// Mostriamo SEMPRE il paywall: se la publication esiste e la data è plausibile,
// proporre "abbonati" è la UX corretta (anche se non c'è quella edition specifica,
// l'utente è arrivato qui perché non è entitled o non è loggato).
return <PaywallPlaceholder publicationName={publication.name} editionDate={date} />;
```

**Decisione**: usiamo strategia A. Niente service role nel route. Non si fa la distinzione "edizione inesistente vs paywall": se non vedi l'edition, vedi il paywall. È una UX semantically corretta per questo flusso.

Eccezione `notFound()`: solo se la **publication** stessa non esiste o non è attiva (lì `is_active=true` è la sola policy).

### 7.2 Componente HeyzineEmbed

**File**: `components/edition/heyzine-embed.tsx`

Client component (oppure server, indifferente — nessuna interazione).

Riceve `flipbookId` e `heyzineUrl` (può essere null). Renderizza un iframe full-width responsive in aspect ratio adatto a un giornale (4:5 verticale di default, ma il container deve essere adattabile).

```tsx
type Props = {
  flipbookId: string;
  heyzineUrl: string | null;
};

export function HeyzineEmbed({ flipbookId, heyzineUrl }: Props) {
  // URL pattern Heyzine: prefer heyzineUrl if provided, otherwise build from id
  const src = heyzineUrl ?? `https://heyzine.com/flip-book/${flipbookId}.html`;

  return (
    <div className="w-full">
      <div className="relative w-full" style={{ aspectRatio: "4 / 5" }}>
        <iframe
          src={src}
          allowFullScreen
          allow="clipboard-write"
          className="absolute inset-0 h-full w-full rounded-lg border bg-muted"
          title="Sfogliatore edizione"
        />
      </div>
      <p className="mt-2 text-xs text-muted-foreground">
        Lo sfogliatore si apre a tutto schermo cliccando sull&apos;icona.
      </p>
    </div>
  );
}
```

### 7.3 Componente PaywallPlaceholder

**File**: `components/edition/paywall-placeholder.tsx`

Server component (statico, niente JS).

```tsx
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";

type Props = {
  publicationName: string;
  editionDate: string;
};

export function PaywallPlaceholder({ publicationName, editionDate }: Props) {
  const formattedDate = new Date(editionDate).toLocaleDateString("it-IT", {
    day: "numeric", month: "long", year: "numeric"
  });

  return (
    <Card className="mx-auto max-w-xl">
      <CardHeader>
        <CardTitle>Abbonati per leggere</CardTitle>
        <CardDescription>
          {publicationName} — edizione del {formattedDate}
        </CardDescription>
      </CardHeader>
      <CardContent className="space-y-4">
        <p className="text-sm text-muted-foreground">
          Per sfogliare questa edizione serve un abbonamento attivo a {publicationName}.
        </p>
        <Button disabled className="w-full">
          Abbonati (non ancora disponibile)
        </Button>
        <p className="text-xs text-muted-foreground">
          Il sistema di abbonamento è in fase di attivazione.
        </p>
      </CardContent>
    </Card>
  );
}
```

### 7.4 Layout pagina edizione

Header con: nome testata (link a `/dashboard`) + data + numero edizione (se presente).
Sotto, se entitled: l'embed. Sennò: il paywall placeholder.

Esempio scaffold di `page.tsx`:

```tsx
import { notFound } from "next/navigation";
import Link from "next/link";
import { createClient } from "@/lib/supabase/server";
import { HeyzineEmbed } from "@/components/edition/heyzine-embed";
import { PaywallPlaceholder } from "@/components/edition/paywall-placeholder";

type Props = {
  params: Promise<{ "publication-slug": string; date: string }>;
};

export default async function EditionPage({ params }: Props) {
  const { "publication-slug": slug, date } = await params;

  // Validazione date
  if (!/^\d{4}-\d{2}-\d{2}$/.test(date)) notFound();

  const supabase = await createClient();

  const { data: publication } = await supabase
    .from("publications")
    .select("id, name, slug, description")
    .eq("slug", slug)
    .eq("is_active", true)
    .maybeSingle();

  if (!publication) notFound();

  const { data: edition } = await supabase
    .from("editions")
    .select("id, title, edition_number, num_pages, heyzine_flipbook_id, heyzine_url, edition_date")
    .eq("publication_id", publication.id)
    .eq("edition_date", date)
    .eq("is_published", true)
    .maybeSingle();

  const formattedDate = new Date(date).toLocaleDateString("it-IT", {
    day: "numeric", month: "long", year: "numeric"
  });

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="mb-6">
        <Link href="/dashboard" className="text-sm text-muted-foreground hover:text-foreground">
          ← Torna alle tue testate
        </Link>
      </div>

      <header className="mb-6">
        <h1 className="text-3xl font-bold">{publication.name}</h1>
        <p className="mt-1 text-muted-foreground">
          {edition?.title ?? `Edizione del ${formattedDate}`}
          {edition?.edition_number && ` · n. ${edition.edition_number}`}
        </p>
      </header>

      {edition && edition.heyzine_flipbook_id ? (
        <HeyzineEmbed
          flipbookId={edition.heyzine_flipbook_id}
          heyzineUrl={edition.heyzine_url}
        />
      ) : (
        <PaywallPlaceholder
          publicationName={publication.name}
          editionDate={date}
        />
      )}
    </div>
  );
}
```

### 7.5 Dashboard aggiornata

**File**: `app/dashboard/page.tsx` — sostituire integralmente.

Server component. Logica:

1. Verifica utente (già fatto in Sprint 03, mantieni)
2. Fetch delle subscriptions attive dell'utente
3. Estrai gli array di publication IDs uniti da tutte le subscriptions attive
4. Fetch delle publications corrispondenti
5. Per ogni publication, fetch dell'**ultima** edition pubblicata (RLS filtra: vedi solo le entitled)
6. Render: una `<PublicationCard>` per ogni publication con link all'edizione

```tsx
import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";
import { PublicationCard } from "@/components/dashboard/publication-card";

export default async function DashboardPage() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect("/login");

  // Subscriptions attive
  const { data: subs } = await supabase
    .from("subscriptions")
    .select("publications_included, tier, current_period_end")
    .eq("status", "active")
    .gt("current_period_end", new Date().toISOString());

  const publicationIds = Array.from(new Set((subs ?? []).flatMap(s => s.publications_included ?? [])));

  if (publicationIds.length === 0) {
    return (
      <div className="container mx-auto px-4 py-16">
        <h1 className="text-2xl font-bold">Le tue testate</h1>
        <p className="mt-4 text-muted-foreground">
          Non hai abbonamenti attivi. Quando ti abbonerai a una testata,
          la troverai elencata qui.
        </p>
      </div>
    );
  }

  // Publications
  const { data: publications } = await supabase
    .from("publications")
    .select("id, name, slug, description")
    .in("id", publicationIds)
    .eq("is_active", true);

  // Ultima edition per ciascuna
  const cards = await Promise.all((publications ?? []).map(async (p) => {
    const { data: latest } = await supabase
      .from("editions")
      .select("edition_date, title, edition_number, num_pages")
      .eq("publication_id", p.id)
      .eq("is_published", true)
      .order("edition_date", { ascending: false })
      .limit(1)
      .maybeSingle();
    return { publication: p, latest };
  }));

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="mb-6 text-2xl font-bold">Le tue testate</h1>
      <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
        {cards.map(({ publication, latest }) => (
          <PublicationCard
            key={publication.id}
            publication={publication}
            latest={latest}
          />
        ))}
      </div>
    </div>
  );
}
```

### 7.6 Componente PublicationCard

**File**: `components/dashboard/publication-card.tsx`

Server component statico.

```tsx
import Link from "next/link";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";

type LatestEdition = {
  edition_date: string;
  title: string | null;
  edition_number: string | null;
  num_pages: number | null;
} | null;

type Props = {
  publication: { id: string; name: string; slug: string; description: string | null };
  latest: LatestEdition;
};

export function PublicationCard({ publication, latest }: Props) {
  const formattedDate = latest
    ? new Date(latest.edition_date).toLocaleDateString("it-IT", {
        day: "numeric", month: "long", year: "numeric"
      })
    : null;

  return (
    <Card>
      <CardHeader>
        <CardTitle>{publication.name}</CardTitle>
        {publication.description && (
          <CardDescription className="line-clamp-2">
            {publication.description}
          </CardDescription>
        )}
      </CardHeader>
      <CardContent className="space-y-3">
        {latest ? (
          <>
            <p className="text-sm text-muted-foreground">
              Ultima edizione: <strong>{formattedDate}</strong>
              {latest.edition_number && ` · n. ${latest.edition_number}`}
            </p>
            <Button asChild className="w-full">
              <Link href={`/edizione/${publication.slug}/${latest.edition_date}`}>
                Sfoglia
              </Link>
            </Button>
          </>
        ) : (
          <p className="text-sm text-muted-foreground">
            Nessuna edizione disponibile al momento.
          </p>
        )}
      </CardContent>
    </Card>
  );
}
```

### 7.7 Commit

```bash
git -C C:\001-Sviluppo\editorkit add .
git -C C:\001-Sviluppo\editorkit commit -m "feat: pagina edizione + embed heyzine + dashboard testate (sprint 05)"
git -C C:\001-Sviluppo\editorkit push origin main
```

L'auto-deploy Netlify è attivo: il push triggera il build automatico.

---

## 8. CRITERI DI ACCETTAZIONE

- [ ] `npm run dev` e `npm run build` partono senza errori
- [ ] `/dashboard` da loggato mostra "Giornale di Monza" come card con il bottone "Sfoglia"
- [ ] Click su "Sfoglia" porta a `/edizione/giornale-di-monza/2026-04-28`
- [ ] Su quella pagina, da loggato con subscription attiva, si vede l'iframe Heyzine (con il flipbook_id placeholder darà errore Heyzine, è atteso fino a quando l'orchestratore aggiorna il valore — la **struttura** della pagina è ok)
- [ ] Apertura di `/edizione/giornale-di-monza/2026-04-28` da non loggato → middleware Sprint 03 NON protegge (la rotta è pubblica) → la pagina si carica → la query edition torna null → mostra `PaywallPlaceholder`
- [ ] Apertura di `/edizione/testata-inesistente/2026-04-28` → 404
- [ ] Apertura di `/edizione/giornale-di-monza/INVALIDDATE` → 404
- [ ] Da loggato senza subscription attiva (test secondario, opzionale) → paywall placeholder
- [ ] Commit pushato su main → auto-deploy Netlify completato → tutto sopra funziona anche su staging

---

## 9. NOTE PER L'AGENTE

- **Server component fetch**: usare SEMPRE il client da `lib/supabase/server.ts`, mai il browser client per le query DB. Le RLS si basano su `auth.uid()` che funziona solo col server client autenticato.
- **NO middleware update**: la pagina `/edizione/...` è pubblica (la vediamo anche da non loggato per mostrare il paywall). Non aggiungere `/edizione` al matcher di protezione.
- **NO service role**: niente uso di `SUPABASE_SERVICE_ROLE_KEY` in questo sprint. Tutte le query passano per il server client utente, RLS fa il filtro.
- **Date validation**: regex `^\d{4}-\d{2}-\d{2}$` è sufficiente, non serve di più.
- **Heyzine flipbook_id placeholder**: lo sai, l'iframe darà errore Heyzine fino all'aggiornamento da parte dell'orchestratore. La spec è considerata superata se la struttura del DOM è corretta e il branching entitled/paywall funziona.
- **Lingua**: italiano per tutti i testi user-facing.
- **NO Stripe, NO checkout**: il bottone del paywall è disabled. L'integrazione vera arriva con Sprint 04 dopo la decisione canalizzazione.
- **In caso di errore bloccante** (build failure, tipi rotti, query che esplode): FERMARE e segnalare a Paolo. Niente pivot non concordati.

---

## 10. OUTPUT ATTESO PER PAOLO

```
[CLAUDE CODE FRONTEND SPRINT 05 — DONE]

Repo: https://github.com/Paolocesareo/editorkit (commit <sha>)
Auto-deploy Netlify: ✅ build state ready
URL staging: https://editorkit-staging.netlify.app

File creati:
- app/edizione/[publication-slug]/[date]/page.tsx (nuovo)
- components/edition/heyzine-embed.tsx (nuovo)
- components/edition/paywall-placeholder.tsx (nuovo)
- components/dashboard/publication-card.tsx (nuovo)
- app/dashboard/page.tsx (rifatto)

Test eseguiti in locale:
- /dashboard mostra Giornale di Monza con card: ✅
- Click "Sfoglia" → /edizione/giornale-di-monza/2026-04-28: ✅
- Pagina edizione da entitled: iframe presente (con placeholder flipbook_id, errore Heyzine atteso): ✅
- Pagina edizione da non loggato: paywall placeholder visibile: ✅
- /edizione/testata-fake/2026-04-28: 404: ✅
- /edizione/giornale-di-monza/data-invalida: 404: ✅

Pronto per ritest end-to-end su staging dopo aggiornamento flipbook_id reale da parte dell'orchestratore.
```

---

*Sprint 05 — Versione 1 — 30 aprile 2026*
