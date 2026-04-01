# Architettura — Videogioco della Vita

## Stack tecnologico
- **Bot**: Telegram → n8n (Railway) → Supabase
- **Webapp**: React + Vite + TypeScript + Tailwind (Vercel)
- **DB**: Supabase PostgreSQL (24 tabelle + goals)
- **AI**: Groq (Whisper per vocali, LLaMA per classificazione, Llama Scout per vision)
- **Proxy**: Supabase Edge Function "whisper-proxy" (bypassa bug gzip n8n)

## Architettura multi-livello
- **Livello 1** (attuale): Sistema personale Gianmarco
- **Livello 2** (prossimo): Collaboratori con Supabase separato + shared_tasks
- **Livello 3** (futuro): SaaS multi-tenant (repo separato vita-saas)

## Decisioni architetturali
- user_id in tutte le tabelle da subito
- chatId dinamico nei workflow (no hardcoded)
- Due repo: videogioco-vita (personale) + vita-saas (futuro)
- Collaboratori: Supabase separato + terzo "vita-shared" per task condivise

Dettagli completi in BLOCCO.md sezione 12.
