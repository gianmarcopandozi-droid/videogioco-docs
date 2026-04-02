# REGOLE SISTEMA PERSONALE GIANMARCO

## PROCEDURA OBBLIGATORIA PRIMA DI OGNI MODIFICA

### Step 0: Leggi TUTTO
1. Leggi questo file CLAUDE.md interamente
2. Leggi RIEPILOGO.md per lo stato attuale
3. Leggi TUTTE le skill disponibili pertinenti

### Step 1: Consulta le skill pertinenti

**QUANDO LEGGERE LE SKILL (obbligatorio):**
- Prima di creare/modificare componenti UI → `frontend-design`
- Prima di modificare workflow n8n → `n8n-workflow-patterns` + `n8n-code-javascript`
- Prima di debug complesso → `systematic-debugging`
- Per fix semplici (config, bug singolo) → non necessario

**Per workflow n8n** — prima di toccare QUALSIASI workflow:
- n8n-workflow-patterns, n8n-node-configuration, n8n-code-javascript
- n8n-validation-expert, n8n-expression-syntax, n8n-mcp-tools-expert

**Per frontend webapp** — prima di creare/modificare pagine o componenti:
- Leggere e seguire `~/.claude/plugins/marketplaces/claude-plugins-official/plugins/frontend-design/skills/frontend-design/SKILL.md` per design production-grade
- Usare `systematic-debugging` per bug UI
- Usare `verification-before-completion` prima di dichiarare "fatto"

**Per qualsiasi implementazione** — seguire le skill superpowers:
- `brainstorming` → `writing-plans` → `subagent-driven-development` o `executing-plans`
- `test-driven-development` per ogni feature/bugfix
- `requesting-code-review` prima di merge

### Step 2: Diagnostica PRIMA di fixare
- Leggi l'esecuzione n8n per vedere l'errore ESATTO
- Identifica il nodo che fallisce e il messaggio di errore
- Traccia il flusso dati nodo per nodo
- NON proporre fix senza aver letto l'errore reale

### Step 3: Verifica conflitti PRIMA di implementare
- Controlla che la modifica non rompa nodi esistenti
- Verifica che i nodi disconnessi vengano rimossi o riconnessi
- Usa `validateOnly: true` quando possibile
- Controlla che TUTTI i dati seed necessari esistano nel DB

### Step 4: Implementa con checklist
Per OGNI nodo n8n aggiunto o modificato, verificare:
- [ ] Nodo Telegram ha `chatId: "488886504"` hardcoded
- [ ] Nodo HTTP GET ha `alwaysOutputData: true`
- [ ] Code node NON usa `$helpers`, `fetch()`, o `require()`
- [ ] Code node usa solo `$input`, `$json`, `$('Nodo')`, `$getWorkflowStaticData`
- [ ] Per HTTP calls nei Code node: usa nodi HTTP Request SEPARATI
- [ ] Switch/If node: connections verificate per ogni branch
- [ ] Nodi Notion: usano format proprietà semplificato (property_xxx)
- [ ] Nodi HTTP Supabase: hanno apikey + Authorization + Content-Type headers

### Step 5: Verifica post-deploy
- Controlla che il workflow sia attivo
- Chiedi all'utente di testare il comando specifico
- Leggi l'esecuzione per confermare successo

## ERRORI NOTI DA NON RIPETERE

### Code nodes n8n
- `$helpers.httpRequest()` → NON ESISTE. Usa nodi HTTP Request separati
- `fetch()` → NON DISPONIBILE nel runtime n8n. Usa nodi HTTP Request
- `require()` → BLOCCATO da n8n. Mai usare
- `$json` in Code node "Run Once for All Items" → NON funziona per item singolo. Usa `$input.first().json`

### HTTP Request nodes
- Se il GET restituisce 0 risultati → il nodo successivo NON viene eseguito
  FIX: aggiungere `alwaysOutputData: true` al nodo GET
- Array vuoto PostgREST: `"All object keys must match"` → tutti gli oggetti nel batch devono avere le stesse chiavi
- HTTP GET Supabase ritorna N items separati in n8n → Code node DEVE usare `$input.all().map(i => i.json)` MAI `$input.first().json`
- Per referenziare output di nodo precedente non diretto: `$('NomeNodo').all().map(i => i.json)` MAI `.first()`

### Telegram nodes
- chatId DINAMICO: usare `$('Classifica').first().json.chat_id` in tutti i nodi Telegram
- ECCEZIONE: Report Serale (cron, nessun utente in input) → hardcoded 488886504
- I nodi Telegram creati da zero: SEMPRE impostare chatId dinamico

### Notion nodes
- L'auto-sanitization di n8n corrompe i propertiesUi del nodo Notion (specialmente URL)
  FIX: usare HTTP Request diretto all'API Notion invece del nodo nativo per proprietà URL
- I nomi proprietà in formato semplificato sono `property_nome_campo` (tutto lowercase con underscore)
- Le formule Notion sono READ-ONLY — non scrivibili via API

### Switch nodes
- Il Switch v3.4 crasha con operatore `object exists` su messaggi con array
  FIX: usare `{{ !!$json.field }}` con `boolean equals true`
- IF v2.2 `string/isNotEmpty` valuta false anche quando il campo ha valore (bug con UUID da PostgREST)
  FIX: usare `{{ !!$json.field }}` con `boolean equals true` — stesso pattern del Switch
- `looseTypeValidation: true` necessario per gestire campi undefined

### Supabase REST API
- DDL (CREATE TABLE, ALTER TABLE) NON possibile via REST API anon key — serve SQL Editor
- Le viste (views) richiedono DDL per essere create
- `check constraint` è case-sensitive (es. `tipo` deve essere minuscolo: 'reel' non 'Reel')
- GET che ritorna array vuoto `[]` → il nodo successivo NON viene eseguito anche con `alwaysOutputData: true`. FIX: usare UPSERT con `Prefer: resolution=ignore-duplicates` invece di GET+IF per dedup
- Per dedup link: UNIQUE INDEX su url + INSERT con `return=representation` + `onError: continueRegularOutput`. Il 409 viene catturato, IF controlla `$json.id isNotEmpty`
- `Prefer: resolution=ignore-duplicates` NON funziona se PostgREST non trova il UNIQUE CONSTRAINT (restituisce 409 invece di ignorare). FIX: usare solo `return=representation` e gestire il 409 con onError
- URL Instagram: SEMPRE normalizzare PRIMA dell'insert (strip query params `?igsh=...`, `/reels/` → `/reel/`, trailing slash) per evitare duplicati dello stesso contenuto

### Classificatore
- Il check `text.includes('status')` matcha anche il testo HTML delle pagine web. FIX: mettere il check link (`source === 'link'`) PRIMA di tutti gli altri check nel classificatore
- Quando si inserisce codice JS in n8n via Python, le `$` vengono strippate. FIX: usare `\x24` per rappresentare `$` in Python (`\x24json.source` → `$json.source`)

### Cron/Scheduling
- `GENERIC_TIMEZONE=Europe/Rome` NON funziona su Railway. Convertire TUTTI i cron manualmente in UTC (CEST=UTC+2, CET=UTC+1)
- Dopo OGNI import workflow su Railway: verificare che i cron workflow funzionino leggendo le esecuzioni entro 24h
- Workflow importati da locale possono avere credenziali Notion/vecchie → falliscono su Railway. SEMPRE verificare dopo import

### Video/Audio
- Groq Whisper ha limite 25MB. Video Instagram possono superarlo. FIX: `onError: continueRegularOutput` su Download Video e Whisper Transcript — il riassunto arriva con solo caption
- Whisper trascrive nomi tool in modo fonetico (Epify→Apify, Cloud→Claude). FIX: post-processing con dizionario di replace nel nodo Parse Risultato, NON nel prompt Groq (il modello piccolo lo ignora)
- Groq LLaMA 8B fa riassunti generici. Usare 70B (`llama-3.3-70b-versatile`) per link analysis

### Edge Functions
- whisper-proxy richiede header `x-proxy-token` — senza token ritorna 401
- Dopo OGNI deploy Edge Function: verificare che il secret sia configurato con `supabase secrets list`
- Le Edge Functions per scritture webapp usano service_role key internamente, MAI anon key

### Webapp
- Inizializzare state con `[]` o `null`, MAI con mock data — causa flash di contenuto finto
- Filtri persona: basarli sulla tabella `collaborators` per progetto, NON sulle task assegnate — altrimenti non appaiono collaboratori senza task

## SICUREZZA (aggiornato 2 Aprile 2026)

### RLS Supabase
- RLS e' SELECT-ONLY per anon key su tutte le 26 tabelle (script: `supabase/fix-rls-select-only.sql`)
- La webapp puo solo LEGGERE con anon key — le scritture passano per Edge Functions
- n8n DEVE usare SERVICE_ROLE KEY (non anon key) per le scritture al DB
- Service_role key bypassa RLS completamente

### whisper-proxy Edge Function
- Protetta con header `x-proxy-token` (secret in Supabase secrets: WHISPER_PROXY_SECRET)
- I nodi n8n che chiamano whisper-proxy DEVONO includere header `x-proxy-token`
- Senza token valido la Edge Function ritorna 401

### Webhook Telegram
- Configurato con `secret_token` nel setWebhook
- Il workflow n8n verifica `x-telegram-bot-api-secret-token` header
- Richieste senza secret vengono scartate (previene command injection)

### Script di fix
- `scripts/fix-security.sh` — genera token e mostra comandi per tutti e 3 i fix
- `supabase/fix-rls-select-only.sql` — script SQL per RLS (eseguire in SQL Editor)

## CONDIVISIONE CHIAVI SEGRETE
Quando l'utente deve condividere una chiave segreta (API key, token, password):
- NON chiedere MAI di incollarla direttamente in chat
- Proporre SEMPRE un metodo sicuro:
  1. **File env per progetto:** `~/.env.videogioco` (caricato da .zshrc, in .gitignore globale)
  2. **Variabile d'ambiente:** `export VAR_NAME=xxx` nel terminale, poi riferirsi a `$VAR_NAME`
  3. **Comando con sostituzione:** fornire il comando con placeholder `<IL_TUO_TOKEN>` da eseguire manualmente
- Se l'utente incolla comunque una chiave in chat: avvisare che resta nella cronologia e suggerire di ruotarla dopo
- MAI salvare chiavi in file tracciati da git, CLAUDE.md, memory, o log
- Per leggere le chiavi in script: `export $(grep -v '^#' ~/.env.videogioco | xargs)`
- Per nuovi progetti: creare `~/.env.nomeprogetto` e aggiungere `source` in `~/.zshrc`

## REGOLE ASSOLUTE
- "Segui procedura" / "segui le regole nostre" = esegui Step 0-5 della PROCEDURA OBBLIGATORIA sopra
- Usa subagent (Agent tool) dove possibile per parallelizzare ricerca, diagnostica, e task indipendenti
- Per bug webapp: diagnostica con curl/log PRIMA di toccare codice
- NON usare require() nei Code node
- NON usare OpenAI o Anthropic API — solo Groq
- Spiega la soluzione PRIMA di implementare, aspetta conferma
- Usa nodi HTTP Request nativi per TUTTE le chiamate HTTP
- $vars NON funziona (licenza a pagamento) — chiavi hardcoded nei nodi
- File audio Telegram sono .oga → rinominare a .ogg PRIMA di Whisper
- Vocali: Supabase Edge Function "whisper-proxy" fa da proxy → Groq Whisper (bypassa bug gzip multipart n8n)
- Catena voce: Rinomina Audio → Whisper Proxy (HTTP Request binary a Edge Function) → Prepara Voice
- NON usare nodo HTTP Request multipart per upload audio — bug ECONNRESET hardcoded in n8n
- Conferma Telegram: usare $('Nodo').first().json, MAI $json direttamente
- Classificatore: parseDate() supporta oggi/domani/dopodomani/giorni settimana/date assolute italiane con timezone CET
- Note e Task finiscono entrambi in tabella `activities` — colonna `activity_type` ('task'/'nota') da aggiungere via DDL
- Railway usa UTC — convertire TUTTI i cron manualmente (GENERIC_TIMEZONE non funziona). CEST=UTC+2, CET=UTC+1
- Dopo OGNI import/deploy su Railway: testare TUTTI i workflow cron leggendo le esecuzioni entro 24h
- Dopo OGNI modifica a Code node: verificare che NON usi $helpers, fetch(), require()

## WORKFLOW N8N ATTIVI (ID = Railway)

| Workflow | ID Railway | Trigger |
|----------|-----------|---------|
| Telegram -> Videogioco v5 | ldlVc7iNsXO5Pabt | Webhook Telegram |
| Link Analyzer | DxurpUVCZQwzXIxQ | Webhook interno (da main dopo insert ispirazione) |
| Knowledge Base Notturno | iLCLYxgIVqFt2UbX | Cron notturno |
| Reset Abitudini | SEJOMbdFCOoI8h5Y | Cron 00:01 |
| Morning Briefing | lVdcxKRsF9bkSkWj | Cron 08:00 |
| Penalita Mezzanotte | rY1lHlS8SS9GgI6A | Cron 23:45 |
| Check Achievement | ycDgcIRVy4f6bsIJ | Ogni ora |
| Focus Reminder Timer | GGZjBosqjQ7q3AFy | Timer focus |

## COMANDI BOT TELEGRAM (21 comandi, tutti funzionanti)

| Comando | Tipo | Esempio |
|---------|------|---------|
| status | status | "come sto?" |
| oggi | oggi | "oggi", "cosa devo fare" |
| streak | streak | "streak" |
| monete | monete | "monete", "saldo" |
| abitudine | abitudine | "ho fatto meditazione" |
| completamento | completamento_task | "ho finito il report" |
| task | task | "ricordami di chiamare" |
| nota | nota | "nota: idea per il libro" |
| link | ispirazione | qualsiasi URL |
| foto | foto | foto con caption |
| vizio | vizio | "ho fumato", "ho sprecato" |
| focus | focus | "focus 25", "pomodoro karta 45 min" |
| shop | shop | "negozio", "cosa posso comprare" |
| acquisto | acquisto | "compra serata netflix" |
| mood | mood | "mi sento 4", "umore 3" |
| gratitudine | gratitudine | "sono grato per...", "grato" |
| network | network | "ho parlato con Mario" |
| cga | cga | "cga: aggiornamento cantiere" |
| revenue | revenue | "nuovo cliente karta: Bar Roma, 50 euro mese" |
| nuova abitudine | nuova_abitudine | "nuova abitudine: Yoga, 30 XP" |
| unknown | AI fallback | qualsiasi testo non riconosciuto → prompt 1/2/3 |

## SCHEMA SUPABASE

**Progetto:** hluyhqyawmvooarusclt
**URL:** https://hluyhqyawmvooarusclt.supabase.co

### Tabelle v1 (operative, con dati)
characters (1), level_config (50), life_areas (9), habits (13),
projects (6), activities (15+), xp_log (34+), habit_tracker (13),
inspiration (9+)

### Tabelle v2 (operative, con dati)
shop_items (5), shop_purchases, achievements (8), weekly_challenges,
focus_sessions, mood_log, journal, network_contacts,
revenue_log, books, vices_log, streak_penalties,
cga_log, habit_reminders, collaborators

### Viste
v_character_stats, v_area_progress, v_xp_daily

## DATABASE NOTION (backup/consultazione — Supabase è il DB primario)
- Buone Abitudini: 29b8782b-2fe0-818f-ab73-df3af7debf45
- Registro de Actividad: 29b8782b-2fe0-81c2-a7b0-cf73f0130b82
- personaje new: 29b8782b-2fe0-813a-996c-ed1820699f49
- Attivita: 29b8782b-2fe0-81ba-8adc-f89d69650cee
- Ispirazione: 8b9c4746-c9a7-4b6e-a4a9-9d4d262c33d8
- Personaje page ID: 29b8782b-2fe0-81a2-b2ed-d19f6800fe17

## COSTANTI
- Telegram chat_id: 488886504
- Supabase character_id: a1b2c3d4-0000-0000-0000-000000000001
- Telegram credential ID: X2hYduq0bgSvYQvK
- Notion credential ID: TFJcbIMzvi6tlyOB

## WEBAPP

### Setup
- Directory: ~/Desktop/SISTEMA/videogioco/webapp/
- Tech: React + Vite + TypeScript + Tailwind CSS 4 + Framer Motion + Recharts
- Dev: `npm run dev`
- Build: `npm run build` (tsc + vite build)
- Estetica: Cyberpunk Neon Dark (#0a0a0f bg, #00f0ff cyan, #ff00aa magenta, #ffb800 amber)

### Struttura file
```
src/
├── main.tsx, App.tsx (router)
├── lib/supabase.ts (client + tutti i fetcher/action)
├── lib/mock-data.ts (fallback se no .env)
├── components/ (Layout, Sidebar, SettingsDrawer, XPBar, HabitCard, StatCard, Character, ecc.)
├── pages/ (Dashboard, Quests, LifeAreas, Inspiration, Statistics, Shop, Projects)
└── hooks/ (useAnimatedValue)
```

### 8 pagine
Dashboard, Missioni, Aree Vita, Ispirazione, Statistiche, Negozio, Progetti, Obiettivi

### Regole
- TUTTE le pagine leggono da Supabase (zero mock nel flusso attivo)
- Ogni nuova pagina DEVE usare fetchXxx() da supabase.ts, MAI dati hardcoded
- Acquisti webapp: POST shop_purchases + POST xp_log (coins_earned negativo) per deduction
- SettingsDrawer: solo CRUD abitudini (progetti hanno pagina dedicata)
- Per nuove pagine/componenti UI: invocare skill `frontend-design` per design production-grade

## VERIFICA WEBAPP — OBBLIGATORIO
Dopo modifiche a webapp, eseguire ENTRAMBI:
1. `npm run build` — verifica TypeScript
2. `npm run lint` — verifica regole React/ESLint
Solo `build` NON basta — lint cattura anti-pattern che build ignora.

## DOPO OGNI ERRORE — OBBLIGATORIO
Quando un bug viene trovato e fixato, aggiungere SUBITO nella sezione
"ERRORI NOTI DA NON RIPETERE" sopra: descrizione errore + FIX.
Non aspettare il commit di fine sessione.

## FINE SESSIONE — OBBLIGATORIO
```bash
# 1. Push repo privato
git -C ~/Desktop/SISTEMA/videogioco add docs/ .claude/ webapp/src/ supabase/ scripts/ n8n-export/
git -C ~/Desktop/SISTEMA/videogioco commit -m "sessione $(date +%Y-%m-%d): [descrizione breve]"
git -C ~/Desktop/SISTEMA/videogioco push

# 2. Sync repo pubblico docs (per Claude.ai)
cd /tmp/videogioco-docs && git pull
cp ~/Desktop/SISTEMA/videogioco/docs/RIEPILOGO.md gianmarco/
cp ~/Desktop/SISTEMA/videogioco/docs/BLOCCO.md gianmarco/
cp ~/Desktop/SISTEMA/videogioco/docs/CHANGELOG.md gianmarco/
cp ~/Desktop/SISTEMA/videogioco/.claude/CLAUDE.md gianmarco/
git add . && git commit -m "sync: $(date +%Y-%m-%d)" && git push
```
MOTIVO: Claude.ai legge i file raw da GitHub. ENTRAMBI i push sono obbligatori.
Fare ANCHE dopo ogni commit importante durante la sessione, non solo alla fine.
