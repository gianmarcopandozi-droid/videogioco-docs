# Setup Francesco Marino — Guida Step by Step

**Tempo stimato: 2-2.5 ore** (prima volta, poi 30 min per altri collaboratori)

## Prerequisiti
- PC di Francesco (Windows con WSL2) con connessione internet
- Account email per i servizi

## Step 1 — Creare Bot Telegram (5 min)
1. Francesco apre Telegram, cerca @BotFather
2. Manda `/newbot`
3. Nome: `Francesco Vita Bot` (o altro)
4. Username: `francesco_vita_bot` (deve essere unico)
5. **Salva il token** (es. `1234567890:ABCdefGHIjklMNOpqrsTUVwxyz`)

## Step 2 — Creare Progetto Supabase (10 min)
1. Vai su https://supabase.com → Sign Up (se non ha account)
2. New Project → nome: `life-game-francesco`
3. Region: `eu-west-1` (Europa)
4. Genera password DB e salvala
5. Aspetta che il progetto sia pronto (~2 min)
6. **Salva:** Project URL + Anon Key (da Settings → API)

## Step 3 — Eseguire Schema SQL (15 min)
Nel SQL Editor di Supabase, esegui IN ORDINE:
1. `supabase/schema.sql` — tabelle v1
2. `supabase/schema_v2.sql` — tabelle v2
3. `supabase/goals_schema.sql` — obiettivi
4. `docs/migrations/add-user-id.sql` — colonna user_id
5. `docs/migrations/shopping-list.sql` — lista spesa
6. `scripts/seed-francesco.sql` — dati personaggio Francesco

## Step 4 — Deploy n8n su Railway (15 min)
1. Vai su https://railway.app → Sign Up
2. Scegli piano Hobby ($5/mese)
3. Deploy template n8n: https://railway.app/template/n8n
4. Nelle Variables aggiungi:
   - `WEBHOOK_URL` = URL pubblico Railway
   - `GENERIC_TIMEZONE` = `Europe/Rome` (NON funziona, vedi nota)
5. **NOTA:** I cron vanno in UTC. CEST = UTC+2.

## Step 5 — Importare Workflow (20 min)
1. Dal Mac di Gianmarco, esegui:
   ```bash
   cd ~/Desktop/SISTEMA/videogioco
   ./scripts/setup-collaborator.sh
   ```
2. Lo script chiede: nome, token bot, chat_id, URL Supabase, anon key
3. Genera i workflow con il token bot di Francesco
4. In n8n Railway di Francesco:
   - Vai su Credentials → Add → Telegram API → incolla token bot
   - Import workflow uno alla volta dalla cartella generata
5. Attiva tutti i workflow

## Step 6 — Deploy Edge Function Whisper (5 min)
Dal Mac di Gianmarco con Supabase CLI:
```bash
supabase functions deploy whisper-proxy --project-ref <ID_PROGETTO_FRANCESCO>
```
Oppure deployare via MCP se disponibile.

## Step 7 — Deploy Webapp su Vercel (10 min)
**Prerequisito:** Francesco deve avere un account GitHub e essere aggiunto come collaboratore al repo da Gianmarco prima di questo step.
1. Francesco va su https://vercel.com → Sign Up
2. Dal PC di Francesco:
   ```bash
   git clone https://github.com/gianmarcopandozi-droid/videogioco-vita.git
   cd videogioco-vita/webapp
   cp .env.example .env
   # Edita .env con URL e anon key del SUO Supabase
   npx vercel login
   npx vercel --prod
   ```

## Step 8 — Test Telegram (5 min)
Francesco manda al suo bot:
- `status` → deve vedere "Francesco Marino — Livello 1"
- `ho fatto meditazione` → deve registrare abitudine
- Un vocale → deve trascrivere e classificare
- Un link Instagram → deve analizzare con ScrapeCreators

## Step 9 — Setup Claude Code (15 min)
1. Francesco installa Claude Code sul suo PC (Windows con WSL2)
2. Clona il repo: `git clone videogioco-vita`
3. Il CLAUDE.md condiviso guida Claude Code

## Step 10 — Setup Claude.ai (10 min)
Vedi docs/SETUP-CLAUDE-FRANCESCO.md

---

## Limitazioni note (prima settimana)
- Le task condivise (Karta, Libro) NON si sincronizzano ancora
- Francesco e Gianmarco coordinano le task a voce/Telegram
- Il webhook di sync sarà implementato questa settimana
- I cron vanno convertiti manualmente in UTC (GENERIC_TIMEZONE non funziona)

## Credenziali da salvare
- [ ] Token bot Telegram: ___
- [ ] Chat ID Telegram: ___
- [ ] Supabase URL: ___
- [ ] Supabase Anon Key: ___
- [ ] Railway URL: ___
- [ ] Vercel URL: ___
