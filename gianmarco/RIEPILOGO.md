# Videogioco della Vita — Riepilogo v5

**Data:** 3 Aprile 2026
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

## Workflows n8n (8 attivi su Railway)

| Workflow | ID | Trigger | Stato |
|----------|-----|---------|-------|
| Telegram v5 | ldlVc7iNsXO5Pabt | Webhook | ✅ 106 nodi |
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

## Fix sessione 3 Aprile — Classificatore Bot

| Fix | Nodo | Impatto |
|-----|------|---------|
| Rimosso 'bevuto' da aliasMap | Classifica | "Ho bevuto prosecco" non matcha più "Bere acqua" |
| Aggiunto pattern vizio alcol | Classifica | Prosecco/vino/birra/spritz → vizio |
| Aggiunto nodo SB Carica Task Aperti | Nuovo (106° nodo) | Task aperti disponibili nel classificatore |
| Check _tasks per completamento | Classifica | "Ho fatto X" → completamento solo se X è task reale |
| Prompt Groq con lista task | Groq Fallback | LLM ha contesto task + abitudini per classificare |
| azione_completata → nota ✅ | Parse Groq | Azioni completate non in lista → salvate come nota |
| Prompt Vision lista spesa | Groq Vision | Interpreta grafia (CAbote→carote) |
| Alias camminare/passeggiata | Classifica | "Ho fatto una passeggiata" → abitudine se esiste |

---

## Cose rimaste

### Security
1. ✅ **MCP n8n su Railway** — API key configurata, deploy via REST API funzionante
2. **Nodi n8n da anon a service_role key** — BLOCCATO: service_key in ~/.env.videogioco è duplicato anon key. Serve vera service_role dal Dashboard.
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
