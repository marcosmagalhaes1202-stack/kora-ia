# Infraestrutura e deploy — Kora IA (v1.0)

> Decisões: D09, D10 (💡 a confirmar), D34. Custos e upgrade: `00-visao-geral.md`, seção 5.5.

---

## 1. Ambiente local (desenvolvimento)

| Peça | Como sobe |
|---|---|
| Supabase (Postgres, Auth, Realtime, Studio) | `pnpm db:start` (Supabase CLI, usa Docker) |
| Redis + Evolution API + Postgres da Evolution | `pnpm dev:infra` (`infra/docker-compose.dev.yml`) |
| Web (3000) e worker (4000) | `pnpm dev` |
| Webhooks de fora (ZapSign, celular real na Evolution) | túnel (Cloudflare Tunnel ou ngrok) apontando para o worker |
| R2 | bucket de desenvolvimento separado (`kora-midias-dev`) |

- `.env.local` de cada app a partir do `.env.example`.
- `pnpm db:reset` recria o banco com as migrations e o `seed.sql`.
- E-mails do Auth local caem no Inbucket/Mailpit do Supabase CLI.

---

## 2. Produção (fase inicial)

```
 Cloudflare (DNS + proxy + SSL)
   ├─ app.<dominio>  ─► VPS: web (Next.js)
   └─ api.<dominio>  ─► VPS: worker (Fastify: /webhooks, /interno, /health)

 VPS Hostinger KVM 2 (Brasil) · Ubuntu LTS · Coolify
   ├─ web          (Docker, Next.js standalone)
   ├─ worker       (Docker, Node + ffmpeg)
   ├─ redis        (senha, AOF ligado, volume persistente)
   ├─ evolution    (rede interna; sem domínio público)
   ├─ evolution-db (Postgres só da Evolution, volume persistente)
   └─ backup       (cron: pg_dump Supabase + evolution-db → R2)

 Supabase (projeto de produção, região São Paulo)
 Cloudflare R2 (kora-midias, kora-backups)
```

- **Evolution fica na rede interna do Docker.** O worker chama `http://evolution:8080` e a Evolution manda os webhooks para `http://worker:4000/webhooks/evolution/...`. Nada da Evolution exposto na internet. Para abrir o painel da Evolution, usar túnel SSH.
- **Orçamento de memória** (8 GB): Coolify ~1 GB · web ~0,5 GB · worker ~0,5 GB (picos de ffmpeg) · Redis ~0,2 GB · Evolution ~0,6 GB + instâncias · Postgres da Evolution ~0,3 GB. Folga para ~15–20 números.

### 2.1 Servidor
- Acesso SSH só por chave; usuário sem root; firewall liberando 22, 80 e 443; fail2ban; atualizações automáticas de segurança.
- Painel do Coolify com senha forte e 2FA; de preferência atrás do Cloudflare Access ou só por túnel.
- Swap de 2 GB.

### 2.2 Domínios
- `app.<dominio>` (web), `api.<dominio>` (worker). Domínio ainda **a definir** (PENDENCIAS).
- E-mail transacional com SPF, DKIM e DMARC no mesmo domínio.

### 2.3 Supabase de produção
- Região São Paulo. SMTP próprio no Auth. URLs de redirecionamento do Auth: `https://app.<dominio>/**`.
- Rodar o **Advisor** (segurança e desempenho) a cada fase.
- ⚠️ Plano Free: 500 MB de banco e **sem backup** → backup próprio (seção 3). Projeto Free pausa após 7 dias sem uso (não acontece com o sistema em produção, mas acontece no projeto de testes).

---

## 3. Backups

| O quê | Como | Frequência | Retenção |
|---|---|---|---|
| Banco Supabase | `pg_dump` (conexão direta) → `.sql.gz` → R2 `kora-backups/supabase/` | diário, 03:00 | 30 dias (regra de ciclo de vida do R2) |
| Postgres da Evolution | `pg_dump` → R2 `kora-backups/evolution/` | diário | 7 dias |
| Redis | AOF no volume (filas se recuperam; não precisa de backup externo) | — | — |
| Mídias | já estão no R2 | — | conforme `10-seguranca-lgpd.md` |

- **Teste de restauração mensal**: restaurar o último dump num Supabase local e conferir contagens. Registrar em `PROGRESSO.md`.
- Com Supabase Pro: backups diários do próprio Supabase + manter o nosso semanal.

---

## 4. CI/CD (GitHub Actions)
- **Em todo PR:** `pnpm install --frozen-lockfile` → `typecheck` → `lint` → `test` (com Supabase local para os testes de RLS).
- **Na `main`:**
  1. `supabase db push` no projeto de produção (segredos: `SUPABASE_ACCESS_TOKEN`, `SUPABASE_DB_PASSWORD`, `SUPABASE_PROJECT_REF`).
  2. Chamar o webhook de deploy do Coolify para web e worker.
- Migrations são sempre **compatíveis com a versão anterior do código** (adicionar antes, remover depois), porque o banco atualiza antes dos apps.
- Ambiente de homologação: depois do piloto (projeto Supabase separado + apps com sufixo `-hml`).

---

## 5. Monitoramento
- Health check do Coolify em `/health` (worker) e `/api/health` (web), com reinício automático.
- Monitor de disponibilidade externo (ex.: Uptime Kuma na própria VPS ou serviço grátis) para `app` e `api`, com alerta por e-mail/WhatsApp do dono.
- Alertas internos (notificação no painel master): canal desconectado há mais de 10 min, fila com jobs falhando, backup do dia ausente, disco acima de 80%.
- Bull Board em `api.<dominio>/admin/filas` (senha).

---

## 6. Dockerfiles (orientação)
- **web:** multi-stage, `output: 'standalone'`, usuário não root, `NODE_ENV=production`.
- **worker:** multi-stage, instala `ffmpeg`, usuário não root, `node dist/index.js`.
- Build com Turborepo `prune` para levar só o que cada app usa.

---

## 7. Quando crescer
| Sinal | Ação |
|---|---|
| 2–3 escritórios pagando, ou banco > 350 MB | Supabase **Pro** |
| RAM da VPS > 75% sustentado ou > 15 números | VPS maior (KVM 4) ou Evolution numa VPS separada |
| Muitos eventos de Realtime | Migrar para Broadcast (ver `02-arquitetura.md`, seção 5) |
| Tabela `mensagens` muito grande | Particionar por mês |
