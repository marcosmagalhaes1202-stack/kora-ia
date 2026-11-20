# Kora IA — CRM Jurídico

> Instruções permanentes para o Claude Code. Leia este arquivo inteiro no início de toda sessão.

## 1. O que é

**Kora IA** é um CRM multi-tenant para escritórios de advocacia (foco inicial: **previdenciário**). Junta:

- **Inbox omnichannel:** WhatsApp não oficial (Evolution) no início; API oficial, Instagram e Messenger depois.
- **Kanban único** que a IA cria e move.
- **Agentes de IA por produto** que atendem o lead do anúncio até o contrato assinado.
- **Follow-up**, **contratos ZapSign**, **Google Agenda**, **receita prevista**, **calculadoras** e **painel master** (dono do SaaS).

Visão completa: `docs/00-visao-geral.md`.

## 2. Antes de qualquer tarefa

1. Leia `docs/PROGRESSO.md`: onde paramos e qual é o próximo passo.
2. Leia a fase atual em `docs/ROADMAP.md` e pegue a **primeira tarefa não marcada**.
3. Leia o documento do módulo da tarefa (tabela da seção 9).
4. Se algo não estiver definido, consulte `docs/PENDENCIAS.md`. **Não invente regra de negócio nem regra jurídica.** Registre a dúvida em PENDENCIAS e siga com a opção mais simples e reversível, marcada com `// TODO(pendencia): ...`.

## 3. Ao terminar cada tarefa

1. Rode `pnpm typecheck && pnpm lint && pnpm test`. Tudo verde.
2. Marque `[x]` na tarefa em `docs/ROADMAP.md`.
3. Acrescente uma entrada em `docs/PROGRESSO.md`: data, o que foi feito, decisões, onde parou, próximo passo.
4. Decidiu algo de arquitetura ou produto? Registre em `docs/DECISOES.md`.
5. Faça commit pequeno e descritivo, em português: `feat(inbox): lista de conversas com filtros`.

## 4. Stack

| Camada | Escolha |
|---|---|
| Monorepo | **pnpm workspaces + Turborepo** |
| Web | **Next.js** (App Router, TypeScript strict), **Tailwind CSS v4**, **shadcn/ui** (Radix), **lucide-react**, TanStack Query, TanStack Virtual, Zustand (só estado de interface), react-hook-form + zod, **dnd-kit** (Kanban), Sonner (toasts), **Serwist** (PWA) |
| Worker | **Node.js LTS + TypeScript**, **Fastify** (webhooks e API interna), **BullMQ + Redis** (filas, atrasos, agendamentos), pino (logs) |
| Banco | **Supabase**: Postgres + RLS, Auth (e-mail e senha), Realtime. pgvector fica para depois. |
| Arquivos | **Cloudflare R2** (S3 API) para mídias, contratos e backups. Não usar o Supabase Storage para mídia. |
| IA | **Vercel AI SDK** (`ai`), com provedores Anthropic, Google, OpenAI e xAI atrás de `packages/ai` |
| Voz | ElevenLabs (TTS) por trás de interface própria |
| WhatsApp | **Evolution API** self-hosted atrás da interface `ChannelProvider` (`packages/channels`) |
| Testes | **Vitest** (unidade e integração); Playwright (e2e, a partir da F1) |
| Qualidade | **Biome** (lint + format), TypeScript `strict: true` |
| Deploy | VPS + **Coolify** (Docker). Ver `docs/07-infra-deploy.md` |

Use a **versão estável mais recente** de cada pacote no momento da criação. Registre as versões principais em `docs/PROGRESSO.md`.

## 5. Estrutura do repositório

```
kora-ia/
├─ CLAUDE.md
├─ docs/                      → documentação (fonte da verdade)
├─ apps/
│  ├─ web/                    → Next.js: app do escritório + painel master (/master)
│  └─ worker/                 → Fastify + BullMQ: webhooks, filas, IA, follow-up, jobs
├─ packages/
│  ├─ shared/                 → tipos de domínio, schemas zod, constantes, formatadores (BRL, telefone, datas)
│  ├─ db/                     → tipos gerados do Supabase + helpers de consulta
│  ├─ channels/               → ChannelProvider + EvolutionProvider (+ MetaProvider depois)
│  ├─ ai/                     → motor do agente: montagem do prompt, ferramentas, provedores, voz, transcrição
│  ├─ calc/                   → motor de cálculo jurídico puro (sem I/O), 100% testado
│  └─ queue/                  → nomes de filas, tipos de jobs e cliente BullMQ compartilhado
├─ supabase/
│  ├─ migrations/             → SQL versionado (única forma de mudar o banco)
│  ├─ seed.sql                → dados de desenvolvimento
│  └─ config.toml
└─ infra/
   ├─ docker-compose.dev.yml  → Redis + Evolution API + Postgres da Evolution (dev)
   └─ coolify/                → notas e arquivos de deploy
```

## 6. Comandos

```bash
pnpm i                      # instalar
pnpm dev                    # web + worker em modo dev (turbo)
pnpm dev:infra              # sobe Redis + Evolution (docker compose)
pnpm db:start               # supabase start (local)
pnpm db:new <nome>          # nova migration
pnpm db:reset               # recria o banco local com migrations + seed
pnpm db:types               # gera packages/db/src/database.types.ts
pnpm typecheck | lint | test | build
```
Se algum comando ainda não existir, crie o script no `package.json` raiz.

## 7. Convenções

**Idioma**
- Interface, mensagens, commits e documentação: **português do Brasil**.
- Banco: tabelas e colunas em **português, snake_case, sem acento** (`conversas`, `ultima_mensagem_em`).
- Código: entidades de domínio mantêm o nome em português (`conversa`, `card`, `etapa`); termos técnicos podem ficar em inglês (`handler`, `provider`, `queue`).

**Banco e multi-tenant (inegociável)**
- Toda tabela de dados do cliente tem `escritorio_id uuid not null` + **RLS habilitada** + política padrão (ver `docs/03-banco-de-dados.md`).
- O worker usa `service_role`, que ignora RLS. Por isso **toda consulta do worker filtra `escritorio_id` explicitamente**. Use os helpers de `packages/db` que exigem `escritorioId`.
- Mudança de banco só por migration em `supabase/migrations`. Depois: `pnpm db:types`.
- Datas em `timestamptz` (UTC). Exibição no fuso do escritório (`escritorios.fuso_horario`, padrão `America/Sao_Paulo`).
- Dinheiro em `numeric(12,2)` em reais. Nunca `float`.

**Código**
- TypeScript strict; sem `any` (use `unknown` + zod).
- Toda entrada externa (webhook, formulário, resposta de IA) é validada com zod.
- Server Components por padrão; `"use client"` só quando precisa de interação.
- Mutação na web: **Server Actions** com verificação de sessão e papel. Ações com efeitos externos (enviar mensagem, gerar contrato) enfileiram job em `packages/queue`.
- Segredos nunca chegam ao navegador. Só `NEXT_PUBLIC_SUPABASE_URL` e `NEXT_PUBLIC_SUPABASE_ANON_KEY` são públicas.
- Credenciais de terceiros (ZapSign, Google, Evolution) ficam no banco **cifradas** (AES-256-GCM, chave em `ENCRYPTION_KEY`). Ver `docs/10-seguranca-lgpd.md`.
- Webhooks são **idempotentes** (deduplicar por id externo) e respondem rápido: validam, gravam e enfileiram.
- Jobs têm retry com backoff e são idempotentes.
- Logs com pino, sempre com `escritorio_id` e `conversa_id` quando houver. **Nunca logar** documento, CPF, conteúdo de mensagem ou token.

**Interface**
- Siga `docs/06-design-ux.md` (tokens, componentes, padrões). Não invente cores fora dos tokens.
- Componentes shadcn ficam em `apps/web/src/components/ui` e são ajustados aos tokens do Kora.
- Mobile first nas telas de Inbox e Kanban (PWA). Área de toque ≥ 44 px.
- Todo estado vazio, de carregamento e de erro é desenhado.

## 8. Regras de produto que o código precisa garantir

1. **Isolamento total entre escritórios.** Teste de RLS para cada tabela nova.
2. **A IA nunca pede senha** (gov.br, banco) e nunca promete resultado. Valores são sempre **estimativa** calculada pelo motor (`packages/calc`), nunca inventados.
3. **Uma pergunta por vez** e mensagens curtas (regra da camada Global do agente).
4. **Depois do 2º atendimento, a IA não move mais o card** (`cards.ia_controla = false`).
5. **Humano assumiu, IA pausa.** A IA só volta quando alguém devolve a conversa para ela.
6. **WhatsApp não oficial:** respeitar limites de envio por número (anti-banimento). Ver `docs/05-integracoes.md`.
7. **Parâmetros legais** (salário mínimo, teto etc.) vêm da tabela `parametros_legais`, nunca fixos no código.
8. **LGPD:** dados de saúde e documentos são sensíveis. Acesso por papel, auditoria e retenção (ver `docs/10-seguranca-lgpd.md`).

## 9. Mapa da documentação

| Arquivo | Conteúdo |
|---|---|
| `docs/00-visao-geral.md` | Propósito, módulos, planos, riscos |
| `docs/01-requisitos.md` | Requisitos por módulo, critérios de aceite, papéis e permissões |
| `docs/02-arquitetura.md` | Componentes, fluxos, filas, realtime, variáveis de ambiente |
| `docs/03-banco-de-dados.md` | Schema completo (SQL), RLS, índices |
| `docs/04-agente-ia.md` | Motor dos agentes: 3 camadas, ferramentas, debounce, áudio, memória, custos |
| `docs/05-integracoes.md` | Evolution, ZapSign, ElevenLabs, Google Agenda, Web Push, R2, IA, Meta (futuro) |
| `docs/06-design-ux.md` | Marca Kora IA, tokens, tipografia, componentes, telas |
| `docs/07-infra-deploy.md` | VPS, Coolify, domínios, backups, ambiente local |
| `docs/08-calculadoras.md` | Regras e parâmetros das calculadoras |
| `docs/09-planos-cobranca.md` | Planos, limites, contador de leads, consumo de IA, painel master |
| `docs/10-seguranca-lgpd.md` | Autenticação, papéis, cifragem, LGPD, auditoria |
| `docs/templates/<produto>/` | Roteiro e base de conhecimento de cada agente |
| `docs/ROADMAP.md` | Fases e tarefas `[ ]` / `[x]` |
| `docs/PROGRESSO.md` | Diário de sessões |
| `docs/DECISOES.md` / `docs/PENDENCIAS.md` | Decisões tomadas / o que falta definir |

A cópia oficial da documentação também fica no Project do claude.ai **"CRM Advogados"**. Mudou um doc aqui? Mantenha os dois iguais.
