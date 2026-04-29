# Sprint 01 — Setup Supabase Backend

> Spec per Kimi Code. Backend setup completo del progetto Supabase per EditorKit.

---

## 1. OBIETTIVO

Creare il progetto Supabase cloud per EditorKit, configurare auth, applicare schema database e RLS policies. Output: credenziali pronte da inserire nel frontend + file SQL versionati nel repo.

---

## 2. CONTESTO

Workspace progetto: https://github.com/Paolocesareo/Paolo/blob/master/progetti/editorkit.md

Stack confermato:

- Supabase (Auth + Postgres + RLS)
- Auth: email + magic link, **no signup pubblico** (utenti creati solo via Stripe webhook)
- SMTP custom (SiteGround) per invio email auth
- Multi-tenant: gestione di più editori sotto lo stesso DB tramite `publisher_id`

---

## 3. OWNER

**Kimi Code**

---

## 4. PRE-REQUISITI

- Account Supabase Paolo: attivo (Paolo conferma il piano e fornisce credenziali login se necessario)
- Cartella locale `C:\001-Sviluppo\editorkit\` può non esistere ancora (la creerà Claude Code in Sprint 02)
- Per i file SQL Kimi può scrivere temporaneamente in `C:\001-Sviluppo\editorkit-temp-backend\` se la cartella principale non esiste ancora, poi spostare quando il repo è pronto

---

## 5. CARTELLA DI LAVORO

Tutti i file SQL prodotti vanno salvati con path assoluto:

`C:\001-Sviluppo\editorkit\supabase\migrations\`

Se la cartella non esiste, crearla. NON cambiare la working directory: usare path assoluti.

Convenzione naming file: `001_initial_schema.sql`, `002_rls_policies.sql`, ecc.

---

## 6. DELIVERABLE

Al termine del task Kimi consegna a Paolo:

1. **Project URL Supabase** (formato: `https://xxx.supabase.co`)
2. **Anon Key** (chiave pubblica, finirà in `NEXT_PUBLIC_SUPABASE_ANON_KEY`)
3. **Service Role Key** (chiave server-side, finirà in `SUPABASE_SERVICE_ROLE_KEY`)
4. **File SQL versionati** in `C:\001-Sviluppo\editorkit\supabase\migrations\`:
   - `001_initial_schema.sql`
   - `002_rls_policies.sql`
   - `003_indexes.sql`
5. **Conferma testuale** che SMTP è configurato e funzionante (test email mandata a `paolo@csflab.it`)

---

## 7. SPECIFICHE DETTAGLIATE

### 7.1 Creazione progetto Supabase cloud

- **Nome progetto:** `editorkit-prod`
- **Organization:** quella di Paolo (chiedergli quale se ne ha più di una)
- **Region:** `eu-central-1` (Frankfurt)
- **Database password:** generare password forte (32 char misti), consegnarla a Paolo separatamente
- **Pricing tier:** Pro (Paolo conferma)

### 7.2 Configurazione Auth

Pannello Supabase → Authentication → Providers:

- **Email auth:** ABILITATO
- **Magic link:** ABILITATO
- **Email confirmations:** ABILITATO (richiede verifica email al primo login)
- **Secure email change:** ABILITATO
- **Enable Phone:** DISABILITATO
- **Enable Anonymous Sign-Ins:** DISABILITATO

Pannello Supabase → Authentication → Settings:

- **Allow new users to sign up:** **DISABILITATO** (importante: utenti creati solo da backend via Stripe webhook)
- **Site URL:** `https://editorkit.it` (placeholder, Paolo aggiornerà a deploy reale)
- **Redirect URLs:** aggiungere `http://localhost:3000/**` e `https://editorkit.it/**`

### 7.3 Configurazione SMTP custom

Pannello Supabase → Authentication → Email Templates → Custom SMTP:

| Campo | Valore |
|---|---|
| Sender email | `noreply@callagent.it` (temporaneo, finché non c'è mailbox `editorkit.it`) |
| Sender name | `EditorKit` |
| Host | `c1117347.sgvps.net` |
| Port | `587` |
| Username | `noreply@callagent.it` |
| Password | (Paolo la fornisce separatamente per sicurezza) |
| Minimum interval between emails | `60` secondi (default) |

Test obbligatorio: dal pannello, mandare email di test a `paolo@csflab.it`. Se non arriva, NON procedere e segnalare a Paolo.

### 7.4 Schema database — file `001_initial_schema.sql`

Schema completo da applicare:

```sql
-- ========================================
-- EDITORKIT — INITIAL SCHEMA
-- ========================================

-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ========================================
-- PUBLISHERS — editori sulla piattaforma
-- ========================================
CREATE TABLE publishers (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  vat_number TEXT,
  iban TEXT,
  contact_email TEXT NOT NULL,
  contact_phone TEXT,
  commission_rate NUMERIC(5,4) NOT NULL CHECK (commission_rate >= 0 AND commission_rate <= 1),
  num_publications INTEGER NOT NULL DEFAULT 0,
  contract_start DATE,
  contract_end DATE,
  is_active BOOLEAN NOT NULL DEFAULT true,
  custom_domain TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_publishers_active ON publishers(is_active) WHERE is_active = true;
CREATE INDEX idx_publishers_custom_domain ON publishers(custom_domain) WHERE custom_domain IS NOT NULL;

-- ========================================
-- PUBLICATIONS — testate
-- ========================================
CREATE TABLE publications (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  publisher_id UUID NOT NULL REFERENCES publishers(id) ON DELETE RESTRICT,
  name TEXT NOT NULL,
  slug TEXT NOT NULL,
  description TEXT,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(publisher_id, slug)
);

CREATE INDEX idx_publications_publisher ON publications(publisher_id);
CREATE INDEX idx_publications_active ON publications(is_active) WHERE is_active = true;

-- ========================================
-- EDITIONS — singole edizioni pubblicate
-- ========================================
CREATE TABLE editions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  publication_id UUID NOT NULL REFERENCES publications(id) ON DELETE RESTRICT,
  edition_date DATE NOT NULL,
  edition_number TEXT,
  title TEXT,
  pdf_url TEXT,
  heyzine_flipbook_id TEXT,
  heyzine_url TEXT,
  thumbnail_url TEXT,
  num_pages INTEGER,
  aspect_ratio NUMERIC(5,4),
  is_published BOOLEAN NOT NULL DEFAULT false,
  published_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(publication_id, edition_date)
);

CREATE INDEX idx_editions_publication ON editions(publication_id);
CREATE INDEX idx_editions_date ON editions(edition_date DESC);
CREATE INDEX idx_editions_published ON editions(is_published, published_at DESC) WHERE is_published = true;

-- ========================================
-- USER PROFILES — estende auth.users
-- ========================================
CREATE TABLE user_profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT NOT NULL,
  full_name TEXT,
  stripe_customer_id TEXT UNIQUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_profiles_email ON user_profiles(email);
CREATE INDEX idx_user_profiles_stripe ON user_profiles(stripe_customer_id) WHERE stripe_customer_id IS NOT NULL;

-- Trigger: crea automaticamente user_profiles quando viene creato un auth.users
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.user_profiles (id, email, full_name)
  VALUES (
    NEW.id,
    NEW.email,
    COALESCE(NEW.raw_user_meta_data->>'full_name', NULL)
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();

-- ========================================
-- SUBSCRIPTIONS — abbonamenti attivi
-- ========================================
CREATE TYPE subscription_tier AS ENUM ('tier_1', 'tier_2', 'tier_3');
CREATE TYPE subscription_status AS ENUM ('active', 'past_due', 'canceled', 'paused', 'incomplete');

CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  tier subscription_tier NOT NULL,
  stripe_subscription_id TEXT UNIQUE,
  stripe_price_id TEXT,
  publications_included UUID[] NOT NULL DEFAULT '{}',
  publisher_id UUID REFERENCES publishers(id) ON DELETE RESTRICT,
  status subscription_status NOT NULL DEFAULT 'incomplete',
  current_period_start TIMESTAMPTZ,
  current_period_end TIMESTAMPTZ,
  cancel_at_period_end BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscriptions_user ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_status ON subscriptions(status, current_period_end);
CREATE INDEX idx_subscriptions_publisher ON subscriptions(publisher_id);
CREATE INDEX idx_subscriptions_stripe ON subscriptions(stripe_subscription_id);

-- ========================================
-- TRANSACTIONS — ledger di tutte le transazioni
-- ========================================
CREATE TABLE transactions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE RESTRICT,
  publisher_id UUID NOT NULL REFERENCES publishers(id) ON DELETE RESTRICT,
  subscription_id UUID REFERENCES subscriptions(id) ON DELETE SET NULL,
  stripe_charge_id TEXT UNIQUE,
  stripe_invoice_id TEXT,
  transaction_date TIMESTAMPTZ NOT NULL DEFAULT now(),
  amount_gross NUMERIC(10,2) NOT NULL,
  stripe_fee NUMERIC(10,2) NOT NULL,
  amount_net_of_stripe NUMERIC(10,2) NOT NULL,
  csf_commission NUMERIC(10,2) NOT NULL,
  publisher_net NUMERIC(10,2) NOT NULL,
  currency TEXT NOT NULL DEFAULT 'EUR',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_transactions_user ON transactions(user_id);
CREATE INDEX idx_transactions_publisher ON transactions(publisher_id, transaction_date);
CREATE INDEX idx_transactions_date ON transactions(transaction_date DESC);

-- ========================================
-- SETTLEMENTS — rendicontazione mensile editori
-- ========================================
CREATE TYPE settlement_status AS ENUM ('pending', 'invoiced', 'paid', 'disputed');

CREATE TABLE settlements (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  publisher_id UUID NOT NULL REFERENCES publishers(id) ON DELETE RESTRICT,
  period_year INTEGER NOT NULL,
  period_month INTEGER NOT NULL CHECK (period_month BETWEEN 1 AND 12),
  total_gross NUMERIC(10,2) NOT NULL DEFAULT 0,
  total_stripe_fee NUMERIC(10,2) NOT NULL DEFAULT 0,
  total_csf_commission NUMERIC(10,2) NOT NULL DEFAULT 0,
  total_net_to_pay NUMERIC(10,2) NOT NULL DEFAULT 0,
  num_transactions INTEGER NOT NULL DEFAULT 0,
  status settlement_status NOT NULL DEFAULT 'pending',
  invoice_received_at TIMESTAMPTZ,
  invoice_number TEXT,
  paid_at TIMESTAMPTZ,
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(publisher_id, period_year, period_month)
);

CREATE INDEX idx_settlements_publisher ON settlements(publisher_id, period_year DESC, period_month DESC);
CREATE INDEX idx_settlements_status ON settlements(status);

-- ========================================
-- TRIGGERS — updated_at automatico
-- ========================================
CREATE OR REPLACE FUNCTION public.update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_publishers_updated_at BEFORE UPDATE ON publishers FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_publications_updated_at BEFORE UPDATE ON publications FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_editions_updated_at BEFORE UPDATE ON editions FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_user_profiles_updated_at BEFORE UPDATE ON user_profiles FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_subscriptions_updated_at BEFORE UPDATE ON subscriptions FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER set_settlements_updated_at BEFORE UPDATE ON settlements FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

### 7.5 RLS Policies — file `002_rls_policies.sql`

```sql
-- ========================================
-- ROW LEVEL SECURITY — abilitazione
-- ========================================
ALTER TABLE publishers ENABLE ROW LEVEL SECURITY;
ALTER TABLE publications ENABLE ROW LEVEL SECURITY;
ALTER TABLE editions ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;
ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE settlements ENABLE ROW LEVEL SECURITY;

-- ========================================
-- USER PROFILES — utente vede solo se stesso
-- ========================================
CREATE POLICY "users_read_own_profile"
  ON user_profiles FOR SELECT
  USING (auth.uid() = id);

CREATE POLICY "users_update_own_profile"
  ON user_profiles FOR UPDATE
  USING (auth.uid() = id);

-- ========================================
-- SUBSCRIPTIONS — utente vede solo le sue
-- ========================================
CREATE POLICY "users_read_own_subscriptions"
  ON subscriptions FOR SELECT
  USING (auth.uid() = user_id);

-- ========================================
-- PUBLICATIONS — pubblicamente leggibili (sono il "catalogo")
-- ========================================
CREATE POLICY "publications_public_read_active"
  ON publications FOR SELECT
  USING (is_active = true);

-- ========================================
-- PUBLISHERS — pubblicamente leggibili dati base (info pubbliche)
-- ========================================
CREATE POLICY "publishers_public_read_active"
  ON publishers FOR SELECT
  USING (is_active = true);

-- ========================================
-- EDITIONS — il cuore dell'entitlement
-- ========================================
-- Un utente vede un'edizione SE:
-- 1. L'edizione è pubblicata
-- 2. Ha una subscription attiva che include la testata di quella edizione
CREATE POLICY "editions_paid_users_see_published"
  ON editions FOR SELECT
  USING (
    is_published = true
    AND publication_id IN (
      SELECT unnest(publications_included)
      FROM subscriptions
      WHERE user_id = auth.uid()
        AND status = 'active'
        AND current_period_end > now()
    )
  );

-- ========================================
-- TRANSACTIONS — utente vede solo le proprie
-- ========================================
CREATE POLICY "users_read_own_transactions"
  ON transactions FOR SELECT
  USING (auth.uid() = user_id);

-- ========================================
-- SETTLEMENTS — solo service_role (no accesso utenti finali)
-- Default: nessuna policy = nessun accesso. Service role bypassa RLS.
-- ========================================
-- Nessuna policy: solo backend service_role può leggere/scrivere
```

### 7.6 File `003_indexes.sql`

Vuoto per ora. Eventuali indici aggiuntivi specifici per query lente verranno aggiunti dopo profiling.

```sql
-- Indices aggiuntivi (placeholder per future ottimizzazioni)
```

---

## 8. CRITERI DI ACCETTAZIONE

Spunta tutti per considerare lo sprint chiuso:

- [ ] Progetto Supabase `editorkit-prod` esiste in regione Frankfurt
- [ ] Auth email + magic link configurato
- [ ] Signup pubblico DISABILITATO
- [ ] SMTP SiteGround configurato e test email a `paolo@csflab.it` ricevuta
- [ ] Tutte le 7 tabelle create senza errori
- [ ] Tutti gli indici creati
- [ ] Trigger `handle_new_user` attivo (test: creare utente da SQL editor, verificare che `user_profiles` venga popolato)
- [ ] Trigger `updated_at` attivi su tutte le tabelle che li hanno
- [ ] RLS abilitato su tutte le 7 tabelle
- [ ] Policy `editions_paid_users_see_published` testata con dati di esempio:
  - Creare 1 publisher, 1 publication, 1 edition pubblicata, 1 utente con subscription attiva → l'utente vede l'edition ✅
  - Stesso utente senza subscription → NON vede l'edition ✅
- [ ] File SQL salvati in `C:\001-Sviluppo\editorkit\supabase\migrations\` con nomi esatti specificati
- [ ] Project URL, anon key, service role key consegnati a Paolo

---

## 9. NOTE PER L'AGENTE

- **Lingua codice:** SQL standard Postgres + estensioni Supabase
- **Stile SQL:** snake_case, lowercase per keywords (preferenza Paolo)
- **Commenti:** in italiano sopra ogni sezione, brevi
- **Errori:** se uno step fallisce, FERMARE e segnalare a Paolo. NON proseguire con workaround creativi
- **Sicurezza:** la service_role key NON va in nessun file committato, va solo consegnata a Paolo separatamente
- **Test obbligatori:**
  1. Email SMTP test (dopo punto 7.3)
  2. Trigger user creation (dopo applicazione 001)
  3. RLS entitlement (dopo applicazione 002)
- **In caso di dubbio:** chiedere a Paolo. NON inventare default. Esempio: se Paolo non specifica IBAN per il publisher di test, lasciare NULL, non inventare un IBAN finto.

---

## 10. OUTPUT ATTESO PER PAOLO

Al termine, Kimi consegna a Paolo un messaggio nel formato:

```
[KIMI BACKEND SPRINT 01 — DONE]

Project URL: https://xxxxxxx.supabase.co
Anon Key: eyJhbGc...
Service Role Key: eyJhbGc... (riservata, NON committata)
Database password: (consegnata a parte)

File creati:
- C:\001-Sviluppo\editorkit\supabase\migrations\001_initial_schema.sql
- C:\001-Sviluppo\editorkit\supabase\migrations\002_rls_policies.sql
- C:\001-Sviluppo\editorkit\supabase\migrations\003_indexes.sql

Test eseguiti:
- SMTP test email: ✅ ricevuta
- Trigger handle_new_user: ✅ funzionante
- RLS entitlement editions: ✅ verificato

Pronto per Sprint 02 (Claude Code frontend).
```

---

*Sprint 01 — Versione 1 — 29 aprile 2026*
