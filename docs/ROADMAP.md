# Roadmap de construção — Kora IA

> Uma tarefa = uma sessão curta do Claude Code, com entrega testável. Marque `[x]` ao concluir e registre em `PROGRESSO.md`.
> **Marco "Piloto":** F0 a F3 prontos → 1º escritório usando de verdade.
> Referências: `CLAUDE.md` · `02-arquitetura.md` · `03-banco-de-dados.md`

---

## F0 — Base (fundação)

### F0.1 Monorepo
- [ ] Criar o monorepo pnpm + Turborepo com `apps/web`, `apps/worker` e `packages/{shared,db,channels,ai,calc,queue}`.
- [ ] TypeScript strict compartilhado (`tsconfig.base.json`), Biome, Vitest, scripts da raiz (CLAUDE.md, seção 6).
- [ ] `.env.example` em cada app, com todas as variáveis de `02-arquitetura.md`, seção 9.
- [ ] `infra/docker-compose.dev.yml` com Redis e Evolution API (+ Postgres da Evolution).
- **Aceite:** `pnpm dev` sobe web (porta 3000) e worker (porta 4000, `GET /health` → 200); `pnpm test` roda.

### F0.2 Banco: núcleo multi-tenant
- [ ] `supabase init`; migration `0001_nucleo`: extensões, schema `app`, funções `app.meu_escritorio_id()`, `app.meu_papel()`, `app.is_master()`, enums do núcleo.
- [ ] Tabelas `planos`, `escritorios`, `perfis`, `setores`, `membros`, `convites`, `pagamentos_manuais`, `auditoria` + RLS.
- [ ] Trigger que cria `perfis` ao criar usuário no Auth; trigger genérico de `atualizado_em`.
- [ ] `seed.sql`: 3 planos, 1 escritório de teste, 1 usuário master e 1 admin do escritório.
- [ ] Testes de RLS (pgTAP ou Vitest com dois usuários de escritórios diferentes).
- **Aceite:** usuário do escritório A não lê nem altera nada do escritório B; master lê tudo.

### F0.3 Autenticação e acesso
- [ ] Login com e-mail e senha, "esqueci minha senha", sessão SSR (`@supabase/ssr`).
- [ ] Middleware: sem sessão → `/entrar`; escritório `bloqueado` → tela de acesso suspenso; `/master` só para master.
- [ ] Convite de membro por e-mail (link com token) e aceite com definição de senha.
- [ ] Tela "Equipe": listar, convidar, mudar papel e setor, desativar.
- **Aceite:** fluxo completo de convite funcionando local; papéis respeitados nas ações.

### F0.4 Design system e layout
- [ ] Tokens do `06-design-ux.md` no Tailwind v4 (`@theme`), fontes Inter e Montserrat via `next/font`.
- [ ] Componentes base ajustados: Button, Input, Select, Badge/Etiqueta, Avatar, Card, Tabs/Segmented, Dialog, Sheet, Dropdown, Tooltip, Toast, Skeleton, EmptyState, IaSelo.
- [ ] Shell do app: sidebar recolhível, topbar com busca (⌘K), sino de notificações, menu do usuário; navegação inferior no celular.
- [ ] Página `/design` (só em dev) mostrando todos os componentes.
- **Aceite:** shell responsivo de 360 px a 1440 px; foco visível; contraste AA.

### F0.5 Painel master
- [ ] `/master/escritorios`: lista com plano, status, "pago até", leads do mês / limite.
- [ ] Criar e editar escritório; limites personalizados; bloquear/liberar; observações.
- [ ] `/master/planos`: CRUD de planos.
- [ ] Registro manual de pagamentos (mensalidade, implantação, excedente) com atualização opcional do "pago até".
- [ ] `/master/parametros`: tabela `parametros_legais` (entra na F4, mas a tela nasce aqui vazia).
- [ ] Auditoria das ações do master.
- **Aceite:** o master cria um escritório, convida o admin, registra um pagamento e bloqueia/libera o acesso.

### F0.6 PWA e notificações push
- [ ] Manifest, ícones (a partir da logo), service worker com Serwist; instalar no Android e no iPhone (tela de início).
- [ ] Tabelas `notificacoes` e `push_inscricoes`; pedir permissão; envio via `web-push` (worker).
- [ ] Central de notificações (sino) com Realtime.
- **Aceite:** notificação de teste chega no celular com o app fechado.

### F0.7 Deploy
- [ ] VPS + Coolify; apps web e worker; Redis; Evolution + Postgres; domínios e HTTPS (ver `07-infra-deploy.md`).
- [ ] Projeto Supabase de produção; migrations aplicadas pelo CI.
- [ ] GitHub Actions: typecheck, lint, test em cada PR; deploy automático da `main`.
- [ ] Job diário de backup (`pg_dump` → R2) com retenção de 30 dias + teste de restauração documentado.
- **Aceite:** produção no ar com login funcionando e backup do dia no R2.

---

## F1 — Atendimento (inbox + funil)

### F1.1 Canais WhatsApp (Evolution)
- [ ] `ChannelProvider` + `EvolutionProvider` (enviar texto, mídia, áudio, "digitando", marcar lido, baixar mídia, status).
- [ ] Tela "Canais": conectar número por QR code, status em tempo real, reconectar, desconectar.
- [ ] Webhook `/webhooks/evolution/:canalId` com segredo, idempotente; eventos de mensagem, status e conexão.
- [ ] Limite de números por plano.
- **Aceite:** conectar um número e ver o status mudar sozinho na tela.

### F1.2 Entrada e saída de mensagens
- [ ] Normalizar mensagem recebida → `contatos`, `contato_canais`, `conversas`, `mensagens`; mídia baixada e salva no R2 (`midias`).
- [ ] Envio pela fila `enviar-mensagem` com status (pendente → enviada → entregue → lida / falhou) e reenvio.
- [ ] Mensagens enviadas pelo próprio celular do escritório também aparecem no sistema (`fromMe`).
- [ ] Rotas `/api/midias/:id` com URL assinada e checagem de escritório.
- **Aceite:** conversa de ida e volta com texto, áudio, imagem e PDF, sem duplicar mensagem.

### F1.3 Inbox
- [ ] Lista de conversas (virtualizada): foto, nome, prévia, horário, não lidas, canal/número, selo IA/humano, etiquetas.
- [ ] Filtros: canal, número, status, modo (IA/humano), responsável, setor, etiqueta, produto, "sem resposta há mais de X". Visões salvas.
- [ ] Painel da conversa: bolhas, áudio com player, imagens, documentos, respostas, status, "digitando…".
- [ ] Composer: texto, emoji, anexo, gravar áudio, respostas rápidas com `/`, nota interna.
- [ ] Painel lateral do contato/card (resumo da IA, campos, documentos, etiquetas, linha do tempo).
- [ ] Realtime: novas mensagens e mudanças de conversa sem recarregar.
- [ ] Layout mobile: lista → conversa → detalhes.
- **Aceite:** 2 atendentes no mesmo escritório veem a conversa atualizar em tempo real; busca acha por nome e telefone.

### F1.4 Assumir, devolver e transferir
- [ ] Botões "Assumir" (pausa a IA) e "Devolver para a IA"; enviar mensagem como humano assume automaticamente.
- [ ] Transferir para membro ou setor com motivo e resumo; notificação para quem recebe.
- [ ] Histórico em `transferencias` e na linha do tempo.
- **Aceite:** com a IA pausada, nenhuma resposta automática sai.

### F1.5 Funil (Kanban)
- [ ] Funil padrão criado com o escritório (etapas de `03-banco-de-dados.md`, seção 6).
- [ ] Quadro com colunas, arrastar e soltar (dnd-kit), ordenação persistente, contagem e soma por coluna.
- [ ] Card: nome, produto, etiquetas, responsável, selo de quem controla (IA/equipe), valor previsto, tempo na etapa, próximo passo.
- [ ] Visões "Comercial" e "Pós-venda", filtros e busca; modo lista.
- [ ] Detalhe do card (sheet): campos, documentos, contratos, reuniões, tarefas, receita, linha do tempo.
- [ ] Editar etapas (nome, cor, ordem, tipo, se a IA pode mover).
- **Aceite:** mover card registra evento na linha do tempo e aparece para os outros em tempo real.

### F1.6 Etiquetas, tarefas e respostas rápidas
- [ ] CRUD de etiquetas (contato e card), cores pastel, etiquetas automáticas reservadas.
- [ ] Tarefas por card com responsável e prazo; "Minhas tarefas".
- [ ] Respostas rápidas com texto e mídia.
- **Aceite:** filtro por etiqueta funciona no Inbox e no Kanban.

### F1.7 Chat interno
- [ ] Conversas diretas, por setor e geral; @menção com notificação; link para conversa de cliente.
- **Aceite:** menção gera push para o membro citado.

### F1.8 Contador de leads e limites
- [ ] `leads_contabilizados` + `uso_mensal`: conta o contato novo atendido no mês (regra em `09-planos-cobranca.md`).
- [ ] Alertas de 80% e 100% (escritório e master).
- **Aceite:** contador do master bate com a contagem manual no seed de teste.

---

## F2 — Agente de IA

### F2.1 Templates e agentes
- [ ] Tabelas `config_global_ia`, `templates_produto`, `template_versoes`, `produtos`, `agentes`, `audios_gravados`, `conhecimento_escritorio`.
- [ ] Importar o template `salario-maternidade` a partir de `docs/templates/salario-maternidade/` (script de seed).
- [ ] `/master/templates`: editar camada Global e templates, publicar nova versão.
- [ ] Central de agentes do escritório: ativar produto, personalizar (nome, variáveis, horários, modelo, voz, áudio), subir áudios gravados por etapa.
- **Aceite:** escritório ativa o agente de salário-maternidade e vê o prompt final montado (pré-visualização).

### F2.2 Motor da conversa
- [ ] Roteamento: gatilho do anúncio → agente do produto; sem gatilho → triagem.
- [ ] Debounce (espera o cliente terminar de mandar mensagens) e trava por conversa.
- [ ] Montagem do prompt em 3 camadas + contexto do card + memória; cache de prompt.
- [ ] Resposta estruturada em várias mensagens curtas, com "digitando…" e pausas naturais.
- [ ] Ferramentas da seção 6 de `04-agente-ia.md` (salvar campo, mover etapa, etiquetar, pedir documento, calcular, transferir, agendar etc.).
- [ ] Registro de `execucoes_agente` e `consumo_ia`.
- **Aceite:** conversa de teste completa do roteiro do salário-maternidade até o pedido de documentos, uma pergunta por vez.

### F2.3 Áudio, imagem e documento
- [ ] Transcrição de áudio recebido; leitura de imagem/PDF (RG, CNH, comprovante) com extração de dados.
- [ ] Validação do documento (é o documento pedido? legível?) e checklist do card.
- [ ] Resposta em áudio (ElevenLabs) conforme regra; envio de áudios gravados por etapa.
- **Aceite:** cliente manda foto do RG → IA confirma os dados e marca o documento como recebido.

### F2.4 Handoff, memória e qualidade
- [ ] Transferência automática nos casos 🔴 do template, com resumo.
- [ ] Horário de atendimento da IA (fora do horário: mensagem configurada).
- [ ] Memória do contato (resumo + fatos) atualizada ao fim de cada bloco de conversa.
- [ ] Playground: conversar com o agente sem WhatsApp.
- [ ] Guardas: nunca pedir senha, não prometer resultado, limite de mensagens por lead.
- **Aceite:** suíte de conversas-teste (`packages/ai/__tests__/cenarios`) passa.

---

## F3 — Fechamento

### F3.1 ZapSign
- [ ] Integração por escritório (token), modelos por produto e mapeamento de variáveis → campos do card.
- [ ] Ferramenta `gerar_contrato` (IA) e botão manual; envio do link no WhatsApp.
- [ ] Webhooks: criado, visualizado, assinado, recusado → card, notificações, dashboard; salvar PDF assinado no R2 na hora.
- [ ] Mensagem de boas-vindas, transferência para o 2º atendimento e `ia_controla = false`.
- **Aceite:** contrato assinado no sandbox move o card para "2º atendimento" e notifica a equipe.

### F3.2 Google Agenda e reuniões
- [ ] OAuth Google por membro; criar evento (com link do Meet opcional) ao marcar reunião.
- [ ] Ferramenta `agendar_reuniao` com horários livres; lembretes para cliente e equipe.
- **Aceite:** reunião marcada pela IA aparece na agenda do advogado.

### F3.3 Follow-up
- [ ] Sequências por gatilho (sem resposta, contrato pendente, documentos pendentes, data relativa como DPP, reativação).
- [ ] Passos: texto fixo, texto gerado pela IA, áudio gravado; janela de horário; regras de parada.
- [ ] Agendamento com BullMQ (jobs atrasados); cancelamento automático quando o cliente responde.
- [ ] Sequências padrão do template de salário-maternidade.
- **Aceite:** sem resposta por 2h → follow-up enviado; cliente responde → resto da sequência cancelado.

### F3.4 Receita prevista
- [ ] `regras_honorarios` por produto; receita criada ao assinar (valor padrão ou cálculo); edição manual; marcar como recebida.
- **Aceite:** contrato de salário-maternidade gera receita prevista de 30% do total estimado.

---

## F4 — Gestão

### F4.1 Dashboard
- [ ] Leads, conversas, contratos gerados e assinados por produto, canal e período; taxa de conversão por etapa; tempo de primeira resposta.
- [ ] Receita prevista × realizada por mês; consumo de IA do escritório.

### F4.2 Calculadoras — fase 1
- [ ] `packages/calc`: salário-maternidade, BPC (renda), auxílio-acidente, qualidade de segurado, atrasados/valor da causa/RPV/honorários, com testes de referência.
- [ ] `parametros_legais` e `indices_mensais` (job mensal BCB/IBGE).
- [ ] Telas das calculadoras, salvar no card, PDF e resumo para WhatsApp; ferramenta `calcular_estimativa` usa o mesmo motor.

---

## F5 — Depois do piloto
- [ ] Cobrança automática (Asaas ou Stripe).
- [ ] WhatsApp API oficial; Instagram Direct; Messenger.
- [ ] Meta Ads + Conversions API (atribuição por anúncio).
- [ ] Templates BPC/LOAS e auxílio-acidente (após os roteiros).
- [ ] Calculadoras fase 2 e 3; trabalhista e criminal.
- [ ] Busca semântica (pgvector) se as bases de conhecimento crescerem.
- [ ] Modo escuro.
