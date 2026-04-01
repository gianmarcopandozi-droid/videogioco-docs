# Changelog — Videogioco della Vita

## 2026-03-31 — Sessione 3 (maratona completa)

### Infrastruttura
- Deploy n8n su Railway (n8n-production-cfc6.up.railway.app) — bot sempre online
- Deploy webapp su Vercel (webapp-nine-hazel.vercel.app) — fix 404 con vercel.json
- GitHub repo privato (gianmarcopandozi-droid/videogioco-vita)
- GENERIC_TIMEZONE=Europe/Rome su Railway — cron in ora italiana

### Bot Telegram
- Fix classificatore: parseDate() con date assolute ("15 aprile"), vizi regex, focus senza minuti, timezone CET
- Edge Function whisper-proxy per vocali — bypassa bug gzip multipart n8n
- Focus reminder: workflow separato con timer Wait + messaggio Telegram
- chatId dinamico in 14 nodi Telegram (Report Serale resta hardcoded: cron personale)
- Fix Reset Giornaliero e Morning Briefing — riscritti per Supabase (erano Notion)

### Webapp
- Pagina Obiettivi CRUD — gerarchia Vision → Milestone → Action, 9 vision seed
- Achievement display su Dashboard — badge con glow/dimmed
- Priority badges su Missioni — URGENTE/ALTA/NORMALE/BASSA
- Task raggruppate per data — Scadute/Oggi/Domani/Settimana/Più avanti
- Task ordinate per due_date (era created_at)
- Flash mock data rimosso da tutte le pagine — loading skeleton

### Architettura
- Analisi multi-livello: personale → collaboratori → SaaS
- user_id aggiunto a 16 tabelle — pronto per multi-utente
- chatId dinamico — pronto per collaboratori
- Script setup-collaborator.sh
- Schema shared_tasks per collaborazione Karta/Libro
- Migration add-user-id.sql eseguita su Supabase

## 2026-03-30 — Sessione 2
- Migrazione completa da Notion a Supabase (zero nodi Notion)
- Webapp React con 7 pagine cyberpunk neon
- 21 comandi bot Telegram funzionanti
- CRUD abitudini nel SettingsDrawer
- Pagina Progetti con collaboratori
- Shop con deduction monete

## 2026-03-29 — Sessione 1
- Piano iniziale del sistema
- Schema Supabase v1 + v2
- Webapp React scaffold
- 5 workflow n8n creati

## 2026-03-31 (sera) — Sessione 3b
- Link Analyzer v2: trascrizione audio video via whisper-proxy Edge Function
- ScrapeCreators integrato per Instagram/TikTok/YouTube (Jina bloccato su YT)
- Routing per piattaforma: IG/TT/YT → ScrapeCreators, web → Jina
- Fix URL Instagram /reels/ → /reel/
- Post-processing nomi tool (Epify→Apify, Cloud→Claude)
- Groq modello 70B per riassunti dettagliati (era 8B)
- Prompt ristrutturato: estrae step concreti, file, configurazioni, tool specifici
- Fix titolo "Instagram" → summary/caption nel PATCH Supabase
- Webapp Ispirazione: filtri, ricerca, edit inline, tool badges
- Piano per prossima sessione: Groq Vision per screenshot video + lista spesa

## 2026-04-01 (pomeriggio) — Sessione 4b
- Kanban board: 3 colonne Da fare/In corso/Fatto con drag & drop
- Campo assigned_to: assegnazione task a collaboratori
- Filtro persona per progetto (basato su tabella collaborators)
- 16 task inserite (7 Karta + 9 VisitLenola) con assegnazioni
- Progetti aggiornati con description e keywords completi
- Edit/elimina collaboratori dalla pagina Progetti
- Fix classificatore: link early return (evita match "status" nell'HTML)
- Fix Link Analyzer: continueOnError su Whisper per video >25MB
- Fix duplicati: UPSERT su inspiration con UNIQUE url, messaggio "già salvato"
- Groq 70B per riassunti link (era 8B)
- Post-processing nomi tool (Epify→Apify, Cloud→Claude)
- parent_project colonna: CGA→Karta, Lenola Card→VisitLenola
- Fix Yoga: emoji 🧘 + area Salute Fisica
- Security audit completo pre-setup Francesco
- Seed Francesco + script setup aggiornato con sed token
- Docs setup Francesco + Claude.ai
- Pulizia: 20 duplicati inspiration + 9 task vecchie

## 2026-04-01 (sera) — Sessione 4c
- Fix achievements 400 (order by name, not created_at)
- Fix LIVELLO 0 flash (loading skeleton completo su Dashboard)
- Antigravity integrato: audit UX + Batch 1 (shimmer, hover, color coding)
- Antigravity: lint fixes (hoisting, eslint-disable)
- Color coding aree vita sulle HabitCard
- Regola: npm run lint obbligatorio dopo modifiche webapp
- Dedup link con UNIQUE url + UPSERT + messaggio "già salvato"
- Filtro persona basato su collaborators (non su task)
- Edit/elimina collaboratori
- Security audit completo
