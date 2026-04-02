# Videogioco della Vita — Riepilogo v5

**Data:** 2 Aprile 2026
**Stato:** Sistema completo, operativo su Railway + Vercel + Supabase — security hardening in corso

---

## Infrastruttura

| Servizio | URL | Stato |
|----------|-----|-------|
| n8n | n8n-production-cfc6.up.railway.app | ✅ Railway |
| Webapp | webapp-nine-hazel.vercel.app | ✅ Vercel |
| DB | hluyhqyawmvooarusclt.supabase.co | ✅ Supabase |
| Docs | github.com/gianmarcopandozi-droid/videogioco-docs | ✅ Pubblico |
| Repo | github.com/gianmarcopandozi-droid/videogioco-vita | ✅ Privato |

---

## Workflows n8n (7 attivi su Railway)

| Workflow | ID | Trigger | Stato |
|----------|-----|---------|-------|
| Telegram v5 | ldlVc7iNsXO5Pabt | Webhook | ✅ 103 nodi |
| Reset Abitudini | SEJOMbdFCOoI8h5Y | Cron 22:01 UTC | ✅ |
| Morning Briefing | lVdcxKRsF9bkSkWj | Cron 06:00 UTC | ✅ |
| Penalita Mezzanotte | rY1lHlS8SS9GgI6A | Cron 21:45 UTC | ✅ |
| Check Achievement | ycDgcIRVy4f6bsIJ | Ogni ora | ✅ |
| Focus Reminder | GGZjBosqjQ7q3AFy | Webhook | ✅ |
| Link Analyzer | DxurpUVCZQwzXIxQ | Webhook | ✅ 16 nodi |
| Knowledge Base | iLCLYxgIVqFt2UbX | Cron 21:00 UTC | ✅ |

Tutti i cron in UTC (CEST = UTC+2).

---

## Comandi Bot (21 + vocali + link + foto)

Tutti funzionanti. Link analizzati con ScrapeCreators + Whisper + Groq 70B.
Foto classificate: lista_spesa → shopping_list, screenshot/foto → inspiration.

---

## Database Supabase (26 tabelle + Edge Functions)

### Tabelle
characters, level_config, life_areas, habits, projects, activities, xp_log, habit_tracker, inspiration, shop_items, shop_purchases, achievements, weekly_challenges, focus_sessions, mood_log, journal, network_contacts, revenue_log, books, vices_log, streak_penalties, cga_log, habit_reminders, collaborators, goals, shopping_list

### Edge Functions
- whisper-proxy — Proxy Groq Whisper (vocali + video transcription)

### RLS ⚠️ (hardening in corso)
- Script `supabase/fix-rls-select-only.sql` pronto — SELECT-only per anon su 26 tabelle
- Bloccato: n8n deve passare a service_role key prima di applicare RLS
- whisper-proxy: auth con header x-proxy-token pronta, da deployare
- Webhook Telegram: secret_token da configurare

---

## Webapp React (10 pagine)

| Pagina | Descrizione |
|--------|-------------|
| Dashboard | Personaggio, XP/HP bar, abitudini, achievement, best streak |
| Missioni | Task per data (Scadute/Oggi/Domani/Settimana), priority badges |
| Spesa | Lista spesa con checkbox, dedup, pulisci completati |
| Obiettivi | Vision → Milestone → Action, CRUD, progresso |
| Progetti | CRUD, collaboratori, task, focus |
| Aree Vita | Radar chart + barre progresso |
| Ispirazione | Filtri, ricerca, edit, tool badges, contatore processati |
| Statistiche | Grafici XP, streak, top aree |
| Negozio | Items, saldo, acquisti con deduction |
| Impostazioni | CRUD abitudini (drawer) |

---

## Webapp — Fix sessione 2 Aprile (mattina)

| Fix | File | Impatto |
|-----|------|---------|
| Mock data solo DEV | supabase.ts | In PROD se DB down → errore, non dati finti |
| Error handling | 9 pagine | Try/catch + UI errore + bottone "Riprova" |
| Code splitting | App.tsx | React.lazy su 8 pagine, Statistics+Recharts (375KB) lazy |
| useMemo filtri | Inspiration.tsx | Ricerca non ricalcola a ogni keystroke |
| useMemo progress | Objectives.tsx | Da O(N²) a O(N) nel calcolo progresso |
| Loading skeleton | Shop, Projects, Objectives | 3 pagine non avevano loading state |

## Cose rimaste

### Security (bloccate — serve intervento manuale)
1. **MCP n8n su Railway** — serve API key da n8n Settings
2. **Nodi n8n da anon a service_role key** — serve MCP o manuale su Railway
3. **Eseguire fix-rls-select-only.sql** — dopo punto 2
4. **Deploy whisper-proxy con auth token** — serve supabase CLI
5. **Secret_token webhook Telegram** — eseguire manualmente con curl

### Feature
- **Setup Francesco** — script pronto, ~90 min
- **Groq Vision screenshot video** — estrarre testo da slide/UI nei reel
- **Board view Ispirazione** — colonne per progetto, drag & drop
- **Onboarding flow** — per nuovi utenti

### Monitoraggio
- Verificare cron stanotte (reset 00:01, penalità 23:45)
- Knowledge Base push alle 23:00 su GitHub
- Morning Briefing alle 08:00
