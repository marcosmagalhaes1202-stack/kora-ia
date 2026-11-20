# Banco de dados — Kora IA (v1.0)

> Postgres no Supabase. Este documento é o **desenho de referência**. As migrations reais ficam em `supabase/migrations/`, criadas por fase (ver `ROADMAP.md`). Se a migration divergir deste doc, atualize o doc.
> Convenções: tabelas e colunas em português, snake_case, sem acento; `id uuid default gen_random_uuid()`; `criado_em`/`atualizado_em timestamptz`; dinheiro `numeric(12,2)`.

---

## 1. Mapa das tabelas

| Grupo | Tabelas | Fase |
|---|---|---|
| Núcleo | `planos`, `escritorios`, `perfis`, `setores`, `membros`, `convites`, `pagamentos_manuais`, `auditoria` | F0 |
| Uso e limites | `uso_mensal`, `leads_contabilizados`, `consumo_ia` | F1/F2 |
| Notificações | `notificacoes`, `push_inscricoes` | F0 |
| Canais e mensagens | `canais`, `webhook_eventos`, `contatos`, `contato_canais`, `conversas`, `mensagens`, `midias`, `respostas_rapidas`, `transferencias` | F1 |
| Funil | `funis`, `etapas`, `cards`, `card_eventos`, `documentos_card`, `tarefas`, `etiquetas`, `contato_etiquetas`, `card_etiquetas`, `visoes_salvas` | F1 |
| Chat interno | `canais_internos`, `participantes_internos`, `mensagens_internas` | F1 |
| Agentes | `config_global_ia`, `templates_produto`, `template_versoes`, `produtos`, `agentes`, `audios_gravados`, `conhecimento_escritorio`, `execucoes_agente` | F2 |
| Fechamento | `integracoes`, `modelos_contrato`, `contratos`, `reunioes`, `sequencias_followup`, `passos_followup`, `execucoes_followup`, `regras_honorarios`, `receitas` | F3 |
| Calculadoras | `parametros_legais`, `indices_mensais`, `calculos` | F4 |

---

## 2. Enums

```sql
create type status_escritorio  as enum ('teste','ativo','bloqueado','cortesia','cancelado');
create type papel_membro       as enum ('admin','advogado','comercial','atendimento','financeiro');
create type ref_pagamento      as enum ('mensalidade','implantacao','excedente','outro');
create type tipo_canal         as enum ('whatsapp_nao_oficial','whatsapp_oficial','instagram','messenger');
create type provedor_canal     as enum ('evolution','meta');
create type status_canal       as enum ('conectado','desconectado','aguardando_qr','erro');
create type status_conversa    as enum ('aberta','aguardando','encerrada');
create type modo_conversa      as enum ('ia','humano','pausada');
create type direcao_mensagem   as enum ('entrada','saida');
create type autor_mensagem     as enum ('contato','ia','membro','sistema','celular'); -- 'celular' = enviada pelo aparelho, fora do sistema
create type tipo_mensagem      as enum ('texto','audio','imagem','video','documento','sticker','localizacao','contato','reacao','sistema');
create type status_mensagem    as enum ('recebida','pendente','enviada','entregue','lida','falhou');
create type tipo_etapa         as enum ('entrada','comercial','fechamento','pos_venda','perdido');
create type status_card        as enum ('aberto','ganho','perdido','arquivado');
create type escopo_etiqueta    as enum ('contato','card');
create type status_documento   as enum ('pendente','recebido','validado','rejeitado');
create type tipo_agente        as enum ('produto','triagem');
create type regra_audio        as enum ('nunca','quando_cliente_pede','quando_cliente_manda_audio','etapas_marcadas');
create type gatilho_followup   as enum ('sem_resposta','contrato_pendente','documentos_pendentes','data_relativa','reativacao','manual');
create type tipo_passo         as enum ('texto_fixo','ia_gerada','audio_gravado','template_oficial');
create type status_execucao    as enum ('agendada','enviada','cancelada','pulada','falhou');
create type status_contrato    as enum ('gerado','enviado','visualizado','assinado','recusado','expirado','cancelado');
create type tipo_reuniao       as enum ('video','ligacao','presencial');
create type tipo_receita       as enum ('atrasados','parcela_mensal','beneficio_total','outro');
create type status_receita     as enum ('prevista','recebida','cancelada');
```
Os mesmos valores ficam em `packages/shared/dominio` (gerados de `database.types.ts`).

---

## 3. Funções de acesso e padrão de RLS

```sql
create schema if not exists app;

-- escritório do usuário logado (MVP: 1 escritório por usuário)
create or replace function app.meu_escritorio_id() returns uuid
language sql stable security definer set search_path = '' as $$
  select m.escritorio_id from public.membros m
  where m.user_id = auth.uid() and m.ativo limit 1
$$;

create or replace function app.meu_papel() returns public.papel_membro
language sql stable security definer set search_path = '' as $$
  select m.papel from public.membros m
  where m.user_id = auth.uid() and m.ativo limit 1
$$;

create or replace function app.is_master() returns boolean
language sql stable security definer set search_path = '' as $$
  select coalesce((select p.is_master from public.perfis p where p.user_id = auth.uid()), false)
$$;

grant usage on schema app to authenticated;
grant execute on all functions in schema app to authenticated;
```

**Política padrão** (toda tabela com `escritorio_id`):
```sql
alter table public.<tabela> enable row level security;

create policy "<tabela>_escritorio" on public.<tabela>
  for all to authenticated
  using      (escritorio_id = (select app.meu_escritorio_id()) or (select app.is_master()))
  with check (escritorio_id = (select app.meu_escritorio_id()) or (select app.is_master()));
```
- O `(select ...)` em volta da função faz o Postgres calcular uma vez por consulta (desempenho).
- **Políticas restritivas por papel** (somam-se à padrão), exemplo para tabelas só do admin:
```sql
create policy "<tabela>_so_admin_escreve" on public.<tabela> as restrictive
  for insert to authenticated
  with check ((select app.meu_papel()) = 'admin' or (select app.is_master()));
-- repetir para update e delete
```
- Aplicar "só admin escreve" em: `canais`, `integracoes`, `agentes`, `audios_gravados`, `conhecimento_escritorio`, `etapas`, `funis`, `modelos_contrato`, `sequencias_followup`, `passos_followup`, `regras_honorarios`, `setores`, `membros`, `convites`, `etiquetas`.
- `receitas`: comercial e atendimento só leem/escrevem receitas de cards em que são responsáveis.
- Tabelas **globais** (`planos`, `config_global_ia`, `templates_produto`, `template_versoes`, `parametros_legais`, `indices_mensais`): leitura para `authenticated`, escrita só master.
- `pagamentos_manuais`, `auditoria`, `webhook_eventos`, `consumo_ia`: leitura só master (e admin lê o próprio `consumo_ia` agregado via `uso_mensal`); escrita pelo worker/servidor.
- **Teste obrigatório:** para cada tabela nova, um teste com usuários de dois escritórios.

---

## 4. Núcleo

```sql
create table planos (
  id uuid primary key default gen_random_uuid(),
  nome text not null,
  limite_leads_mes int not null,
  limite_usuarios int not null,
  limite_numeros int not null,
  limite_agentes int,
  preco_referencia numeric(12,2),
  preco_lead_excedente numeric(12,2),
  ativo boolean not null default true,
  criado_em timestamptz not null default now()
);

create table escritorios (
  id uuid primary key default gen_random_uuid(),
  nome text not null,
  slug text unique not null,
  documento text,                         -- CNPJ/CPF do escritório
  oab text,
  status status_escritorio not null default 'teste',
  plano_id uuid references planos(id),
  pago_ate date,
  teste_ate date,
  limites_personalizados jsonb not null default '{}',  -- {limite_leads_mes, limite_usuarios, limite_numeros}
  fuso_horario text not null default 'America/Sao_Paulo',
  configuracoes jsonb not null default '{}',           -- logo, instagram, site, cores do PDF etc.
  observacoes text,
  criado_em timestamptz not null default now(),
  atualizado_em timestamptz not null default now()
);

create table perfis (
  user_id uuid primary key references auth.users(id) on delete cascade,
  nome text not null,
  avatar_url text,
  telefone text,
  is_master boolean not null default false,
  preferencias jsonb not null default '{}',             -- notificações, tema, visões
  criado_em timestamptz not null default now()
);

create table setores (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  nome text not null,                                   -- Comercial, Financeiro, Atendimento
  cor text,
  unique (escritorio_id, nome)
);

create table membros (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  papel papel_membro not null,
  setor_id uuid references setores(id) on delete set null,
  ativo boolean not null default true,
  criado_em timestamptz not null default now(),
  unique (user_id)                                      -- MVP: um escritório por usuário
);

create table convites (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  email text not null,
  papel papel_membro not null,
  setor_id uuid references setores(id),
  token_hash text not null unique,
  expira_em timestamptz not null,
  aceito_em timestamptz,
  convidado_por uuid references auth.users(id),
  criado_em timestamptz not null default now()
);

create table pagamentos_manuais (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  data date not null,
  valor numeric(12,2) not null,
  forma text not null,                                  -- pix, boleto, cartão, transferência
  referencia ref_pagamento not null,
  competencia date,                                     -- mês de referência
  observacao text,
  registrado_por uuid references auth.users(id),
  criado_em timestamptz not null default now()
);

create table auditoria (
  id bigint generated always as identity primary key,
  escritorio_id uuid references escritorios(id) on delete set null,
  user_id uuid references auth.users(id) on delete set null,
  acao text not null,                                   -- 'escritorio.bloquear', 'master.entrar_como', 'contrato.reenviar'...
  entidade text,
  entidade_id uuid,
  dados jsonb,
  ip inet,
  criado_em timestamptz not null default now()
);
```

---

## 5. Canais, contatos e mensagens

```sql
create table canais (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  tipo tipo_canal not null,
  provedor provedor_canal not null,
  nome text not null,                                   -- "Comercial 1"
  cor text,
  identificador text,                                   -- número E.164 / page id / ig id
  instancia text unique,                                -- nome da instância na Evolution
  credenciais_cifradas text,                            -- token da instância / token Meta (AES-GCM)
  webhook_segredo_hash text not null,
  status status_canal not null default 'aguardando_qr',
  status_detalhe text,
  agente_padrao_id uuid,                                -- FK adicionada na F2 (agentes)
  limites jsonb not null default '{"por_minuto":20,"por_dia":1000,"followup_por_dia":200}',
  ativo boolean not null default true,
  conectado_em timestamptz,
  criado_em timestamptz not null default now()
);

create table webhook_eventos (
  id bigint generated always as identity primary key,
  escritorio_id uuid references escritorios(id) on delete cascade,
  canal_id uuid references canais(id) on delete cascade,
  provedor text not null,                               -- 'evolution' | 'zapsign' | 'meta'
  tipo text not null,
  id_externo text,
  payload jsonb not null,
  processado_em timestamptz,
  erro text,
  criado_em timestamptz not null default now(),
  unique (provedor, id_externo)
);
-- contém dados pessoais: apagar após 7 dias (rotina diária)

create table contatos (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  nome text,                                            -- nome confirmado
  nome_perfil text,                                     -- nome do WhatsApp/Instagram
  telefone text,                                        -- E.164
  email text,
  avatar_midia_id uuid,
  origem jsonb not null default '{}',                   -- {canal_id, gatilho, anuncio, ctwa_clid}
  resumo_ia text,                                       -- memória curta do contato
  memoria jsonb not null default '{}',                  -- fatos estruturados (ver 04-agente-ia, seção 8)
  bloqueado boolean not null default false,             -- pediu para não receber mensagens
  ultimo_contato_em timestamptz,
  criado_em timestamptz not null default now(),
  atualizado_em timestamptz not null default now(),
  unique (escritorio_id, telefone)
);

create table contato_canais (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  contato_id uuid not null references contatos(id) on delete cascade,
  canal_id uuid not null references canais(id) on delete cascade,
  identificador_externo text not null,                  -- jid do WhatsApp / igsid / psid
  unique (canal_id, identificador_externo)
);

create table conversas (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  contato_id uuid not null references contatos(id) on delete cascade,
  canal_id uuid not null references canais(id) on delete cascade,
  status status_conversa not null default 'aberta',
  modo modo_conversa not null default 'ia',
  agente_id uuid,                                       -- FK na F2
  card_id uuid,                                         -- card ativo (FK após criar cards)
  responsavel_id uuid references membros(id) on delete set null,
  setor_id uuid references setores(id) on delete set null,
  nao_lidas int not null default 0,
  ultima_mensagem_em timestamptz,
  ultima_mensagem_previa text,
  ultima_entrada_em timestamptz,                        -- janela de 24h (API oficial) e SLA
  ultima_saida_em timestamptz,
  ia_pausada_ate timestamptz,
  criado_em timestamptz not null default now(),
  unique (canal_id, contato_id)                         -- uma conversa contínua por contato e número
);

create table midias (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  chave_r2 text not null unique,
  mime text not null,
  tamanho_bytes bigint,
  nome_arquivo text,
  duracao_seg numeric,
  sha256 text,
  criado_por uuid references auth.users(id),
  criado_em timestamptz not null default now()
);

create table mensagens (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  conversa_id uuid not null references conversas(id) on delete cascade,
  canal_id uuid not null references canais(id) on delete cascade,
  direcao direcao_mensagem not null,
  autor autor_mensagem not null,
  autor_membro_id uuid references membros(id) on delete set null,
  tipo tipo_mensagem not null,
  conteudo text,                                        -- texto ou legenda
  midia_id uuid references midias(id) on delete set null,
  transcricao text,                                     -- áudio
  leitura_ia jsonb,                                     -- imagem/documento: {descricao, tipo_documento, dados_extraidos, legivel}
  interna boolean not null default false,               -- nota interna (nunca vai ao cliente)
  responde_a_id uuid references mensagens(id) on delete set null,
  id_externo text,
  status status_mensagem not null,
  erro text,
  execucao_agente_id uuid,                              -- qual turno da IA gerou
  metadados jsonb not null default '{}',
  enviada_em timestamptz,                               -- horário do provedor
  criado_em timestamptz not null default now(),
  unique (canal_id, id_externo)
);

create table respostas_rapidas (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  atalho text not null,                                 -- "/docs"
  conteudo text,
  midia_id uuid references midias(id),
  unique (escritorio_id, atalho)
);

create table transferencias (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  conversa_id uuid not null references conversas(id) on delete cascade,
  de_ia boolean not null default false,
  de_membro_id uuid references membros(id),
  para_membro_id uuid references membros(id),
  para_setor_id uuid references setores(id),
  motivo text,
  resumo text,
  criado_em timestamptz not null default now()
);
```

---

## 6. Funil

```sql
create table funis (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  nome text not null default 'Funil',
  criado_em timestamptz not null default now()
);

create table etapas (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  funil_id uuid not null references funis(id) on delete cascade,
  nome text not null,
  chave text,                                           -- chave de sistema (abaixo); null em etapas livres
  tipo tipo_etapa not null,
  visao text not null default 'comercial',              -- 'comercial' | 'pos_venda'
  ordem int not null,
  cor text,
  ia_pode_mover boolean not null default false,
  unique (funil_id, chave)
);

create table cards (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  funil_id uuid not null references funis(id),
  etapa_id uuid not null references etapas(id),
  contato_id uuid not null references contatos(id) on delete cascade,
  conversa_id uuid references conversas(id) on delete set null,
  produto_id uuid,                                      -- FK na F2
  titulo text not null,
  responsavel_id uuid references membros(id) on delete set null,
  ia_controla boolean not null default true,            -- false depois do 2º atendimento
  passo_roteiro text,                                   -- ex.: 'E3.P2' (estado do roteiro)
  campos jsonb not null default '{}',                   -- dados coletados (schema do template)
  resumo_ia text,
  proximo_passo text,
  score int,                                            -- 0–100 chance de direito/fechamento
  valor_previsto numeric(12,2),
  valor_real numeric(12,2),
  status status_card not null default 'aberto',
  motivo_perda text,
  posicao text not null,                                -- fractional indexing (ordem na coluna)
  data_reuniao timestamptz,
  movido_em timestamptz not null default now(),
  arquivado_em timestamptz,
  criado_em timestamptz not null default now(),
  atualizado_em timestamptz not null default now()
);

create table card_eventos (
  id bigint generated always as identity primary key,
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  card_id uuid not null references cards(id) on delete cascade,
  tipo text not null,          -- 'criado','etapa','campo','etiqueta','documento','contrato','reuniao','nota','transferencia','receita'
  autor autor_mensagem not null,
  autor_membro_id uuid references membros(id),
  dados jsonb not null default '{}',                    -- {de_etapa, para_etapa, campo, valor...}
  criado_em timestamptz not null default now()
);

create table documentos_card (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  card_id uuid not null references cards(id) on delete cascade,
  tipo text not null,                                   -- 'rg_cnh','cpf','comprovante_endereco','certidao_nascimento'...
  obrigatorio boolean not null default true,
  status status_documento not null default 'pendente',
  midia_id uuid references midias(id),
  dados_extraidos jsonb,
  observacao text,
  atualizado_em timestamptz not null default now(),
  unique (card_id, tipo)
);

create table tarefas (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  card_id uuid references cards(id) on delete cascade,
  titulo text not null,
  descricao text,
  responsavel_id uuid references membros(id),
  vence_em timestamptz,
  concluida_em timestamptz,
  criado_por uuid references membros(id),
  criado_em timestamptz not null default now()
);

create table etiquetas (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  nome text not null,
  cor text not null,                                    -- token pastel (06-design-ux)
  escopo escopo_etiqueta not null,
  chave text,                                           -- etiquetas automáticas: 'pediu-audio','docs-pendentes'...
  unique (escritorio_id, escopo, nome)
);
create table contato_etiquetas (
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  contato_id uuid not null references contatos(id) on delete cascade,
  etiqueta_id uuid not null references etiquetas(id) on delete cascade,
  primary key (contato_id, etiqueta_id)
);
create table card_etiquetas (
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  card_id uuid not null references cards(id) on delete cascade,
  etiqueta_id uuid not null references etiquetas(id) on delete cascade,
  primary key (card_id, etiqueta_id)
);

create table visoes_salvas (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  user_id uuid references auth.users(id) on delete cascade,  -- null = visão do escritório
  tela text not null,                                   -- 'inbox' | 'funil'
  nome text not null,
  filtros jsonb not null,
  ordem int not null default 0
);
```

**Funil padrão** (criado com o escritório):

| ordem | nome | chave | tipo | visão | IA pode mover |
|---|---|---|---|---|---|
| 1 | Novo contato | `novo` | entrada | comercial | ✓ |
| 2 | Em qualificação | `qualificacao` | comercial | comercial | ✓ |
| 3 | Documentos | `documentos` | comercial | comercial | ✓ |
| 4 | Reunião agendada | `reuniao` | comercial | comercial | ✓ |
| 5 | Contrato enviado | `contrato_enviado` | fechamento | comercial | ✓ |
| 6 | Contrato assinado | `contrato_assinado` | fechamento | comercial | ✓ |
| 7 | 2º atendimento | `segundo_atendimento` | pos_venda | pos_venda | ✓ (só para entrar) |
| 8 | Protocolado | — | pos_venda | pos_venda | — |
| 9 | Exigência / Recurso | — | pos_venda | pos_venda | — |
| 10 | Concedido | — | pos_venda | pos_venda | — |
| 11 | Follow-up / Recuperação | `followup` | comercial | comercial | ✓ |
| 12 | Perdido / Sem direito | `perdido` | perdido | comercial | ✓ |

A IA usa sempre a **chave**, nunca o nome (o escritório pode renomear).

---

## 7. Chat interno

```sql
create table canais_internos (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  tipo text not null,                                   -- 'direto' | 'setor' | 'geral'
  nome text,
  setor_id uuid references setores(id) on delete cascade,
  criado_em timestamptz not null default now()
);
create table participantes_internos (
  canal_interno_id uuid not null references canais_internos(id) on delete cascade,
  membro_id uuid not null references membros(id) on delete cascade,
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  lido_ate timestamptz,
  primary key (canal_interno_id, membro_id)
);
create table mensagens_internas (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  canal_interno_id uuid not null references canais_internos(id) on delete cascade,
  autor_id uuid not null references membros(id),
  conteudo text not null,
  mencoes uuid[] not null default '{}',
  conversa_ref_id uuid references conversas(id) on delete set null,
  card_ref_id uuid references cards(id) on delete set null,
  criado_em timestamptz not null default now()
);
```
RLS extra: `direto` e `setor` só para participantes.

---

## 8. Agentes

```sql
create table config_global_ia (                         -- camada 1 (só master)
  id uuid primary key default gen_random_uuid(),
  versao int not null unique,
  prompt_md text not null,
  regras jsonb not null default '{}',                   -- limites, palavras proibidas, delays
  ativo boolean not null default false,
  criado_em timestamptz not null default now()
);

create table templates_produto (                        -- camada 2 (só master)
  id uuid primary key default gen_random_uuid(),
  chave text unique not null,                           -- 'salario-maternidade'
  nome text not null,
  area text not null default 'previdenciario',
  versao_publicada int not null default 1,
  criado_em timestamptz not null default now()
);
create table template_versoes (
  id uuid primary key default gen_random_uuid(),
  template_id uuid not null references templates_produto(id) on delete cascade,
  versao int not null,
  roteiro_md text not null,
  base_conhecimento_md text not null,
  campos_schema jsonb not null,                         -- JSON Schema dos campos do card
  documentos jsonb not null,                            -- checklist por situação
  gatilhos text[] not null default '{}',                -- frases do anúncio
  etiquetas_auto text[] not null default '{}',
  ferramentas text[] not null,                          -- ferramentas liberadas
  variaveis text[] not null,                            -- {{agente}}, {{escritorio}}, {{percentual}}...
  etapas_audio text[] not null default '{}',            -- etapas que aceitam áudio gravado (E1, E4-A...)
  followups_padrao jsonb not null default '[]',
  valor_previsto_padrao numeric(12,2),
  notas text,
  publicado_em timestamptz,
  unique (template_id, versao)
);

create table produtos (                                 -- produto ativado pelo escritório
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  template_id uuid references templates_produto(id),
  nome text not null,
  ativo boolean not null default true,
  valor_previsto_padrao numeric(12,2),
  unique (escritorio_id, template_id)
);

create table agentes (                                  -- camada 3
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  tipo tipo_agente not null,
  produto_id uuid references produtos(id) on delete cascade,
  template_versao int,                                  -- null = acompanha a publicada
  nome_atendente text not null,                         -- {{agente}}
  ativo boolean not null default false,
  modelo text not null,                                 -- 'anthropic:<modelo>' etc.
  modelo_reserva text,
  voz_id text,
  regra_audio regra_audio not null default 'quando_cliente_manda_audio',
  horarios jsonb not null default '{"sempre": true}',   -- ou {seg:[["08:00","18:00"]],...}
  mensagem_fora_horario text,
  variaveis jsonb not null default '{}',                -- {instagram, percentual, advogado, oab...}
  instrucoes_extras_md text,
  gatilhos_extras text[] not null default '{}',
  limite_mensagens_por_lead int not null default 80,
  criado_em timestamptz not null default now(),
  atualizado_em timestamptz not null default now()
);

create table audios_gravados (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  agente_id uuid not null references agentes(id) on delete cascade,
  chave_etapa text not null,                            -- 'E1', 'E4-A', 'E8'...
  midia_id uuid not null references midias(id),
  transcricao text,                                     -- a IA sabe o que o áudio diz
  ativo boolean not null default true,
  unique (agente_id, chave_etapa)
);

create table conhecimento_escritorio (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  agente_id uuid references agentes(id) on delete cascade,  -- null = todos os agentes
  titulo text not null,
  conteudo_md text not null,
  atualizado_em timestamptz not null default now()
);

create table execucoes_agente (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  conversa_id uuid not null references conversas(id) on delete cascade,
  agente_id uuid references agentes(id) on delete set null,
  mensagens_entrada uuid[] not null,
  resposta jsonb,                                       -- mensagens geradas
  ferramentas jsonb not null default '[]',              -- chamadas e resultados
  modelo text,
  descartada boolean not null default false,            -- chegou mensagem nova antes de enviar
  latencia_ms int,
  erro text,
  criado_em timestamptz not null default now()
);
```

---

## 9. Fechamento, agenda, follow-up e receita

```sql
create table integracoes (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  membro_id uuid references membros(id) on delete cascade,  -- Google: por membro
  tipo text not null,                                   -- 'zapsign' | 'google_agenda'
  credenciais_cifradas text not null,
  config jsonb not null default '{}',                   -- {calendar_id, email, sandbox}
  status text not null default 'ativa',
  criado_em timestamptz not null default now(),
  unique (escritorio_id, tipo, membro_id)
);

create table modelos_contrato (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  produto_id uuid references produtos(id) on delete cascade,
  nome text not null,                                   -- "Contrato + procuração SM"
  zapsign_template_id text not null,
  mapeamento jsonb not null,                            -- {"{{NOME}}": "campos.nome_completo", ...}
  opcoes jsonb not null default '{}',                   -- idioma, lembretes, envio automático
  ativo boolean not null default true
);

create table contratos (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  card_id uuid not null references cards(id) on delete cascade,
  modelo_id uuid references modelos_contrato(id),
  zapsign_doc_token text unique,
  status status_contrato not null default 'gerado',
  link_assinatura text,
  dados_preenchidos jsonb not null,
  arquivo_assinado_midia_id uuid references midias(id),
  enviado_em timestamptz,
  visualizado_em timestamptz,
  assinado_em timestamptz,
  criado_em timestamptz not null default now()
);

create table reunioes (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  card_id uuid references cards(id) on delete cascade,
  contato_id uuid not null references contatos(id) on delete cascade,
  responsavel_id uuid references membros(id),
  inicio timestamptz not null,
  fim timestamptz not null,
  tipo tipo_reuniao not null,
  link text,
  google_event_id text,
  status text not null default 'marcada',               -- marcada | realizada | faltou | cancelada
  criado_por_ia boolean not null default false,
  criado_em timestamptz not null default now()
);

create table sequencias_followup (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  produto_id uuid references produtos(id) on delete cascade,  -- null = todos
  nome text not null,
  gatilho gatilho_followup not null,
  config_gatilho jsonb not null default '{}',           -- {campo:'dpp', deslocamento:'-15 days'}
  janela jsonb not null default '{"inicio":"08:00","fim":"20:00","dias":[1,2,3,4,5,6]}',
  parar_se jsonb not null default '["respondeu","assinou","pediu_parar","humano","perdido"]',
  ativa boolean not null default true
);
create table passos_followup (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  sequencia_id uuid not null references sequencias_followup(id) on delete cascade,
  ordem int not null,
  atraso interval not null,                             -- depois do gatilho ou do passo anterior
  tipo tipo_passo not null,
  conteudo text,                                        -- texto fixo (com variáveis) ou instrução para a IA
  midia_id uuid references midias(id),
  unique (sequencia_id, ordem)
);
create table execucoes_followup (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  sequencia_id uuid not null references sequencias_followup(id) on delete cascade,
  passo_id uuid not null references passos_followup(id) on delete cascade,
  card_id uuid references cards(id) on delete cascade,
  conversa_id uuid not null references conversas(id) on delete cascade,
  agendada_para timestamptz not null,
  status status_execucao not null default 'agendada',
  motivo text,
  mensagem_id uuid references mensagens(id),
  criado_em timestamptz not null default now()
);

create table regras_honorarios (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  produto_id uuid references produtos(id) on delete cascade,  -- null = padrão do escritório
  base text not null default 'atrasados',               -- 'atrasados' | 'beneficio_total'
  percentual numeric(5,2) not null default 30,
  parcelas_futuras_qtd int not null default 0,
  parcelas_futuras_percentual numeric(5,2),
  informar_ao_cliente boolean not null default true,    -- D25: SM mostra a conta
  unique (escritorio_id, produto_id)
);

create table receitas (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  card_id uuid not null references cards(id) on delete cascade,
  contrato_id uuid references contratos(id) on delete set null,
  calculo_id uuid,                                      -- FK na F4
  tipo tipo_receita not null,
  descricao text,
  valor_previsto numeric(12,2) not null,
  valor_recebido numeric(12,2),
  previsto_para date,
  recebido_em date,
  status status_receita not null default 'prevista',
  origem text not null default 'padrao_produto',        -- padrao_produto | calculadora | manual
  criado_em timestamptz not null default now()
);
```

---

## 10. Uso, notificações e calculadoras

```sql
create table uso_mensal (
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  competencia date not null,                            -- 1º dia do mês
  leads int not null default 0,
  mensagens_ia int not null default 0,
  tokens_entrada bigint not null default 0,
  tokens_saida bigint not null default 0,
  tokens_cache bigint not null default 0,
  caracteres_voz bigint not null default 0,
  segundos_transcritos numeric not null default 0,
  custo_estimado_usd numeric(12,4) not null default 0,
  alerta_80_em timestamptz,
  alerta_100_em timestamptz,
  primary key (escritorio_id, competencia)
);

create table leads_contabilizados (
  id bigint generated always as identity primary key,
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  contato_id uuid not null references contatos(id) on delete cascade,
  contabilizado_em timestamptz not null default now()
);
create index on leads_contabilizados (contato_id, contabilizado_em desc);

create table consumo_ia (
  id bigint generated always as identity primary key,
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  conversa_id uuid references conversas(id) on delete set null,
  execucao_id uuid references execucoes_agente(id) on delete set null,
  tipo text not null,                                   -- 'texto' | 'visao' | 'transcricao' | 'voz'
  provedor text not null,
  modelo text,
  tokens_entrada int, tokens_saida int, tokens_cache int,
  caracteres int, segundos numeric,
  custo_usd numeric(12,6) not null,
  criado_em timestamptz not null default now()
);

create table notificacoes (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid references escritorios(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  tipo text not null,
  titulo text not null,
  corpo text,
  link text,
  lida_em timestamptz,
  criado_em timestamptz not null default now()
);
create table push_inscricoes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  endpoint text unique not null,
  p256dh text not null,
  auth text not null,
  user_agent text,
  criado_em timestamptz not null default now()
);
-- RLS de notificacoes e push_inscricoes: user_id = auth.uid()

-- calculadoras: ver 08-calculadoras.md, seção 7
create table parametros_legais (
  id uuid primary key default gen_random_uuid(),
  nome text not null,                                   -- 'salario_minimo','teto_rgps','limite_bpc_per_capita','teto_jef'
  valor numeric(14,4) not null,
  vigencia_inicio date not null,
  vigencia_fim date,
  fonte text not null,
  unique (nome, vigencia_inicio)
);
create table indices_mensais (
  indice text not null,                                 -- 'INPC','IPCA-E','SELIC','TAXA_LEGAL','POUPANCA'
  competencia date not null,
  valor numeric(12,8) not null,
  fonte text not null,
  primary key (indice, competencia)
);
create table calculos (
  id uuid primary key default gen_random_uuid(),
  escritorio_id uuid not null references escritorios(id) on delete cascade,
  card_id uuid references cards(id) on delete set null,
  tipo text not null,
  entradas jsonb not null,
  resultado jsonb not null,
  versao_motor text not null,
  parametros_usados jsonb not null,
  pdf_midia_id uuid references midias(id),
  criado_por uuid references membros(id),
  criado_em timestamptz not null default now()
);
```

### Função do contador de leads
```sql
create or replace function app.contabilizar_lead(p_escritorio uuid, p_contato uuid)
returns boolean language plpgsql security definer set search_path = '' as $$
declare v_comp date := date_trunc('month', now() at time zone 'America/Sao_Paulo')::date;
begin
  if exists (select 1 from public.leads_contabilizados
             where contato_id = p_contato and contabilizado_em > now() - interval '30 days') then
    return false;
  end if;
  insert into public.leads_contabilizados (escritorio_id, contato_id) values (p_escritorio, p_contato);
  insert into public.uso_mensal (escritorio_id, competencia, leads) values (p_escritorio, v_comp, 1)
    on conflict (escritorio_id, competencia) do update set leads = public.uso_mensal.leads + 1;
  return true;
end $$;
-- chamada só pelo worker (service_role); revogar execute de authenticated
```

### Função de limites efetivos
`app.limites_do_escritorio(p_escritorio uuid) returns jsonb` → para cada limite (`limite_leads_mes`, `limite_usuarios`, `limite_numeros`, `limite_agentes`), usa `escritorios.limites_personalizados` se houver; senão, o do plano. Usada pela web e pelo worker (`09-planos-cobranca.md`).

---

## 11. Índices essenciais

```sql
create index on conversas (escritorio_id, ultima_mensagem_em desc);
create index on conversas (escritorio_id, responsavel_id) where status <> 'encerrada';
create index on mensagens (conversa_id, criado_em desc);
create index on mensagens using gin (to_tsvector('portuguese', coalesce(conteudo,''))); -- busca
create index on contatos (escritorio_id, telefone);
create index on contatos using gin (to_tsvector('simple', coalesce(nome,'') || ' ' || coalesce(nome_perfil,'')));
create index on cards (escritorio_id, etapa_id, posicao) where arquivado_em is null;
create index on cards (contato_id);
create index on card_eventos (card_id, criado_em desc);
create index on execucoes_followup (status, agendada_para) where status = 'agendada';
create index on notificacoes (user_id, criado_em desc) where lida_em is null;
create index on consumo_ia (escritorio_id, criado_em);
create index on webhook_eventos (criado_em);
```
- Toda FK usada em filtro recebe índice (o Supabase Advisor aponta as que faltarem; rodar o **Advisor de segurança e desempenho** a cada fase).

## 12. Triggers
- `perfis` criado ao inserir em `auth.users` (nome do metadata do convite).
- `atualizado_em = now()` em update (tabelas com a coluna).
- Ao criar `escritorios`: cria funil + etapas padrão, setores padrão (Comercial, Atendimento, Financeiro), etiquetas automáticas e regras de honorários padrão.
- Ao inserir `mensagens` de entrada: atualiza `conversas` (prévia, `ultima_entrada_em`, `nao_lidas + 1`) e `contatos.ultimo_contato_em`. (Pode ficar no worker; escolher um lugar só.)
- Ao mudar `cards.etapa_id`: grava `card_eventos` e `movido_em`.

## 13. Realtime
Habilitar a publicação `supabase_realtime` para: `mensagens`, `conversas`, `cards`, `notificacoes`, `canais`, `mensagens_internas`, `contratos`.
