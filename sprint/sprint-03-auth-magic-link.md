# Sprint 03 — Auth Flow Magic Link End-to-End

> Spec per Claude Code. Implementazione completa del flusso di autenticazione magic link: `/login`, callback handler, middleware di protezione, hook `useUser()`, logout.

---

## 1. OBIETTIVO

Mettere in piedi l'autenticazione magic link end-to-end sul frontend EditorKit. Al termine dello sprint:

- L'utente può richiedere un magic link da `/login`
- Cliccando il link nell'email atterra su `/dashboard` autenticato
- `/dashboard` è una rotta protetta: chi non è loggato viene rimandato a `/login`
- L'utente può fare logout dall'header
- La sessione persiste tra refresh

Lo Stripe e il paywall sono **fuori scope** di questo sprint (parcheggiati su decisione canalizzazione). L'entitlement (chi ha pagato cosa) sarà mockato in Sprint successivo. Qui ci interessa solo il "sei chi dici di essere".

---

## 2. CONTESTO

Workspace progetto: https://github.com/Paolocesareo/Paolo/blob/master/progetti/editorkit.md

Sprint precedenti chiusi:
- Sprint 01 (backend Supabase): https://github.com/Paolocesareo/editorkit-specs/blob/main/sprint/sprint-01-supabase-setup.md
- Sprint 02 (frontend scaffolding): https://github.com/Paolocesareo/editorkit-specs/blob/main/sprint/sprint-02-frontend-scaffolding.md

Stack consolidato:

- Next.js 15 App Router + TypeScript
- Tailwind CSS v4 + shadcn/ui
- `@supabase/ssr` (già installato in Sprint 02)
- Auth Supabase: email + magic link attivi, **signup pubblico DISABILITATO**

Implicazione del signup disabilitato: il magic link funziona solo per utenti già presenti in `auth.users`. Per testare, Paolo crea l'utente di test manualmente da Supabase Studio prima di lanciare il flow.

---

## 3. OWNER

**Claude Code**

---

## 4. PRE-REQUISITI

- Sprint 02 chiuso: scaffolding Next.js attivo in `C:\001-Sviluppo\editorkit\`
- `.env.local` popolato con credenziali Supabase reali (`editorkit-prod`, già in essere)
- `npm run dev` parte senza errori sulla baseline corrente
- Utente di test creato manualmente in `auth.users` di Supabase (Paolo lo fa prima di testare)
- SMTP SiteGround configurato (sender `noreply@callagent.it`, già attivo)

---

## 5. CARTELLA DI LAVORO

Path assoluto: `C:\001-Sviluppo\editorkit\`

Usare path assoluti per ogni operazione, NON cambiare working directory.

NON toccare `supabase/` (cartella di Kimi).

I file `lib/supabase/client.ts` e `lib/supabase/server.ts` esistono già da Sprint 02 e devono restare invariati.

---

## 6. DELIVERABLE

Al termine Claude Code consegna a Paolo:

1. Pagina `/login` funzionante (form magic link)
2. Route handler `app/auth/callback/route.ts` per scambio code → session
3. File `middleware.ts` alla root del progetto che protegge `/dashboard`
4. Hook `hooks/use-user.ts` per accesso utente client-side
5. Componente `components/auth/logout-button.tsx`
6. Header aggiornato con Accedi/Logout in base allo stato auth
7. `/dashboard` aggiornata con email utente visibile e logout button
8. Pagina `/login/check-email` (conferma post-invio link)
9. Gestione errori callback (link scaduto/invalido) con messaggio chiaro
10. Commit pushato su `Paolocesareo/editorkit` con messaggio convenzionale

---

## 7. SPECIFICHE DETTAGLIATE

### 7.1 Pagina `/login`

**File**: `app/login/page.tsx`

Sostituire integralmente il placeholder esistente. Deve essere un **client component** (`"use client"` in cima) perché gestisce form state.

Comportamento:

- Form con campo `email` (required, type="email")
- Pulsante submit "Invia magic link"
- On submit: chiama `supabase.auth.signInWithOtp({ email, options: { emailRedirectTo: ... } })`
- `emailRedirectTo` deve puntare a `${NEXT_PUBLIC_SITE_URL}/auth/callback`
- Stato di caricamento durante la chiamata (button disabled, label "Invio in corso...")
- Su successo: redirect a `/login/check-email?email=<email>` (con email come query param per mostrarla nella conferma)
- Su errore: mostra messaggio sotto al form (es. "Errore: <message>"). Se l'errore Supabase è del tipo "signups not allowed" → testo italiano: "Questa email non è registrata. Contatta l'editore."

Layout: container centrato, max-width contenuto. Usare `Card`, `Input`, `Label`, `Button` di shadcn/ui (già installati).

Esempio scheletro UI:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { createClient } from "@/lib/supabase/client";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";

export default function LoginPage() {
  const [email, setEmail] = useState("");
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const router = useRouter();
  const supabase = createClient();

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError(null);

    const { error } = await supabase.auth.signInWithOtp({
      email,
      options: {
        emailRedirectTo: `${window.location.origin}/auth/callback`,
      },
    });

    if (error) {
      setError(
        error.message.includes("not allowed")
          ? "Questa email non è registrata. Contatta l'editore."
          : `Errore: ${error.message}`
      );
      setLoading(false);
      return;
    }

    router.push(`/login/check-email?email=${encodeURIComponent(email)}`);
  }

  return (
    <div className="container mx-auto flex min-h-[calc(100vh-3.5rem)] items-center justify-center px-4 py-16">
      <Card className="w-full max-w-md">
        <CardHeader>
          <CardTitle>Accedi a EditorKit</CardTitle>
          <CardDescription>Ti inviamo un link sicuro via email per accedere.</CardDescription>
        </CardHeader>
        <CardContent>
          <form onSubmit={handleSubmit} className="space-y-4">
            <div className="space-y-2">
              <Label htmlFor="email">Email</Label>
              <Input
                id="email"
                type="email"
                value={email}
                onChange={(e) => setEmail(e.target.value)}
                required
                placeholder="tuo@email.it"
                disabled={loading}
              />
            </div>
            {error && <p className="text-sm text-destructive">{error}</p>}
            <Button type="submit" className="w-full" disabled={loading}>
              {loading ? "Invio in corso..." : "Invia magic link"}
            </Button>
          </form>
        </CardContent>
      </Card>
    </div>
  );
}
```

### 7.2 Pagina `/login/check-email`

**File**: `app/login/check-email/page.tsx`

Server component. Legge `email` dalla query string e mostra messaggio di conferma:

```tsx
type Props = {
  searchParams: Promise<{ email?: string }>;
};

export default async function CheckEmailPage({ searchParams }: Props) {
  const { email } = await searchParams;
  return (
    <div className="container mx-auto flex min-h-[calc(100vh-3.5rem)] items-center justify-center px-4 py-16">
      <div className="max-w-md text-center">
        <h1 className="text-2xl font-bold">Controlla la tua email</h1>
        <p className="mt-4 text-muted-foreground">
          Abbiamo inviato un link di accesso a <strong>{email ?? "te"}</strong>.
          Clicca il link per completare il login. Il link scade in 1 ora.
        </p>
        <p className="mt-4 text-sm text-muted-foreground">
          Non vedi nulla? Controlla la cartella spam o riprova tra qualche minuto.
        </p>
      </div>
    </div>
  );
}
```

### 7.3 Callback handler

**File**: `app/auth/callback/route.ts` (Route Handler, no page)

Gestisce il code emesso da Supabase quando l'utente clicca il magic link. Lo scambia per la sessione e redirecta a `/dashboard` (o `next` se presente).

```ts
import { createClient } from "@/lib/supabase/server";
import { NextResponse } from "next/server";

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get("code");
  const next = searchParams.get("next") ?? "/dashboard";

  if (code) {
    const supabase = await createClient();
    const { error } = await supabase.auth.exchangeCodeForSession(code);
    if (!error) {
      return NextResponse.redirect(`${origin}${next}`);
    }
  }

  return NextResponse.redirect(`${origin}/auth/error`);
}
```

### 7.4 Pagina errore auth

**File**: `app/auth/error/page.tsx`

Server component, mostra messaggio chiaro che il link è scaduto/invalido + CTA per tornare al login:

```tsx
import Link from "next/link";
import { Button } from "@/components/ui/button";

export default function AuthErrorPage() {
  return (
    <div className="container mx-auto flex min-h-[calc(100vh-3.5rem)] items-center justify-center px-4 py-16">
      <div className="max-w-md text-center">
        <h1 className="text-2xl font-bold">Link non valido</h1>
        <p className="mt-4 text-muted-foreground">
          Il link di accesso è scaduto o non è valido. I link magic link durano 1 ora
          e funzionano una sola volta.
        </p>
        <Button asChild className="mt-6">
          <Link href="/login">Richiedi un nuovo link</Link>
        </Button>
      </div>
    </div>
  );
}
```

### 7.5 Middleware di protezione

**File**: `middleware.ts` (alla root del progetto, NON dentro `app/`)

Il middleware fa due cose:

1. Refresh della sessione Supabase (necessario con `@supabase/ssr` per mantenere i cookie validi)
2. Protezione delle rotte: se l'utente non è loggato e prova ad accedere a `/dashboard*`, redirect a `/login?next=<original-path>`

```ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  const {
    data: { user },
  } = await supabase.auth.getUser();

  const isProtected = request.nextUrl.pathname.startsWith("/dashboard");

  if (isProtected && !user) {
    const url = request.nextUrl.clone();
    url.pathname = "/login";
    url.searchParams.set("next", request.nextUrl.pathname);
    return NextResponse.redirect(url);
  }

  return supabaseResponse;
}

export const config = {
  matcher: [
    /*
     * Match all request paths except for:
     * - _next/static (static files)
     * - _next/image (image optimization)
     * - favicon.ico
     * - public files (.svg, .png, ...)
     */
    "/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

### 7.6 Hook `useUser()`

**File**: `hooks/use-user.ts`

Espone `{ user, loading }` per componenti client. Si sottoscrive ai cambi di stato auth.

```ts
"use client";

import { useEffect, useState } from "react";
import type { User } from "@supabase/supabase-js";
import { createClient } from "@/lib/supabase/client";

export function useUser() {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const supabase = createClient();

    supabase.auth.getUser().then(({ data }) => {
      setUser(data.user);
      setLoading(false);
    });

    const { data: subscription } = supabase.auth.onAuthStateChange((_event, session) => {
      setUser(session?.user ?? null);
    });

    return () => subscription.subscription.unsubscribe();
  }, []);

  return { user, loading };
}
```

### 7.7 Logout button

**File**: `components/auth/logout-button.tsx`

Client component. Su click chiama `signOut()` e redirecta a `/`.

```tsx
"use client";

import { useRouter } from "next/navigation";
import { createClient } from "@/lib/supabase/client";
import { Button } from "@/components/ui/button";

export function LogoutButton() {
  const router = useRouter();
  const supabase = createClient();

  async function handleLogout() {
    await supabase.auth.signOut();
    router.push("/");
    router.refresh();
  }

  return (
    <Button variant="ghost" size="sm" onClick={handleLogout}>
      Esci
    </Button>
  );
}
```

### 7.8 Header aggiornato

**File**: `components/layout/header.tsx`

Trasformarlo in client component (`"use client"`). Usare `useUser()` per decidere se mostrare "Accedi" o "Esci" + email troncata.

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
          {loading ? null : user ? (
            <>
              <Link href="/dashboard" className="text-muted-foreground hover:text-foreground">
                {user.email}
              </Link>
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

### 7.9 Dashboard aggiornata

**File**: `app/dashboard/page.tsx`

Server component. Usa il server client per verificare l'utente. Anche se il middleware già protegge, fare il check qui è doppia sicurezza e ci dà l'oggetto user senza chiamate client-side ulteriori.

```tsx
import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";

export default async function DashboardPage() {
  const supabase = await createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) redirect("/login");

  return (
    <div className="container mx-auto px-4 py-16">
      <h1 className="text-2xl font-bold">Le tue testate</h1>
      <p className="mt-2 text-muted-foreground">
        Sei loggato come <strong>{user.email}</strong>.
      </p>
      <p className="mt-6 text-sm text-muted-foreground">
        L'elenco delle testate apparirà qui non appena Sprint 05 sarà completato.
      </p>
    </div>
  );
}
```

### 7.10 Verifica `.env.local`

Controllare che `NEXT_PUBLIC_SITE_URL` sia valorizzato a `http://localhost:3000`. Se manca, aggiungerlo. È usato implicitamente da `window.location.origin` in `/login`, ma teniamo l'env coerente.

### 7.11 Commit

Path assoluto, no `cd`:

```bash
git -C C:\001-Sviluppo\editorkit add .
git -C C:\001-Sviluppo\editorkit commit -m "feat: auth flow magic link end-to-end (sprint 03)"
git -C C:\001-Sviluppo\editorkit push origin main
```

---

## 8. CRITERI DI ACCETTAZIONE

- [ ] `npm run dev` parte senza errori
- [ ] `/login` mostra form email funzionante
- [ ] Submit con email valida → redirect a `/login/check-email?email=...`
- [ ] Email magic link arriva alla casella di test (verificare cartella spam)
- [ ] Click sul link → atterro su `/dashboard` autenticato, l'header mostra l'email
- [ ] Apertura diretta `/dashboard` da non loggato → redirect a `/login?next=/dashboard`
- [ ] Click "Esci" dall'header → torna su `/`, header torna a "Accedi"
- [ ] Riapertura `/dashboard` post-logout → di nuovo redirect a `/login`
- [ ] Link magic link scaduto/già usato → atterro su `/auth/error` con messaggio chiaro
- [ ] Email NON presente in `auth.users` → messaggio "Questa email non è registrata. Contatta l'editore."
- [ ] Refresh pagina su `/dashboard` → resto loggato (sessione persiste)
- [ ] Commit pushato su `Paolocesareo/editorkit` branch `main`

---

## 9. NOTE PER L'AGENTE

- **Non installare** librerie nuove. Tutto quello che serve è già nel package.json da Sprint 02 (`@supabase/ssr`, `@supabase/supabase-js`, shadcn components base).
- **Non toccare** `lib/supabase/client.ts` e `lib/supabase/server.ts`. Sono già configurati correttamente.
- **`exchangeCodeForSession`** funziona perché abbiamo `@supabase/ssr` v0.5+. Se Supabase restituisce errore "code verifier missing", è un problema di cookie/PKCE: in tal caso verificare che il middleware sia attivo (refresh sessione) e che `cookies` sia gestito correttamente dal server client.
- **Signup disabilitato**: Paolo deve creare l'utente di test manualmente in Supabase Studio → Authentication → Add user → "Create new user" prima di testare. Il magic link a un'email non registrata ritorna errore.
- **In caso di errore bloccante** (es. callback handler che 500-a, middleware che redirect-loop), FERMARE e segnalare a Paolo prima di sostituire approcci. Niente pivot non concordati.
- **Lingua UI**: italiano. Tutti i testi user-facing in italiano.

---

## 10. OUTPUT ATTESO PER PAOLO

Al termine, Claude Code consegna:

```
[CLAUDE CODE FRONTEND SPRINT 03 — DONE]

Repo: https://github.com/Paolocesareo/editorkit (commit <sha>)
Cartella locale: C:\001-Sviluppo\editorkit\
Dev server: http://localhost:3000

File creati:
- app/login/page.tsx (rifatto)
- app/login/check-email/page.tsx (nuovo)
- app/auth/callback/route.ts (nuovo)
- app/auth/error/page.tsx (nuovo)
- middleware.ts (nuovo, alla root)
- hooks/use-user.ts (nuovo)
- components/auth/logout-button.tsx (nuovo)
- components/layout/header.tsx (aggiornato)
- app/dashboard/page.tsx (aggiornato)

Test eseguiti in locale:
- /login + submit + magic link + click + atterraggio dashboard: ✅
- protezione /dashboard da non loggato: ✅
- logout: ✅
- link scaduto → /auth/error: ✅
- email non registrata → messaggio chiaro: ✅

Pronto per Sprint 04 (Stripe sandbox) o Sprint 05 (pagina edizione + Heyzine).
```

---

*Sprint 03 — Versione 1 — 30 aprile 2026*
