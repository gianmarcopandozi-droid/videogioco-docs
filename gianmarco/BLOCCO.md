# BLOCCO — Azioni manuali e pianificate

## 1. ✅ Schema v2 eseguito su Supabase

## 2. ✅ Deploy Vercel completato
URL: https://webapp-nine-hazel.vercel.app

## 3. ✅ Tutti gli handler n8n collegati (97 nodi)

## 4. ✅ Workflow Penalita Mezzanotte creato e attivo

## 5. ✅ Workflow Check Achievement creato e attivo

## 6. ✅ Colonna description aggiunta a projects

## 7. ✅ Deploy n8n su Railway
URL: https://n8n-production-cfc6.up.railway.app

## 8. FIX RLS Supabase (BLOCCATO — serve service_role key)

### Problema attuale
Tutte le 24 tabelle hanno policy `FOR ALL USING (true)` — chiunque con la anon key può SELECT, INSERT, UPDATE, DELETE su tutto.

### BLOCCO (3 Aprile 2026)
`SUPABASE_SERVICE_KEY` in `~/.env.videogioco` è un duplicato della anon key (role:"anon").
Serve la vera service_role key dal Supabase Dashboard → Settings → API.
Senza questa key, non è possibile:
- Aggiornare i nodi n8n da anon a service_role
- Applicare RLS SELECT-only (i workflow n8n smetterebbero di funzionare)

### Piano
1. **Cambiare tutte le policy a SELECT-only per anon:**
```sql
-- Per ogni tabella:
DROP POLICY IF EXISTS "Allow all for anon" ON <tabella>;
DROP POLICY IF EXISTS "Allow all anon" ON <tabella>;
CREATE POLICY "Anon read only" ON <tabella> FOR SELECT USING (true);
```

2. **Creare Edge Functions per le scritture dalla webapp:**
   - `complete-habit` — segna abitudine fatta + registra XP
   - `complete-task` — completa task + registra XP
   - `purchase-item` — compra item shop + deduci monete
   - `create-task` — crea nuova task
   - `update-project` — aggiorna progetto
   - `update-collaborator` — aggiorna collaboratore

3. **Aggiornare supabase.ts nella webapp:**
   - Le funzioni di scrittura (completeHabit, createTask, ecc.) dovranno chiamare Edge Functions invece di Supabase client diretto

4. **Tabelle che restano read-write per anon (nessuna):**
   - TUTTE le scritture passano per Edge Functions con service_role key

### Tabelle critiche (priorità alta)
- characters, xp_log, habit_tracker — economia del gioco
- shop_items, shop_purchases — negozio
- revenue_log — dati finanziari
- journal — dati privati
- collaborators — contiene telegram_id

### Stima: ~1-2 ore di lavoro

## 9. ✅ Obiettivi a 3 livelli (implementato)

## 10. ✅ Fix Reset Giornaliero per Railway (31/03/2026)

### Problema
Il workflow "Reset Abitudini Giornaliero" usava nodi Notion (credenziale TFJcbIMzvi6tlyOB non presente su Railway). Falliva con errore "Credential does not exist".

### Fix applicato
Riscritto il workflow per usare Supabase direttamente:
- Trigger: 00:01 (con GENERIC_TIMEZONE=Europe/Rome → mezzanotte italiana)
- PATCH `habit_tracker?done_today=eq.true` → `{"done_today": false}`
- Log del reset
- Workflow attivo su Railway (ID: SEJOMbdFCOoI8h5Y)

### Verifica
- GENERIC_TIMEZONE=Europe/Rome ✅ (aggiunto dall'utente nelle Variables Railway)
- Workflow attivo ✅
- Prossima esecuzione attesa: 01/04/2026 ore 00:01 CET
- Da verificare domattina che `habit_tracker.done_today` sia tutto `false`

## 11. ✅ Reset manuale abitudini (31/03/2026)
- 13 righe habit_tracker resettate a done_today=false
- xp_today è campo calcolato dalla vista, non resettabile

### Architettura decisa
- **Vision** (lungo termine): "Diventare imprenditore digitale di successo"
- **Milestone** (medio termine): "Lanciare Karta con 10 clienti paganti"
- **Action** (breve termine): task concreti collegati

### Schema
```sql
CREATE TABLE goals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  level TEXT NOT NULL CHECK (level IN ('vision', 'milestone', 'action')),
  parent_id UUID REFERENCES goals(id),
  life_area_id UUID REFERENCES life_areas(id),
  project_id UUID REFERENCES projects(id),
  progress NUMERIC(5,2) DEFAULT 0,
  status TEXT DEFAULT 'attivo' CHECK (status IN ('attivo','completato','abbandonato')),
  due_date DATE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Collegamento opzionale task → obiettivo
ALTER TABLE activities ADD COLUMN IF NOT EXISTS goal_id UUID REFERENCES goals(id);
```

### Calcolo progresso automatico
- Action: 0% o 100% (basato su status)
- Milestone: media % delle actions figlie
- Vision: media % dei milestones figli
- Implementabile come vista SQL o calcolo client-side

### Ordine implementazione
1. Tabella goals + colonna goal_id su activities (DDL)
2. Vista v_goal_progress (SQL)
3. Pagina Obiettivi nella webapp (separata da Aree Vita)
4. Integrazione bot Telegram (dopo)

### Pagina webapp
- Pagina dedicata /objectives (non integrata in Aree Vita)
- Visualizzazione: Vision → Milestones → Actions (accordion/albero)
- Barre progresso che si aggiornano automaticamente
- Link bidirezionale con Aree Vita (ogni obiettivo ha un'area)

### Stima: ~3-4 ore di lavoro

## 13. Sistema Link/Ispirazione Intelligente (da implementare)

### Problema attuale
Quando Gianmarco manda un link su Telegram:
- Salva titolo HTML + piattaforma + progetto "Generale"
- Nessuna analisi del contenuto
- Nessun riassunto
- Progetto e area sempre "Generale"
- Webapp read-only, no filtri, no edit
- I video YouTube non vengono trascritti
- Le conoscenze si perdono — Claude Code non sa cosa hai salvato

### Architettura nuova

```
FLUSSO LINK (2 fasi):

Fase 1 — Salvataggio immediato (< 2 secondi)
  Telegram link → Classifica → Supabase INSERT (titolo base, url, piattaforma)
  → Telegram: "🔗 Link salvato! Analizzo il contenuto..."
  → HTTP POST webhook al workflow "Link Analyzer" (background)

Fase 2 — Analisi background (30-120 secondi)
  Workflow "Link Analyzer" (separato, come Focus Reminder):
  → Jina AI Reader (GET https://r.jina.ai/{url}) → testo pulito markdown
  → [SE YouTube] YouTube Caption API → trascrizione completa
  → [SE no caption] Fallback: solo titolo + descrizione Jina
  → Groq LLaMA: analizza contenuto + genera riassunto strutturato
  → Supabase UPDATE inspiration (aggiunge summary, tools, project, area)
  → Telegram: riassunto formattato all'utente
  → Append a docs/knowledge-base.md (via Supabase Edge Function o batch)
```

### Schema DB — Modifiche alla tabella inspiration

```sql
ALTER TABLE inspiration ADD COLUMN IF NOT EXISTS summary TEXT;
ALTER TABLE inspiration ADD COLUMN IF NOT EXISTS tools_mentioned TEXT[];
ALTER TABLE inspiration ADD COLUMN IF NOT EXISTS life_area_id UUID REFERENCES life_areas(id);
ALTER TABLE inspiration ADD COLUMN IF NOT EXISTS key_takeaways TEXT[];
ALTER TABLE inspiration ADD COLUMN IF NOT EXISTS content_raw TEXT; -- testo completo estratto
ALTER TABLE inspiration ADD COLUMN IF NOT EXISTS processed BOOLEAN DEFAULT FALSE;
ALTER TABLE inspiration ADD COLUMN IF NOT EXISTS processed_at TIMESTAMPTZ;
```

### Formato riassunto (salvato in summary + mandato su Telegram)

```
📋 RIASSUNTO: "Come creare reel con AI in 5 minuti"
🎯 Takeaway: Usa CapCut + ChatGPT per script, Eleven Labs per voce
🔧 Tool: CapCut, Eleven Labs, Opus Clip
💡 Applicabile a: Karta (automatizzazione reel per clienti)
📁 Progetto: Karta | 🏷️ Area: Carriera Professionale
```

### Servizi esterni

| Servizio | Uso | Costo |
|----------|-----|-------|
| **Jina AI Reader** | Scraping qualsiasi URL → testo markdown | Gratuito (rate limit generoso) |
| **YouTube Caption API** | Trascrizione video YouTube | Gratuito |
| **Groq LLaMA 3.1** | Analisi contenuto + riassunto + classificazione | Gratuito |
| **Whisper (fallback)** | Solo se video senza caption | Già integrato via Edge Function |

### Jina AI Reader — Come funziona

```
GET https://r.jina.ai/https://example.com/article
→ Ritorna testo pulito in markdown, senza HTML/JS/CSS
→ Headers: Accept: text/plain
→ Rate limit: ~200 req/giorno free tier
```

Per YouTube:
```
GET https://r.jina.ai/https://youtube.com/watch?v=XYZ
→ Ritorna titolo + descrizione + trascrizione (se caption disponibili)
```

### Workflow "Link Analyzer" (nuovo, separato)

```
Nodi:
1. Webhook trigger (riceve: inspiration_id, url, platform)
2. Respond OK (non blocca il main workflow)
3. HTTP GET Jina Reader (https://r.jina.ai/{url})
4. [IF YouTube + no caption] → Whisper proxy (Edge Function)
5. Code: prepara prompt per Groq
6. HTTP POST Groq LLaMA: analizza + riassumi + classifica
7. Code: parsa risposta Groq (summary, tools, project, area, takeaways)
8. HTTP PATCH Supabase: aggiorna inspiration con tutti i campi
9. Telegram: manda riassunto formattato
```

### Prompt Groq per analisi link

```
Sei un assistente che analizza contenuti web per Gianmarco.

Gianmarco ha questi PROGETTI attivi:
- Karta (agenzia digitale, tool AI, clienti, reel, marketing)
- VisitLenola (associazione, turismo, Lenola Card)
- Casa (ristrutturazione appartamento)
- Indizi Cosmici (progetto creativo/spirituale)
- Libro (scrittura)
- CGA (cantiere, costruzioni)

E queste AREE DELLA VITA:
- Carriera Professionale
- Salute Fisica
- Finanze
- Relazione Amorosa
- Famiglia
- Sociale
- Vita Spirituale
- Creatività
- Sviluppo Personale

Analizza questo contenuto e rispondi in JSON:
{
  "summary": "riassunto in 2-3 frasi in italiano",
  "key_takeaways": ["takeaway 1", "takeaway 2", "takeaway 3"],
  "tools_mentioned": ["Tool1", "Tool2"],
  "suggested_project": "nome progetto dalla lista sopra o Generale",
  "suggested_area": "nome area dalla lista sopra",
  "actionable": true/false,
  "action_suggestion": "cosa potrebbe fare Gianmarco con questo contenuto"
}
```

### Webapp — Pagina Ispirazione migliorata

**Filtri (priorità alta):**
- Per progetto (tab come Missioni)
- Per piattaforma (Instagram/YouTube/TikTok/Web)
- Per area della vita
- Per "processato" (con/senza riassunto)
- Ricerca testo libera (cerca in titolo + summary + tools)

**Edit inline (priorità media):**
- Click su card → modal/drawer con tutti i campi
- Modifica: progetto, area, notes, summary
- Riassegna progetto/area
- Elimina con conferma

**Board view (priorità bassa, futuro):**
- Colonne per progetto
- Drag & drop tra progetti
- Contatore per colonna

### Knowledge Base — docs/knowledge-base.md

File auto-generato, appendato ogni volta che un link viene processato:

```markdown
# Knowledge Base — Tool e Idee Salvate

## 2026-04-01 — Come automatizzare reel con AI
- **Fonte:** https://youtube.com/watch?v=XYZ
- **Tool:** CapCut, Eleven Labs, Opus Clip
- **Progetto:** Karta
- **Takeaway:** Usa CapCut per editing automatico, Eleven Labs per voiceover, Opus Clip per tagliare long-form in shorts
- **Azione:** Implementare pipeline reel automatici per clienti Karta

## 2026-04-01 — Nuovo framework React per dashboard
- **Fonte:** https://example.com/article
- **Tool:** Tremor, shadcn/ui
- **Progetto:** Generale
- **Takeaway:** Tremor ha componenti chart pronti per dashboard analytics
- **Azione:** Valutare per prossima versione webapp
```

**Come Claude Code lo usa:**
- A inizio sessione legge knowledge-base.md
- Sa quali tool hai scoperto e per quale progetto
- Può suggerire: "Hai salvato un video su Opus Clip per reel — vuoi che lo integriamo in Karta?"

### Edge Function per knowledge-base.md

Opzione A: Edge Function `update-knowledge-base` che appende al file via GitHub API
Opzione B: Batch notturno (workflow n8n cron) che legge inspiration processate del giorno e genera/aggiorna il file

**Consiglio: Opzione B** (batch notturno) — più semplice, meno dipendenze, il file si aggiorna una volta al giorno.

### Ordine implementazione

1. **Nuove colonne DB** (DDL, 2 min)
2. **Workflow "Link Analyzer"** su Railway (n8n, 30 min)
3. **Modifica workflow principale** per trigger background (5 min)
4. **Filtri webapp** sulla pagina Ispirazione (20 min)
5. **Edit inline webapp** (15 min)
6. **Batch notturno knowledge-base.md** (20 min)
7. **Board view** (futuro)

### Stima totale: ~2 ore

### Dipendenze
- Jina AI: nessun account necessario per free tier
- YouTube Caption: nessun API key necessario (Jina lo gestisce)
- Groq: già integrato
- GitHub API: serve token per scrivere knowledge-base.md (o push da n8n via git)

---

## 12. Architettura Multi-livello

### Panoramica

```
Livello 1 — Sistema Personale (Gianmarco)     ← OGGI, funzionante
Livello 2 — Collaboratori (Francesco + altri)  ← PROSSIMO STEP
Livello 3 — Prodotto SaaS (utenti paganti)     ← FUTURO, repo separato
```

---

### 1. REPO STRUCTURE — Due repo o uno?

**Consiglio: DUE repo separati.**

| Aspetto | Monorepo | Due repo |
|---------|----------|----------|
| Complessità iniziale | Bassa (tutto insieme) | Media (setup doppio) |
| Scalabilità | Si ingarbuglia con auth/multi-tenant | Pulito, ogni repo ha il suo scope |
| Deploy | Complicato (Vercel deve capire quale buildare) | Semplice (un deploy per repo) |
| Condivisione codice | Facile (import diretto) | Serve pacchetto npm condiviso o copia |
| Collaboratori | Vedono tutto il tuo codice personale | Vedono solo il loro repo |
| SaaS futuro | Va estratto comunque | Già separato |

**Struttura consigliata:**

```
REPO 1: videogioco-vita (privato)
├── webapp/          ← React app personale (Vercel)
├── supabase/        ← Schema, migrations, edge functions
├── n8n-export/      ← Workflow export per backup
├── CLAUDE.md        ← Istruzioni per Claude Code
├── RIEPILOGO.md     ← Stato sistema
└── BLOCCO.md        ← Azioni pianificate

REPO 2: vita-saas (futuro, quando pronto)
├── webapp/          ← React app multi-tenant
├── supabase/        ← Schema con RLS per user_id
├── api/             ← Edge Functions per auth/billing
└── CLAUDE.md        ← Istruzioni specifiche SaaS
```

**Il codice per Francesco (Livello 2) vive nel REPO 1** perché è un'estensione del sistema personale, non un prodotto separato. Francesco ha la sua istanza n8n + il suo progetto Supabase, ma il codice webapp è lo stesso (deployato su un URL diverso o con una variabile d'ambiente che cambia il personaggio).

---

### 2. GITHUB + BRIDGE CON CLAUDE.AI

**Struttura repo per massimizzare il flusso Claude Code ↔ Claude.ai:**

```
videogioco-vita/
├── .claude/
│   └── CLAUDE.md              ← Regole per Claude Code (già esiste)
├── docs/
│   ├── RIEPILOGO.md           ← Stato sistema (aggiornato ogni sessione)
│   ├── BLOCCO.md              ← Azioni pianificate
│   ├── ARCHITETTURA.md        ← Decisioni architetturali (per Claude.ai)
│   └── CHANGELOG.md           ← Log di ogni sessione Claude Code
├── webapp/                     ← Codice React
├── supabase/                   ← Schema + Edge Functions
└── n8n-export/                 ← Workflow backup
```

**Flusso:**
1. Claude Code fa commit con messaggio strutturato: `feat: pagina obiettivi + fix classificatore`
2. Push su GitHub (manuale o automatico via hook)
3. Claude.ai legge i raw file da GitHub:
   - `https://raw.githubusercontent.com/gianmarcopandozi/videogioco-vita/main/docs/RIEPILOGO.md`
   - `https://raw.githubusercontent.com/gianmarcopandozi/videogioco-vita/main/docs/BLOCCO.md`
4. Claude.ai ha il contesto aggiornato senza dover rileggere tutto il codice

**CHANGELOG.md** è la chiave: ogni sessione Claude Code appende:
```
## 2026-03-31 — Sessione 3
- Fix classificatore (date, vizi, focus)
- Deploy Railway
- Pagina Obiettivi
- Focus reminder con workflow separato
```

Claude.ai legge CHANGELOG e sa cosa è cambiato senza leggere diff di codice.

---

### 3. SETUP FRANCESCO — Step esatti

**Prerequisiti:** Francesco ha un telefono con Telegram. Nient'altro.

| Step | Chi | Cosa | Tempo |
|------|-----|------|-------|
| 1 | Tu | Crea progetto Supabase "vita-francesco" | 5 min |
| 2 | Tu | Esegui schema.sql + schema_v2.sql + goals_schema.sql | 10 min |
| 3 | Tu | Seed: personaggio "Francesco", 50 livelli, 9 aree, abitudini personalizzate | 15 min |
| 4 | Tu | Crea bot Telegram via @BotFather (es. @FrancescoVitaBot) | 2 min |
| 5 | Tu | Deploy n8n su Railway (secondo template, progetto separato) | 10 min |
| 6 | Tu | Importa workflow con credenziali Francesco (token bot, URL Supabase) | 15 min |
| 7 | Tu | Cambia tutti i chatId hardcoded con il chat_id di Francesco | 10 min |
| 8 | Tu | Deploy Edge Function whisper-proxy su progetto Francesco | 5 min |
| 9 | Tu | Deploy webapp su Vercel con .env di Francesco | 5 min |
| 10 | Francesco | Apre il bot Telegram, manda "status" | 1 min |
| 11 | Francesco | Tu gli spieghi i comandi base | 10 min |

**Totale: ~90 minuti.** Francesco non deve fare nulla di tecnico.

**Costo mensile per Francesco:** ~$5 Railway + $0 Supabase free tier + $0 Vercel free tier = **~$5/mese**.

**Cosa può fare da solo:** Usare il bot Telegram e la webapp. Nient'altro.
**Cosa richiede il tuo aiuto:** Setup iniziale, personalizzazione abitudini, fix bug.

---

### 4. SINCRONIZZAZIONE KARTA E LIBRO

**Problema:** Francesco crea task "Prepara demo Karta per cliente X" → deve apparire nel tuo sistema. Tu crei "Review codice Francesco" → deve apparire nel suo.

**Soluzione più semplice: tabella condivisa su un terzo progetto Supabase.**

```
Supabase "vita-shared"
├── shared_tasks
│   ├── id UUID
│   ├── title TEXT
│   ├── project TEXT ('karta', 'libro')
│   ├── created_by TEXT ('gianmarco', 'francesco')
│   ├── assigned_to TEXT[] ('gianmarco', 'francesco')
│   ├── status TEXT
│   ├── due_date DATE
│   └── created_at TIMESTAMPTZ
```

**Sincronizzazione via webhook:**
1. Quando Gianmarco crea task con progetto "Karta" → n8n POST a `shared_tasks`
2. Quando Francesco crea task con progetto "Karta" → suo n8n POST a `shared_tasks`
3. Entrambe le webapp leggono `shared_tasks` filtrate per `assigned_to`
4. Nessun polling — i task condivisi sono su Supabase realtime (subscription)

**Alternativa ancora più semplice (V1):**
- Entrambi i sistemi puntano alla STESSA tabella `activities` con un campo `owner`
- RLS filtra per owner ma i task con progetto condiviso sono visibili a entrambi
- Nessun terzo DB, nessun webhook

**Consiglio:** parti con l'alternativa semplice (stesso DB, campo owner + RLS). Scala al terzo DB solo se servono più di 2-3 collaboratori.

---

### 5. QUANDO PASSARE AL SAAS

**Feature che si trasferiscono direttamente:**
- Schema DB (goals, habits, activities, xp_log) → identico, aggiungi `user_id`
- Webapp React → 95% riusabile, aggiungi auth (Supabase Auth)
- Edge Functions → riusabili con auth middleware
- Logica classificatore → copia del Code node

**Feature da riscrivere:**
- n8n workflow → non scala per multi-tenant. Serve un API backend (Edge Functions o Next.js API routes)
- Telegram bot → un bot per utente non scala. Serve un bot centralizzato con routing per chat_id
- Onboarding → non esiste, va costruito da zero
- Billing → Stripe integration da zero
- Admin dashboard → da zero

**Trigger per passare al SaaS:**
1. Il sistema personale funziona stabilmente da 1+ mese
2. 2-3 collaboratori lo usano senza problemi
3. Almeno 5 persone esterne chiedono di usarlo
4. Hai un prezzo in mente e un landing page

**NON passare al SaaS prima** di aver validato che il sistema funziona per te e per almeno un collaboratore per 30+ giorni. Il SaaS prematuro è la tomba dei side project.

---

### Prossimi passi consigliati

1. ~~Questa settimana: Crea repo GitHub~~ ✅ Fatto
2. **Prossima settimana:** Setup Francesco (90 min)
3. **Aprile:** Usa il sistema per 30 giorni, raccogli feedback
4. **Maggio:** Valuta SaaS se c'è domanda

---

## 14. Scraping Link Intelligente — Piano definitivo v2 (31/03/2026)

### Decisioni finali

| Piattaforma | Scraping | Trascrizione audio | Analisi |
|-------------|----------|-------------------|---------|
| Instagram | ScrapeCreators → caption + video_url | video_url → download → whisper-proxy → transcript | Caption + Transcript → Groq |
| TikTok | ScrapeCreators → caption + video_url | video_url → download → whisper-proxy → transcript | Caption + Transcript → Groq |
| YouTube | ScrapeCreators → caption + video_url | video_url → download → whisper-proxy → transcript | Caption + Transcript → Groq |
| Articoli/blog | Jina AI Reader → testo completo | N/A | Testo → Groq |

**Cambio da v1:** YouTube usa ScrapeCreators (Jina bloccato da YouTube).

### Architettura — Flusso completo con trascrizione audio

```
Link arriva su Telegram
    ↓
Classifica piattaforma (URL pattern)
    ↓
┌─ Instagram/TikTok/YouTube → ScrapeCreators API
│     ↓
│     Risposta: caption + video_url + autore + likes
│     ↓
│     Download video (HTTP Request binary da video_url)
│     ↓
│     Edge Function whisper-proxy (già esistente!)
│     ↓
│     Trascrizione completa del parlato
│     ↓
│     Caption + Trascrizione → Groq LLaMA analisi
│
└─ Articoli/blog → Jina AI Reader
      ↓
      Testo completo → Groq LLaMA analisi
    ↓
Risultato Groq: summary, takeaways, tools, project, area
    ↓
Supabase PATCH inspiration (+ aggiorna titolo!)
    ↓
Telegram: riassunto dettagliato
    ↓
docs/knowledge-base.md (batch notturno)
```

### Fix necessari (problemi trovati nel test)

1. **Titolo "Instagram"** — Il main workflow salva il titolo dalla pagina HTML.
   Il Link Analyzer deve anche PATCH il titolo con la caption o il summary.

2. **URL /reels/ → /reel/** — ScrapeCreators richiede `/reel/` non `/reels/`.
   ✅ Già fixato con `.replace('/reels/', '/reel/')` nel nodo.

3. **Riassunto generico** — Con solo la caption il riassunto è povero.
   Fix: aggiungere trascrizione audio del video per contenuto completo.

### Flusso video → trascrizione (dettaglio tecnico)

```
ScrapeCreators response
    ↓
Estrai video_url:
  - Instagram: data.xdt_shortcode_media.video_url
  - TikTok: data.video.play_addr (o data.video_url)
  - YouTube: ScrapeCreators ritorna video info (verificare struttura)
    ↓
HTTP Request GET video_url (binary, response format: file)
    ↓
Edge Function whisper-proxy (POST binary audio/video)
  - La Edge Function già accetta binary raw
  - Groq Whisper trascrive audio dal video
  - Ritorna { text: "trascrizione completa" }
    ↓
Normalizza: caption + "\n\nTRASCRIZIONE:\n" + whisper_text
    ↓
Groq LLaMA analisi (ora ha il contenuto completo)
```

### Nota: limiti Whisper
- Max 25MB per file audio (Groq Whisper limit)
- Video Instagram reel: ~2-5MB (ok)
- Video YouTube: potrebbe superare 25MB per video lunghi
- Soluzione: limitare a primi 10 minuti o comprimere

### ScrapeCreators — API Reference aggiornata

```
# Instagram post/reel
GET https://api.scrapecreators.com/v1/instagram/post?url={URL}
Headers: x-api-key: {API_KEY}
Response: { data: { xdt_shortcode_media: {
  edge_media_to_caption: { edges: [{ node: { text: "caption" } }] },
  video_url: "https://...",
  owner: { username: "..." },
  video_view_count: 12345
} } }

# TikTok video
GET https://api.scrapecreators.com/v1/tiktok/video?url={URL}
Headers: x-api-key: {API_KEY}
Response: (verificare struttura — TODO)

# YouTube video
GET https://api.scrapecreators.com/v1/youtube/video?url={URL}
Headers: x-api-key: {API_KEY}
Response: (verificare struttura — TODO)
```

### Costi stimati aggiornati

| Scenario | Link/mese | ScrapeCreators | Whisper (Groq) |
|----------|-----------|----------------|----------------|
| Uso leggero (5 social/giorno) | ~150 | ~$0.90 | Gratuito |
| Uso medio (15 social/giorno) | ~450 | ~$2.70 | Gratuito |
| Articoli/blog | illimitato | $0 (Jina) | N/A |

### Step implementazione IN ORDINE

**Step 1 ✅ — Account ScrapeCreators creato, API key attiva**

**Step 2 ✅ — Routing per piattaforma implementato nel workflow**
- Instagram → ScrapeCreators (con fix /reels/ → /reel/)
- TikTok → ScrapeCreators
- YouTube → Jina (da cambiare a ScrapeCreators)
- Web → Jina

**Step 3 — Aggiungere trascrizione audio al flusso**
- Dopo ScrapeCreators: estrarre video_url dalla risposta
- Download video come binary
- POST a whisper-proxy Edge Function
- Unire caption + trascrizione
- Mandare tutto a Groq per analisi completa

**Step 4 — Fix titolo**
- Link Analyzer PATCH anche il campo `title` con caption/summary

**Step 5 — YouTube su ScrapeCreators**
- Cambiare branch YouTube da Jina a ScrapeCreators
- Verificare endpoint e struttura risposta

**Step 6 — knowledge-base.md batch notturno**

### Stato attuale

- ✅ Link Analyzer workflow attivo con routing piattaforma
- ✅ ScrapeCreators Instagram funzionante (caption estratta)
- ✅ Groq analisi + Supabase PATCH + Telegram riassunto
- ✅ Webapp filtri + edit + ricerca
- ✅ Fix URL /reels/ → /reel/
- ⏳ Trascrizione audio video (Step 3)
- ⏳ Fix titolo (Step 4)
- ⏳ YouTube su ScrapeCreators (Step 5)
- ⏳ knowledge-base.md (Step 6)

