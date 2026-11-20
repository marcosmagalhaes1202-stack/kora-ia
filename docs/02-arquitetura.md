# Arquitetura — Kora IA (v1.0)

> Como as peças se encaixam. Banco: `03-banco-de-dados.md` · Agente: `04-agente-ia.md` · Integrações: `05-integracoes.md` · Infra: `07-infra-deploy.md`.

---

## 1. Visão geral

```
                ┌──────────────────────── VPS (Coolify / Docker) ────────────────────────┐
 Navegador/PWA  │                                                                          │
 ─────────────► │  apps/web (Next.js)  ── Server Actions ──► Redis (BullMQ) ◄── apps/worker │
   (Realtime)   │        │                                                    │   ▲  │      │
       ▲        │        │                               Fastify: /webhooks ──┘   │  │      │
       │        │        ▼                                                         │  ▼      │
       │        │   Supabase (fora da VPS): Postgres+RLS · Auth · Realtime ◄──────┘ Evolution API
       └────────┼────────────────────────────────────────────────────────────── (+ Postgres próprio)
                └──────────────────────────────────────────────────────────────────────────┘
 Serviços externos: Cloudflare R2 (mídias/backups) · Provedores de IA · ElevenLabs · ZapSign · Google Agenda · Web Push
```

| Peça | Responsabilidade |
|---|---|
| **apps/web** | Interface (escritório e master), leitura direta do Supabase com RLS, mutações por Server Actions, enfileiramento de jobs, rotas de mídia e OAuth. |
| **apps/worker** | Recebe webhooks, processa filas: entrada de mensagens, mídia, agente de IA, envio, follow-up, contratos, agenda, notificações, rotinas. Usa `service_role`. |
| **Supabase** | Fonte da verdade: dados, Auth, RLS, Realtime para a interface. |
| **Redis** | Filas BullMQ, travas por conversa, limites de envio, cache curto. |
| **Evolution API** | Ponte com o WhatsApp não oficial (uma instância por número). |
| **R2** | Arquivos (mídias, contratos assinados, PDFs, áudios gravados) e backups. |

**Princípio:** a web nunca fala com WhatsApp, IA ou ZapSign diretamente. Ela grava no banco e enfileira; o worker executa.

---

## 2. Estrutura de pastas (detalhe)

```
apps/web/src/
├─ app/
│  ├─ (auth)/entrar | esqueci-senha | redefinir-senha | convite/[token]
│  ├─ (app)/                      → layout com sidebar/topbar; exige sessão e escritório ativo
│  │  ├─ inbox/  [conversaId]/
│  │  ├─ funil/  [cardId]/         → detalhe do card em sheet (rota interceptada)
│  │  ├─ contatos/ agenda/ tarefas/ chat/ dashboard/ calculadoras/
│  │  ├─ agentes/ [agenteId]/ playground/
│  │  ├─ follow-up/
│  │  └─ configuracoes/ escritorio | equipe | canais | integracoes | etiquetas | respostas-rapidas
│  │                    | funil | contratos | honorarios | notificacoes
│  ├─ master/ escritorios | planos | pagamentos | templates | global | parametros | consumo
│  ├─ api/ midias/[id] | oauth/google/callback | push/inscrever
│  └─ design/                     → vitrine de componentes (só em dev)
├─ components/ ui/ (shadcn ajustado) · inbox/ · funil/ · card/ · shell/ · comum/
├─ lib/ supabase/{server,client,middleware}.ts · auth.ts (sessão, papel) · queue.ts · cripto.ts
├─ hooks/ (useRealtime..., useConversas...)
└─ styles/globals.css            → tokens do 06-design-ux.md

apps/worker/src/
├─ server.ts                      → Fastify: /health, /webhooks/*, /interno/* (JWT), /admin/filas (Bull Board)
├─ webhooks/ evolution.ts · zapsign.ts
├─ filas/ entrada.ts · midia.ts · agente.ts · enviar.ts · followup.ts · contratos.ts · agenda.ts
│         · notificacoes.ts · rotinas.ts
├─ servicos/ conversas.ts · contatos.ts · cards.ts · uso.ts · r2.ts · push.ts
└─ index.ts                       → inicia servidor e workers

packages/
├─ shared/  dominio/ (enums iguais aos do banco) · schemas/ (zod) · formatadores/ · constantes.ts
├─ db/      database.types.ts (gerado) · cliente.ts · consultas/ (sempre recebem escritorioId)
├─ channels/ tipos.ts (ChannelProvider, MensagemNormalizada) · evolution/ · meta/ (F5)
├─ ai/      montar-prompt.ts · executar-turno.ts · ferramentas/ · provedores.ts · voz.ts · transcricao.ts
│           · visao.ts · __tests__/cenarios/
├─ calc/    salario-maternidade.ts · bpc.ts · auxilio-acidente.ts · qualidade-segurado.ts · atrasados.ts
└─ queue/   filas.ts (nomes) · jobs.ts (tipos zod) · cliente.ts
```

---

## 3. Fluxos principais

### 3.1 Mensagem recebida (WhatsApp)
1. Evolution → `POST /webhooks/evolution/:canalId` (header `x-kora-secret`).
2. O worker valida o segredo e o payload (zod), grava o bruto em `webhook_eventos` (deduplicação por id externo) e responde **200 em < 100 ms**.
3. Fila `entrada`: normaliza (`MensagemNormalizada`) → upsert `contatos` / `contato_canais` → acha ou cria a `conversa` aberta → insere `mensagens` (único por `canal_id + id_externo`) → atualiza prévia e não lidas.
4. Mídia → fila `midia`: baixa da Evolution, sobe no R2, cria `midias`; áudio → transcrição; imagem/PDF → leitura (quando a IA vai responder).
5. Se `conversa.modo = 'ia'`, o agente está ativo e o escritório está ativo → fila `agente` com **debounce** (seção 4).
6. Realtime entrega a mudança para quem está com o Inbox aberto.

### 3.2 Turno do agente
Ver `04-agente-ia.md`. Resumo: trava da conversa → monta contexto → chama o modelo com ferramentas → recebe lista de mensagens curtas → **confere se chegou mensagem nova** (se sim, descarta e reagenda) → enfileira os envios com "digitando…" e pausas → grava `execucoes_agente` e `consumo_ia`.

### 3.3 Mensagem enviada por humano
Server Action `enviarMensagem` → confere sessão e papel → `conversa.modo = 'humano'` e `responsavel_id = eu` (se a IA estava atendendo) → insere `mensagens` (status `pendente`) → fila `enviar` → o worker envia pelo `ChannelProvider` → atualiza status e `id_externo`. Os webhooks de status atualizam entregue/lida.

### 3.4 Envio com limites
Fila `enviar`: antes de enviar, o worker consulta o limitador do canal no Redis (por minuto e por dia, configurável em `canais.limites`). Passou do limite → o job volta para a fila com atraso. Mensagens da IA para quem acabou de falar com o escritório têm prioridade sobre follow-ups e reativações.

### 3.5 Contrato
IA (ferramenta `gerar_contrato`) ou botão → fila `contratos` → ZapSign cria o documento a partir do modelo → grava `contratos` → envia o link na conversa. Webhook ZapSign → atualiza o status; quando assinado: baixa o PDF **na hora** (o link expira em 60 min) → R2 → move o card → boas-vindas → transferência para o 2º atendimento → `ia_controla = false` → receita prevista → notificações.

### 3.6 Follow-up
Evento (sem resposta, contrato enviado, documento pedido, campo de data) → cria `execucoes_followup` com `agendada_para` → job atrasado no BullMQ (`jobId = execucao.id`). Na hora de rodar, o worker **reconfere as regras de parada** antes de enviar. Uma mensagem recebida do cliente cancela as execuções pendentes que têm "parar se responder".

### 3.7 Contador de leads
Quando a IA envia a **primeira resposta** a um contato, o worker chama `app.contabilizar_lead(escritorio_id, contato_id)`, que só conta se não houver contagem do mesmo contato nos últimos 30 dias, e incrementa `uso_mensal`. Ao cruzar 80% e 100%, cria notificações. Regras em `09-planos-cobranca.md`.

---

## 4. Filas (BullMQ)

| Fila | Job | Concorrência | Observação |
|---|---|---|---|
| `entrada` | normalizar e gravar mensagem | 10 | idempotente por id externo |
| `midia` | baixar, subir no R2, transcrever | 4 | retry 5x, backoff exponencial |
| `agente` | turno da IA | 4 | **debounce por conversa**: `deduplication { id: conversaId, ttl: 8000, extend: true, replace: true }` (conferir suporte na versão do BullMQ; alternativa: remover o job atrasado e recriar) + trava Redis `lock:conversa:{id}` |
| `enviar` | enviar mensagem/mídia/áudio | 8 | limitador por canal; ordem garantida por conversa (encadear pelo `ordem` da resposta) |
| `followup` | passo de follow-up | 4 | jobs atrasados |
| `contratos` | criar documento ZapSign, baixar assinado | 2 | |
| `agenda` | criar/atualizar evento, lembretes | 2 | |
| `notificacoes` | push + sino | 8 | remove inscrição inválida (410) |
| `rotinas` | jobs repetidos | 1 | reconciliar status dos canais (5 min), alertas de "pago até" (diário), índices mensais (F4), limpeza de `webhook_eventos` (diário) |

- Todo job tem `escritorioId` no payload e é validado com zod (`packages/queue/jobs.ts`).
- Falhas finais vão para a lista de falhas e aparecem no **Bull Board** (`/admin/filas`, só master).

---

## 5. Realtime (interface)
- Supabase Realtime **Postgres Changes** nas tabelas `mensagens`, `conversas`, `cards`, `notificacoes`, `canais`, `mensagens_internas`, filtrando `escritorio_id=eq.<id>` (RLS também se aplica).
- A interface aplica o evento no cache do TanStack Query (sem refetch completo).
- 💡 Se o volume crescer, migrar para **Broadcast** a partir de triggers do banco (canal privado por escritório).
- "Digitando…" do cliente e da IA: Broadcast efêmero (não grava no banco).

---

## 6. Arquivos (R2)
- Bucket `kora-midias`, chave `{escritorio_id}/{aaaa}/{mm}/{uuid}.{ext}`. Bucket `kora-backups` separado.
- O navegador nunca recebe credenciais do R2. `GET /api/midias/:id` confere a sessão e o escritório e redireciona para uma **URL assinada de 5 min**.
- Upload pelo navegador (anexos, áudios gravados): Server Action gera uma URL assinada de PUT, depois registra em `midias`.
- Limite por arquivo: 25 MB (💡 ajustar).

---

## 7. Autenticação e autorização
- Supabase Auth (e-mail e senha) com `@supabase/ssr` (cookies) no Next.js.
- RLS por `escritorio_id` com funções `app.meu_escritorio_id()`, `app.meu_papel()`, `app.is_master()` (ver `03-banco-de-dados.md`, seção 3).
- Server Actions conferem o papel com `exigirPapel(['admin', ...])` antes de gravar.
- Rotas internas do worker (`/interno/*`, ex.: playground com streaming) exigem o JWT do Supabase no header e conferem o papel.
- Master: `perfis.is_master = true`. As telas `/master` usam o cliente `service_role` **só no servidor**, sempre com auditoria.

---

## 8. Erros, idempotência e observabilidade
- **Webhook:** grava o bruto antes de processar → nada se perde se o worker cair. Reprocessável.
- **Mensagens:** índice único `(canal_id, id_externo)`.
- **Turno da IA:** se o modelo falhar, tenta o modelo reserva do agente; se falhar de novo, transfere para humano com aviso interno (o cliente não recebe mensagem de erro).
- **Logs:** pino em JSON (stdout → Coolify), com `escritorio_id`, `conversa_id`, `job_id`. Sem dados pessoais.
- 💡 Sentry (plano grátis) para erros da web e do worker.
- **Saúde:** `GET /health` (worker) confere Redis e Supabase; Coolify reinicia se cair.

---

## 9. Variáveis de ambiente

**apps/web**
| Variável | Uso |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` | cliente Supabase (chave pública/"publishable") |
| `SUPABASE_SERVICE_ROLE_KEY` | só no servidor, telas master e tarefas administrativas |
| `REDIS_URL` | enfileirar jobs |
| `ENCRYPTION_KEY` | cifrar/decifrar credenciais (32 bytes em base64) |
| `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_MIDIAS` | URLs assinadas |
| `NEXT_PUBLIC_VAPID_PUBLIC_KEY` | inscrição de push |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | OAuth Google Agenda |
| `APP_URL`, `WORKER_URL` | URLs públicas |

**apps/worker**
| Variável | Uso |
|---|---|
| `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` | acesso ao banco |
| `REDIS_URL` | filas |
| `ENCRYPTION_KEY` | mesma chave da web |
| `R2_*` (+ `R2_BUCKET_BACKUPS`) | arquivos e backups |
| `EVOLUTION_URL`, `EVOLUTION_API_KEY` | API global da Evolution |
| `WEBHOOK_BASE_URL` | URL pública do worker para registrar webhooks |
| `ANTHROPIC_API_KEY`, `GOOGLE_GENERATIVE_AI_API_KEY`, `OPENAI_API_KEY`, `XAI_API_KEY` | provedores de IA (do dono do SaaS, D02) |
| `ELEVENLABS_API_KEY` | voz |
| `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` | Web Push |
| `ZAPSIGN_API_URL` | produção ou sandbox |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | renovar tokens do Google |
| `BULLBOARD_USER`, `BULLBOARD_PASS` | painel das filas |
| `LOG_LEVEL`, `SENTRY_DSN` (opcional) | observabilidade |
