# Kora IA — Visão Geral e Requisitos (v0.6)

> **Produto:** **Kora IA — CRM Jurídico** (nome definido em 20/11/2026, D29).
> **Status:** descoberta concluída; documentação de construção pronta (20/11/2026). Próximo passo: `ROADMAP.md` → F0.1.
> **Convenção:** 💡 = **sugestão ainda não aprovada**. ✅ = **decidido**. Itens sem marca foram definidos pelo dono do produto na descrição inicial. ⚠️ = restrição/risco técnico.
> **Ver também:** `CLAUDE.md` · `docs/ROADMAP.md` · `docs/PROGRESSO.md` · `docs/DECISOES.md` · `docs/PENDENCIAS.md` · `docs/01` a `docs/10` · Whiteboard da arquitetura: https://claude.ai/artifact/SsxMBaojAsFQLv4zrUjXhT

---

## 1. Propósito

Não é só uma caixa de entrada de mensagens: é um **sistema completo de gestão comercial** para escritórios de advocacia, com um **agente de IA que atende do primeiro contato até o contrato assinado**, e que **potencializa os resultados do tráfego pago**.

- **Modelo:** SaaS multi-tenant, entregue aos clientes advogados junto com o serviço de tráfego.
- ✅ **Custo de IA embutido no preço do plano.** O sistema mede o consumo por escritório e aplica limites por plano.
- **Público:** advogados e escritórios — do jovem advogado que faz tudo sozinho ao escritório com equipe.
- ✅ **Foco inicial: previdenciário.** Agentes, templates e calculadoras primeiro para o previdenciário; depois viram modelo para outras áreas.
- ✅ **Templates iniciais:** salário-maternidade (pronto: roteiro v0.2 + base v1.1), BPC/LOAS, auxílio-acidente.
- ✅ **Base dos agentes:** roteiros de atendimento do dono + pesquisa de informações atualizadas.

---

## 2. Dores do público → resposta do sistema

| Dor | Como o sistema resolve |
|---|---|
| Advogado sozinho faz tudo (atende, qualifica, cobra documento, envia contrato) | Agente de IA como "funcionário a mais", ponta a ponta |
| Lead do tráfego esfria pela demora na resposta | Resposta imediata, 24/7 |
| Cada produto jurídico exige um atendimento diferente | Agentes por produto, com roteiro e base de conhecimento próprios |
| Vendas perdidas por falta de follow-up | Cadências automáticas + reativação de base |
| Mensagens espalhadas (vários números, Instagram, Facebook) | Inbox unificado com filtros |
| Não sabe em que etapa está cada lead | Kanban preenchido e atualizado pela IA |
| Não sabe quanto vai entrar de honorários | Receita prevista automática (atrasados + parcelas mensais) |
| Passagem comercial → jurídico sem contexto | Resumo da conversa no card + transferência formal |
| Cálculos jurídicos em ferramentas separadas | Calculadoras ligadas ao card, com PDF e envio por WhatsApp |
| 💡 Não sabe qual anúncio gera **contrato** | Atribuição por anúncio + Conversions API (fase posterior) |

---

## 3. Módulos

### 3.1 Inbox omnichannel
- Canais: **WhatsApp** (API oficial **e** não oficial, vários números), **Instagram Direct**, **Messenger**.
- ✅ Ordem: WhatsApp não oficial primeiro; API oficial, Instagram e Messenger em fase posterior.
- Ver todas as conversas juntas ou filtrar por canal / por número.
- Foto do contato, nome, design moderno e elegante (ver `06-design-ux.md`).
- Enviar e receber texto, áudio, imagem e documento.
- Notas internas, respostas rápidas com `/`, @menção, selo "IA atendendo / Humano atendendo", busca, não lidas, tempo sem resposta.

### 3.2 Pipeline (Kanban) ✅ quadro único
- **Um único quadro** com todos os contatos e cards.
- A IA **cria e move os cards automaticamente** enquanto acompanha a conversa.
- Depois que o contrato é assinado e o card passa para o **2º atendimento**, a IA **nunca mais move aquele card**; a equipe assume.
- Funil padrão (editável): ver `03-banco-de-dados.md`, seção 6.
- Visões "Comercial" e "Pós-venda" no mesmo quadro + arquivamento; o card mostra quem controla (IA ou equipe).
- Card: produto, origem, responsável, etiquetas, documentos, contrato, reunião, **receita prevista/real**, **cálculos salvos**, linha do tempo, resumo IA, próximo passo.

### 3.3 Etiquetas, filtros e organização
Pedido explícito: **máximo capricho**.
- Etiquetas manuais + automáticas (produto, "pediu áudio", "documento pendente", "sem direito"…).
- Filtros combináveis e visões salvas ("Meus atendimentos", "Contratos aguardando assinatura", "Leads sem resposta > 2h", "Documentos pendentes", "Reuniões de hoje").

### 3.4 Follow-up e reativação
- Cadências configuráveis (mensagens, intervalos), executadas pela IA; reativação de contatos.
- Acompanhamento pós-envio do contrato com mensagem curta se a pessoa demora.
- Regras de parada, janela de horário, limite diário por número.
- ⚠️ API não oficial: limites obrigatórios contra banimento. API oficial (depois): fora de 24h só templates pagos.

### 3.5 Central de Agentes de IA (detalhes em `04-agente-ia.md`)
**Definido:** vários agentes por produto; base de conhecimento sempre consultada; roteiro sequencial com checagem das respostas; **uma pergunta por vez** e mensagens curtas (D26); entende/envia áudio, imagem e documento; áudios gravados por etapa (D27); voz (ElevenLabs ou similar); modelos à escolha (credenciais do dono); memória do cliente; atendimento ponta a ponta até a transferência para o 2º atendimento; handoff humano ↔ IA.

✅ **3 camadas:** Global (só o dono edita) → Template do produto → Personalização do escritório. Prompt montado na hora.

✅ **Roteamento:** mensagem pré-preenchida do anúncio ativa o agente do produto; sem ela, triagem identifica o interesse.

✅ **Documentos antes do contrato** (alimentam o preenchimento). Leitura automática de RG/CNH e comprovante → pede só confirmação.

✅ **A IA pode informar o valor estimado do benefício**, usando o motor das calculadoras, sempre como estimativa e sem prometer resultado. **Honorários explicados em linguagem simples** (D25).

### 3.6 Equipe, setores e comunicação interna
- Notificações internas, comunicação entre membros, transferência de atendimentos.
- Setores: comercial, financeiro, atendimento (2º atendimento jurídico). Até ~10 usuários.
- Permissões por papel (`01-requisitos.md`, seção 1); alerta de SLA.

### 3.7 Integrações (detalhes em `05-integracoes.md`)
- **ZapSign:** ✅ IA coleta dados e documentos, preenche o modelo e envia o link; eventos atualizam card, dashboard e notificações.
- ✅ **Google Agenda:** reunião marcada vira evento com alerta.
- **WhatsApp:** Evolution API no início (💡 a confirmar); oficial depois.
- **Meta (Instagram/Messenger):** depois.
- **Voz:** ElevenLabs ou outra.
- ✅ **Tráfego (Meta Ads + Conversions API):** depois, módulo opcional.

### 3.8 Dashboard e notificações
- Contratos gerados e assinados por produto; notificações em tempo real.
- ✅ Receita prevista × realizada.
- Métricas por canal, produto, etapa, agente, atendente; motivos de perda; consumo de IA.

### 3.9 Multi-tenant, painel master e cobrança ✅ (detalhes em `09-planos-cobranca.md`)
- Cada escritório isolado. Login com e-mail e senha.
- ✅ **Cobrança manual no início:** o dono gera a cobrança fora do sistema e **libera/bloqueia o acesso manualmente**. Cobrança automática depois.
- ✅ **Implantação:** R$ 1.500 (pagamento único).
- Painel master: planos, status, "pago até", limites personalizados, pagamentos manuais, contador de leads (alertas 80%/100%), consumo de IA, templates, parâmetros legais, "entrar como" (auditado).

### 3.10 Mobile
- **PWA** instalável com push (no iPhone, exige adicionar à tela de início).

### 3.11 Financeiro ✅
- **Receita prevista** por contrato: **atrasados** (padrão 30%) + **parcelas futuras mensais** (a definir por produto). **Salário-maternidade:** 30% do valor total do benefício (D23).
- Valor padrão por produto quando ainda não há cálculo; com cálculo, usa o resultado da calculadora. A equipe edita o valor real.

### 3.12 Calculadoras jurídicas ✅ (escopo em `08-calculadoras.md`)
- Previdenciário primeiro; fases 1 (atendimento), 2 (processo), 3 (planejamento). PDF e resumo por WhatsApp; salvar no card.

---

## 4. Requisitos não funcionais
Ver `01-requisitos.md`, seção 17 (fluidez, tempo real, mobile, segurança, acessibilidade, ética OAB) e `10-seguranca-lgpd.md`.

---

## 5. Planos e custos

### 5.1 O que conta como "lead"
- **Lead = contato que recebeu a primeira resposta da IA** e não foi contado nos últimos 30 dias (`09-planos-cobranca.md`).

### 5.2 Planos (mensal)
| Plano | Leads novos/mês | Usuários | WhatsApp | Preço sugerido 💡 |
|---|---|---|---|---|
| Essencial | até 200 | 3 | 1 | R$ 397 |
| Profissional | até 500 | 6 | 2 | R$ 797 |
| Escritório | até 1.000 | 10 | 3 | R$ 1.397 |
- ✅ Estrutura: 3 planos por faixa de leads. ✅ Implantação: **R$ 1.500**.
- 💡 Excedente: ~R$ 1,50 por lead (ou pacote +100 leads).

### 5.3 Custo variável por lead (estimativa)
- **Texto (IA):** modelo econômico (ex.: Claude Haiku 5.5, US$ 0,10 / US$ 0,50 por milhão de tokens) → centavos por atendimento; modelo premium pode passar de R$ 3–5 por lead. Com a base no prompt e cache (D33), o custo da base cai bastante.
- **Voz (ElevenLabs):** ~US$ 0,05 a ~US$ 0,10 por 1.000 caracteres → limitar por lead.
- **Medir o custo real** nos 2–3 primeiros escritórios antes de fechar a tabela.

### 5.4 Cobrança automática (futuro)
- Comparar **Stripe** (cartão 3,99% + R$ 0,39 + 0,7% Billing; Pix só por convite) e **Asaas** (cartão 2,99% + R$ 0,49; Pix/boleto R$ 1,99; sem mensalidade).

### 5.5 Infraestrutura (fase inicial) 💡 (detalhes em `07-infra-deploy.md`)
| Peça | Onde | Custo |
|---|---|---|
| Evolution API + Worker + Redis + App Next.js | VPS **Hostinger KVM 2** (Brasil) + **Coolify** | ~R$ 35–45/mês (promoção, contrato longo) |
| Banco, login, tempo real | **Supabase Free** | R$ 0 |
| Mídias, contratos, backup diário | **Cloudflare R2** (10 GB grátis) | R$ 0 no início |
| DNS + domínio | Cloudflare + registro.br | ~R$ 40/ano |
- Vercel Hobby não permite uso comercial → app na VPS.
- Supabase Free sem backup automático → `pg_dump` diário para o R2.
- Upgrade com 2–3 pagantes: Supabase Pro (US$ 25/mês), VPS maior.

---

## 6. Riscos e restrições
- ⚠️ API não oficial: risco de banimento.
- ⚠️ Evolution API 2.4: ativação obrigatória — dependência externa.
- ⚠️ Política Meta (desde 15/01/2026): proíbe IA de propósito geral na API oficial; atendimento do próprio negócio é permitido.
- ⚠️ Instagram/Messenger multi-cliente: app Meta verificado + revisão.
- ⚠️ Custo de IA embutido: sem medição e limite por escritório, cliente de alto volume pode dar prejuízo.
- ⚠️ Calculadoras: regras e índices mudam; erro de cálculo tem impacto jurídico → parâmetros versionados, testes com casos reais, aviso de estimativa.
- ⚠️ Google Agenda: app precisa de verificação do Google para escopos sensíveis.
- ⚠️ LGPD: dados de saúde → `10-seguranca-lgpd.md`.

---

## 7. Pendências
Ver `docs/PENDENCIAS.md`.

---

## 8. Estrutura da documentação ✅

```
CLAUDE.md                     → regras permanentes do Claude Code (stack, convenções, fluxo de trabalho)
docs/00-visao-geral.md        → este documento
docs/01-requisitos.md         → requisitos por módulo, critérios de aceite, papéis e permissões
docs/02-arquitetura.md        → componentes, fluxos, filas, realtime, variáveis de ambiente
docs/03-banco-de-dados.md     → schema SQL, RLS, índices, funil padrão
docs/04-agente-ia.md          → motor dos agentes (3 camadas, ferramentas, áudio, memória, guardas, testes)
docs/05-integracoes.md        → Evolution, ZapSign, ElevenLabs, transcrição, IA, Google, Push, R2, Meta
docs/06-design-ux.md          → marca Kora IA, tokens, tipografia, componentes, telas
docs/07-infra-deploy.md       → ambiente local, VPS/Coolify, backups, CI/CD, monitoramento
docs/08-calculadoras.md       → escopo, regras e parâmetros das calculadoras
docs/09-planos-cobranca.md    → planos, limites, contador de leads, consumo de IA, painel master
docs/10-seguranca-lgpd.md     → autenticação, autorização, cifragem, LGPD, retenção
docs/templates/<produto>/     → roteiro.md + base-conhecimento.md + config.json
docs/ROADMAP.md               → fases e tarefas com [ ] / [x]
docs/PROGRESSO.md             → diário de sessões
docs/DECISOES.md              → registro de decisões
docs/PENDENCIAS.md            → o que falta definir
```

---

## 9. Roadmap
Detalhado em `ROADMAP.md`. Resumo:
- **F0 — Base:** monorepo, banco multi-tenant, login, design system, painel master, PWA/push, deploy.
- **F1 — Atendimento:** canais WhatsApp, inbox, assumir/transferir, Kanban, etiquetas/tarefas, chat interno, contador de leads.
- **F2 — Agente de IA:** templates e agentes, motor da conversa, áudio/imagem/documento, handoff, memória, playground.
- **F3 — Fechamento:** ZapSign, Google Agenda, follow-up, receita prevista. → **Marco "Piloto"**.
- **F4 — Gestão:** dashboard + calculadoras fase 1.
- **F5 — Depois:** cobrança automática, API oficial, Instagram/Messenger, Meta Ads, novos templates, calculadoras fase 2/3.
