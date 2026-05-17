# EditorKit Specs

Sprint specs e documentazione tecnica del progetto EditorKit (brand pubblico: **FlipKit**).

Questo repository è **pubblico** per consentire agli agenti di sviluppo (Kimi Code, Claude Code) di leggere gli spec via URL raw GitHub senza autenticazione.

**Nessun segreto è committato qui** — credenziali, API key, password vivono solo in `.env.local` o nei pannelli dei servizi.

## Indice sprint

- [Sprint 01 — Setup Supabase backend](./sprint/sprint-01-supabase-setup.md) — Owner: Kimi Code — chiuso
- [Sprint 02 — Frontend Next.js scaffolding](./sprint/sprint-02-frontend-scaffolding.md) — Owner: Claude Code — chiuso
- [Sprint 03 — Auth flow magic link](./sprint/sprint-03-auth-magic-link.md) — Owner: Claude Code — chiuso
- [Sprint 03b — Deploy Netlify staging](./sprint/sprint-03b-netlify-staging.md) — Owner: Claude Code — chiuso
- [Sprint 05 — Pagina edizione + embed Heyzine](./sprint/sprint-05-pagina-edizione.md) — Owner: Claude Code — chiuso
- [Sprint 05b — Edicola pubblica con copertine](./sprint/sprint-05b-edicola-pubblica.md) — Owner: Claude Code — chiuso
- [Sprint 05c — Polish edicola 2 livelli + brand FlipKit](./sprint/sprint-05c-polish-edicola.md) — Owner: Claude Code — chiuso
- [Sprint 05d — Pagina /account minimal + toolbar sfogliatore](./sprint/sprint-05d-account-toolbar.md) — Owner: Claude Code — chiuso
- [Sprint 06 — Onboarding testate (upload manuale admin, pipeline source-agnostic)](./sprint/sprint-06-onboarding-testate.md) — Owner: Kimi Code — chiuso
- [Sprint 06b — Onboarding fix: upload diretto single PUT + conversione async Edge Function](./sprint/sprint-06b-onboarding-fix.md) — Owner: Kimi Code — chiuso (verificato live, PDF reale 110MB)
- [Sprint 07 — Admin entitlement: login staff a password + assegnazione manuale abbonamenti](./sprint/sprint-07-admin-entitlement.md) — Owner: Claude Code — chiuso
- [Sprint P1 — Ricerca full-text archivio edizioni](./sprint/sprint-P1-ricerca-fulltext.md) — Owner: Kimi Code (backend) + Claude Code (UI) — spec v2, architettura decisa (Opzione B), pronta da assegnare

## Workflow

Spec scritta dall'orchestratore → push qui (pubblico) → l'agente owner la legge via raw URL → output `[NOME DONE]` → orchestratore verifica + aggiorna indice + workspace.

---

*Repo creato: 29 aprile 2026 — indice aggiornato: 17 maggio 2026 — Sprint 07 chiuso, Sprint P1 (ricerca full-text) spec scritta con nodo aperto*
