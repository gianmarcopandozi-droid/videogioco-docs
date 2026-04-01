# Videogioco della Vita — Riepilogo v3

**Data:** 30 Marzo 2026
**Stato:** Sistema operativo, vocali funzionanti, Railway-ready

---

## Workflows n8n (5 attivi)

| Workflow | ID | Nodi | Trigger |
|----------|-----|------|---------|
| Telegram -> Videogioco v5 | mJ3FwZFyNXToRnS0 | 97 | Webhook Telegram |
| Reset Abitudini | 8CufFeuKa2hR1UqD | 5 | Cron 00:01 |
| Morning Briefing | YxHPnGM0B57TMkST | 6 | Cron 08:00 |
| Penalita Mezzanotte | GSg0ZoZ3rOtIAvkC | 4 | Cron 23:45 |
| Check Achievement | VoldFmQJ8gpfIZg8 | 4 | Ogni ora |

---

## Comandi Bot Telegram (21 comandi + vocali)

| Comando | Tipo | Handler | Stato |
|---------|------|---------|-------|
| status | status | Supabase GET → format → Telegram | ✅ |
| oggi | oggi | Supabase GET habits+tasks → format | ✅ |
| streak | streak | Supabase GET habit_tracker → format | ✅ |
| monete | monete | Supabase GET stats+purchases → format | ✅ |
| abitudine | abitudine | Supabase mark + xp_log | ✅ |
| completamento | completamento_task | Supabase search+complete | ✅ |
| task | task | Supabase POST activities | ✅ |
| nota | nota | Supabase POST activities | ✅ |
| link | ispirazione | Supabase POST inspiration | ✅ |
| foto | foto | Groq Vision → Supabase | ✅ |
| vizio | vizio | Supabase POST vices_log | ✅ |
| focus | focus | Supabase POST focus_sessions | ✅ |
| shop | shop | Supabase GET shop_items → lista | ✅ |
| acquisto | acquisto | Supabase check coins+POST purchase | ✅ |
| mood | mood | Supabase POST mood_log | ✅ |
| gratitudine | gratitudine | Supabase POST journal | ✅ |
| network | network | Supabase POST network_contacts | ✅ |
| cga | cga | Supabase POST cga_log | ✅ |
| revenue | revenue | Supabase POST revenue_log | ✅ |
| nuova abitudine | nuova_abitudine | Supabase POST habits + tracker | ✅ |
| unknown | unknown | AI Interpreter + prompt 1/2/3 | ✅ |
| **vocale** | voice | Edge Function proxy → Groq Whisper → Classifica | ✅ |

### Architettura vocali (Railway-ready)
```
Telegram voice .oga → Download → Rinomina .ogg →
HTTP Request (binary) → Supabase Edge Function "whisper-proxy" →
Groq Whisper API (multipart via Deno fetch) → testo → Classifica
```
Bypassa bug gzip multipart di n8n. Funziona ovunque (locale, Railway, Cloud).

---

## Database Supabase (24 tabelle + 1 Edge Function)

### Tabelle v1 (operative, con dati)
characters (1), level_config (50), life_areas (9), habits (14), projects (6), activities (15+), xp_log (34+), habit_tracker (14), inspiration (9+)

### Tabelle v2 (operative, con dati)
shop_items (5), shop_purchases, achievements (8), weekly_challenges, focus_sessions, mood_log, journal, network_contacts, revenue_log, books, vices_log, streak_penalties, cga_log, habit_reminders, collaborators

### Viste
v_character_stats, v_area_progress, v_xp_daily

### Edge Functions
- **whisper-proxy** — Proxy per Groq Whisper (riceve binary audio, chiama Groq con multipart, ritorna testo)

---

## Webapp React (8 pagine)

| Pagina | Descrizione | Dati |
|--------|-------------|------|
| Dashboard | Personaggio, XP bar, abitudini checkbox | Supabase live |
| Missioni | Task per progetto con filtri, crea/completa | Supabase live |
| Aree Vita | Radar chart + barre progresso | Supabase live |
| Ispirazione | Grid masonry | Supabase live |
| Statistiche | Grafici XP, streak, top aree | Supabase live |
| Negozio | Items, saldo, compra con deduction monete | Supabase live |
| Progetti | CRUD progetti, collaboratori, task, focus | Supabase live |
| Obiettivi | Vision → Milestone → Action, CRUD, progresso | Supabase live |

---

## Deploy Railway (Step 2 — da fare)

### Prerequisiti
- Account Railway (Hobby plan ~$5/mese)
- Template: `railway.com/deploy/n8n`

### Env vars necessarie
```
N8N_ENCRYPTION_KEY=<openssl rand -hex 32>
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=<password>
WEBHOOK_URL=https://${{RAILWAY_PUBLIC_DOMAIN}}
N8N_DEFAULT_BINARY_DATA_MODE=filesystem
EXECUTIONS_DATA_PRUNE=true
```

### Passi
1. Deploy n8n + PostgreSQL da template Railway
2. Esportare workflow locale: `n8n export:workflow --all`
3. Importare su Railway
4. Aggiornare webhook Telegram con nuovo URL Railway
5. Testare tutti i comandi

### Note
- Docker n8n usa Node.js 24 — stesso bug multipart, ma la Edge Function proxy lo bypassa
- curl NON presente nel container — non serve grazie alla Edge Function
- Le env vars `NODES_EXCLUDE` e `N8N_RESTRICT_FILE_ACCESS_TO` NON servono su Railway

---

## Cose rimaste

### Completati
- ~~Deploy webapp Vercel~~ ✅ https://webapp-nine-hazel.vercel.app
- ~~Deploy n8n Railway~~ ✅ https://n8n-production-cfc6.up.railway.app

### Fix classificatore (applicati su Railway 31/03/2026)
1. ~~Parse Groq perde date~~ ✅ Aggiunto parseDate() in Parse Groq — AI tasks ora hanno scadenza
2. ~~Nota POST senza due_date~~ ✅ Supabase Nota ora include due_date + activity_type
3. ~~Lista vizi restrittiva~~ ✅ Regex flessibili: "ho mangiato X", "ho procrastinato", "ho giocato", ecc.
4. ~~Focus senza minuti~~ ✅ "focus karta" → default 25 min
5. ~~Date assolute~~ ✅ Supportate: "15 aprile", "3 maggio", ecc. (parseDate con mesi italiani)
6. ~~Timezone classificatore~~ ✅ parseDate() usa CET offset
7. **Supabase Task** ora include activity_type='task'

### Azioni manuali necessarie
7. **GENERIC_TIMEZONE su Railway** — Aggiungere variabile `GENERIC_TIMEZONE=Europe/Rome` nelle Variables del servizio n8n su Railway. Questo fix fa funzionare tutti i cron (reset 00:01, morning 08:00, report 21:00, penalità 23:45) in ora italiana.
8. **Colonna activity_type** — Eseguire nel SQL Editor Supabase: `ALTER TABLE activities ADD COLUMN IF NOT EXISTS activity_type TEXT DEFAULT 'task';`

### Fix sicurezza
9. **RLS Supabase** — tutte le 24 tabelle allow all for anon (piano in BLOCCO.md #8)

### Webapp (fix applicati 31/03/2026)
- ~~HP bar su Dashboard~~ ✅ già implementata
- ~~Achievement su Dashboard~~ ✅ badge row con glow per sbloccati, dimmed per locked
- ~~Flash mock data~~ ✅ rimosso da tutte le pagine (loading skeleton)
- ~~Task ordinate per data~~ ✅ raggruppate: Scadute/Oggi/Domani/Settimana/Più avanti
- ~~Priority badge su Missioni~~ ✅ URGENTE/ALTA/NORMALE/BASSA basato su scadenza
- ~~Pagina Obiettivi~~ ✅ CRUD completo con gerarchia Vision → Milestone → Action

### Roadmap
11. **Colonna activity_type** — distinguere task da note nel DB
12. **Onboarding flow** — filosofia di vita, goals, progetti per nuovi utenti
