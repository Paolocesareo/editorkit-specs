# Sprint 02 — Frontend Next.js Scaffolding

> Spec per Claude Code. Setup repo + scaffolding Next.js 15 + integrazione Supabase client.

---

## 1. OBIETTIVO

Creare il repo `Paolocesareo/editorkit` su GitHub e mettere in piedi lo scaffolding Next.js 15 con Tailwind, shadcn/ui e integrazione Supabase client. Il frontend deve essere pronto a connettersi al backend non appena Kimi (Sprint 01) consegna le credenziali.

---

## 2. CONTESTO

Workspace progetto: https://github.com/Paolocesareo/Paolo/blob/master/progetti/editorkit.md

Sprint parallelo backend: https://github.com/Paolocesareo/Paolo/blob/master/progetti/editorkit/sprint/sprint-01-supabase-setup.md

Stack confermato:

- Next.js 15 App Router + TypeScript
- Tailwind CSS v4 + shadcn/ui
- Supabase client per auth e DB
- Hosting target: Netlify (deploy verrà configurato in Sprint successivo)
- Frontale Cloudflare per DNS/CDN/sottodomini editori (configurazione in Sprint successivo)

---

## 3. OWNER

**Claude Code**

---

## 4. PRE-REQUISITI

- Account GitHub Paolo accessibile (PAT già disponibile, Paolo lo fornisce)
- Node.js 20+ installato in locale
- pnpm o npm installato

---

## 5. CARTELLA DI LAVORO

Path assoluto del progetto:

`C:\001-Sviluppo\editorkit\`

Se la cartella non esiste, crearla. NON cambiare la working directory: usare path assoluti per ogni operazione.

Sotto-cartella `C:\001-Sviluppo\editorkit\supabase\` deve essere lasciata intatta — è gestita da Kimi (backend), non sovrascrivere file SQL eventualmente già presenti.

---

## 6. DELIVERABLE

Al termine del task Claude Code consegna a Paolo:

1. **Repo GitHub** `Paolocesareo/editorkit` creato (privato), con primo commit pushato
2. **Cartella locale** `C:\001-Sviluppo\editorkit\` con scaffolding Next.js 15 funzionante
3. **Integrazione Supabase client** configurata con env vars placeholder
4. **Layout base + 3 pagine placeholder** (`/`, `/login`, `/dashboard`)
5. **README.md** del repo con istruzioni come avviare il dev server
6. **Conferma testuale** che `npm run dev` parte senza errori e mostra la home

---

## 7. SPECIFICHE DETTAGLIATE

### 7.1 Inizializzazione progetto Next.js

Comando da eseguire (con path assoluto, senza `cd`):

```bash
npx create-next-app@latest C:\001-Sviluppo\editorkit --typescript --tailwind --app --no-src-dir --import-alias "@/*" --use-npm
```

Risposte alle domande interattive (se chieste):

- ESLint: **Yes**
- Turbopack per `next dev`: **Yes**

### 7.2 Struttura cartelle attesa post-init

```
C:\001-Sviluppo\editorkit\
├── app/
│   ├── layout.tsx          (creato da template, da personalizzare)
│   ├── page.tsx            (home, da rifare)
│   ├── login/
│   │   └── page.tsx        (placeholder)
│   ├── dashboard/
│   │   └── page.tsx        (placeholder)
│   └── globals.css
├── components/
│   ├── ui/                 (shadcn/ui qui)
│   └── layout/
│       └── header.tsx
├── lib/
│   ├── supabase/
│   │   ├── client.ts       (browser client)
│   │   └── server.ts       (server client)
│   └── utils.ts
├── supabase/               (NON TOCCARE — gestita da Kimi)
│   └── migrations/
├── .env.local.example
├── .env.local              (gitignored, con placeholder)
├── .gitignore
├── package.json
├── tsconfig.json
├── tailwind.config.ts
├── next.config.ts
└── README.md
```

### 7.3 Installazione dipendenze aggiuntive

Eseguire (con path assoluto al package.json):

```bash
npm install --prefix C:\001-Sviluppo\editorkit @supabase/supabase-js @supabase/ssr
```

### 7.4 Setup shadcn/ui

```bash
npx shadcn@latest init --cwd C:\001-Sviluppo\editorkit
```

Risposte:

- Style: **Default**
- Base color: **Slate**
- CSS variables: **Yes**

Installare i primi componenti che serviranno:

```bash
npx shadcn@latest add button card input label --cwd C:\001-Sviluppo\editorkit
```

### 7.5 Integrazione Supabase client

**File `lib/supabase/client.ts`** (browser client per componenti client-side):

```typescript
import { createBrowserClient } from '@supabase/ssr'

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

**File `lib/supabase/server.ts`** (server client per RSC e route handlers):

```typescript
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createClient() {
  const cookieStore = await cookies()

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll()
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            )
          } catch {
            // The `setAll` method was called from a Server Component.
            // This can be ignored if you have middleware refreshing user sessions.
          }
        },
      },
    }
  )
}
```

### 7.6 File `.env.local.example`

Da committare nel repo come template (il `.env.local` reale è gitignored):

```
# Supabase (Sprint 01 Kimi consegna questi valori)
NEXT_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGc...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGc...

# Stripe (Sprint successivo)
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# Heyzine (configurato a livello applicativo in Sprint successivo)
HEYZINE_CLIENT_ID=877c907b0a7d03e8
HEYZINE_API_KEY=

# Site
NEXT_PUBLIC_SITE_URL=http://localhost:3000
```

### 7.7 File `.env.local` con placeholder

Stessa struttura, ma con placeholder che permettono al server di partire (anche se non si connetterà davvero a Supabase):

```
NEXT_PUBLIC_SUPABASE_URL=https://placeholder.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=placeholder-anon-key
SUPABASE_SERVICE_ROLE_KEY=placeholder-service-key
NEXT_PUBLIC_SITE_URL=http://localhost:3000
HEYZINE_CLIENT_ID=877c907b0a7d03e8
```

Quando Kimi consegna le credenziali vere, Paolo le sostituirà manualmente in questo file.

### 7.8 Layout base

**File `app/layout.tsx`** — sostituire il template con:

```tsx
import type { Metadata } from "next";
import "./globals.css";
import { Header } from "@/components/layout/header";

export const metadata: Metadata = {
  title: "EditorKit",
  description: "Piattaforma di distribuzione digitale per editori italiani",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="it">
      <body className="min-h-screen bg-background font-sans antialiased">
        <Header />
        <main>{children}</main>
      </body>
    </html>
  );
}
```

**File `components/layout/header.tsx`**:

```tsx
import Link from "next/link";

export function Header() {
  return (
    <header className="border-b">
      <div className="container mx-auto flex h-14 items-center justify-between px-4">
        <Link href="/" className="font-semibold">
          EditorKit
        </Link>
        <nav className="flex items-center gap-4 text-sm">
          <Link href="/login">Accedi</Link>
        </nav>
      </div>
    </header>
  );
}
```

### 7.9 Pagine placeholder

**File `app/page.tsx`** (home):

```tsx
export default function HomePage() {
  return (
    <div className="container mx-auto px-4 py-16">
      <h1 className="text-4xl font-bold tracking-tight">EditorKit</h1>
      <p className="mt-4 text-lg text-muted-foreground">
        Piattaforma di distribuzione digitale per editori italiani.
      </p>
    </div>
  );
}
```

**File `app/login/page.tsx`** (placeholder):

```tsx
export default function LoginPage() {
  return (
    <div className="container mx-auto px-4 py-16">
      <h1 className="text-2xl font-bold">Accedi</h1>
      <p className="mt-2 text-muted-foreground">
        Form di login (placeholder, implementazione in Sprint successivo).
      </p>
    </div>
  );
}
```

**File `app/dashboard/page.tsx`** (placeholder):

```tsx
export default function DashboardPage() {
  return (
    <div className="container mx-auto px-4 py-16">
      <h1 className="text-2xl font-bold">Le tue testate</h1>
      <p className="mt-2 text-muted-foreground">
        Dashboard abbonato (placeholder).
      </p>
    </div>
  );
}
```

### 7.10 README.md del repo

Contenuto:

```markdown
# EditorKit

Piattaforma di distribuzione digitale per editori italiani.

## Stack

- Next.js 15 (App Router) + TypeScript
- Tailwind CSS v4 + shadcn/ui
- Supabase (Auth + Postgres + RLS)
- Stripe Billing
- Heyzine (sfogliatore PDF)

## Setup locale

1. Clone repo
2. `npm install`
3. Copia `.env.local.example` in `.env.local` e compila i valori
4. `npm run dev`
5. Apri http://localhost:3000

## Struttura

- `app/` — pagine Next.js App Router
- `components/` — componenti React (ui shadcn + layout)
- `lib/supabase/` — client Supabase (browser + server)
- `supabase/migrations/` — migrazioni DB (gestite via Supabase CLI)

## Documentazione progetto

Workspace: https://github.com/Paolocesareo/Paolo/blob/master/progetti/editorkit.md
```

### 7.11 Inizializzazione git e push su GitHub

1. Inizializzare git (con path assoluto, no cd):
   ```bash
   git -C C:\001-Sviluppo\editorkit init
   ```

2. Creare repo su GitHub usando API (Paolo fornisce PAT):
   ```bash
   curl -X POST -H "Authorization: token <PAT>" \
     -H "Content-Type: application/json" \
     "https://api.github.com/user/repos" \
     -d '{"name":"editorkit","private":true,"description":"Piattaforma distribuzione digitale per editori italiani"}'
   ```

3. Aggiungere remote:
   ```bash
   git -C C:\001-Sviluppo\editorkit remote add origin https://github.com/Paolocesareo/editorkit.git
   ```

4. Verificare `.gitignore` includa `.env.local`, `node_modules/`, `.next/`

5. Primo commit:
   ```bash
   git -C C:\001-Sviluppo\editorkit add .
   git -C C:\001-Sviluppo\editorkit commit -m "feat: scaffolding Next.js 15 + Tailwind + shadcn/ui + Supabase client"
   git -C C:\001-Sviluppo\editorkit branch -M main
   git -C C:\001-Sviluppo\editorkit push -u origin main
   ```

---

## 8. CRITERI DI ACCETTAZIONE

- [ ] Repo `Paolocesareo/editorkit` creato su GitHub (privato)
- [ ] Cartella `C:\001-Sviluppo\editorkit\` esiste con struttura attesa
- [ ] `npm run dev` parte senza errori
- [ ] http://localhost:3000 mostra la home con testo "EditorKit" e header
- [ ] http://localhost:3000/login mostra placeholder login
- [ ] http://localhost:3000/dashboard mostra placeholder dashboard
- [ ] shadcn/ui inizializzato (esiste `components/ui/button.tsx`)
- [ ] File `lib/supabase/client.ts` e `lib/supabase/server.ts` presenti
- [ ] File `.env.local.example` committato, `.env.local` NON committato (verificare gitignore)
- [ ] README.md committato
- [ ] Push su GitHub riuscito, commit visibile su web

---

## 9. NOTE PER L'AGENTE

- **Stile codice:** TypeScript strict, ESLint default Next.js
- **Convenzione naming:** kebab-case per file, PascalCase per componenti React
- **Lingua UI:** italiano (`<html lang="it">`)
- **NON installare** librerie non strettamente necessarie a questo sprint (no auth provider esterni, no i18n, no testing framework). Mantenere snello.
- **NON toccare** la cartella `supabase/` se esiste già con file dentro — è di Kimi.
- **In caso di errore** (es. shadcn/ui che non si installa, o conflitti dipendenze), FERMARE e segnalare a Paolo. NON cambiare versioni o sostituire librerie senza approvazione.

---

## 10. OUTPUT ATTESO PER PAOLO

Al termine, Claude Code consegna a Paolo un messaggio nel formato:

```
[CLAUDE CODE FRONTEND SPRINT 02 — DONE]

Repo: https://github.com/Paolocesareo/editorkit
Cartella locale: C:\001-Sviluppo\editorkit\
Dev server: http://localhost:3000

Test eseguiti:
- npm run dev: ✅ avvio senza errori
- Home: ✅ visibile
- /login: ✅ placeholder OK
- /dashboard: ✅ placeholder OK

In attesa: credenziali Supabase da Kimi (Sprint 01) per popolare .env.local.

Pronto per Sprint 03 (auth flow integrato).
```

---

*Sprint 02 — Versione 1 — 29 aprile 2026*
