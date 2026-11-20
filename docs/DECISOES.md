# Registro de decisões

> Formato: cada decisão tem número, data, o que foi decidido, por quê e status.
> **Status:** ✅ decidido · 💡 proposto (aguardando confirmação; o Claude Code segue com ele até o dono dizer o contrário) · ❌ descartado · 🔁 substituído
> Pendências abertas: `docs/PENDENCIAS.md`

| # | Data | Decisão | Motivo | Status |
|---|---|---|---|---|
| D01 | 19/11/2026 | Agentes em **3 camadas**: (1) Global — comportamento e forma de responder, só o dono do SaaS edita; (2) Template do produto — roteiro, critérios, documentos, campos do contrato, valor previsto; (3) Personalização do escritório — nome, OAB, voz, horários, honorários. Prompt montado na hora de responder. | Qualidade padronizada; melhoria em um lugar chega a todos; onboarding rápido; templates viram ativo do negócio. | ✅ |
| D02 | 19/11/2026 | **Custo de IA embutido no preço do plano**; credenciais dos provedores ficam com o dono do SaaS. | Simplicidade para o advogado. Exige medir consumo por escritório. | ✅ |
| D03 | 19/11/2026 | **Roteamento:** mensagem pré-preenchida do anúncio ativa o agente do produto; sem ela, triagem identifica o interesse. | Anúncios já chegam com mensagem pronta por produto. | ✅ |
| D04 | 19/11/2026 | **Contrato:** a IA coleta os dados na conversa, preenche o modelo dinâmico da ZapSign e envia o link. | Atendimento ponta a ponta. | ✅ |
| D05 | 19/11/2026 | **"Agendamento" = reunião marcada**, com data no card e Google Agenda para alerta. | Organização e lembrete. | ✅ |
| D06 | 19/11/2026 | **Após o 2º atendimento, a IA para de acompanhar**; equipe move cards e anexa informações. | Pós-venda é humano. | ✅ |
| D07 | 19/11/2026 | **Calculadoras jurídicas** em seção própria, completas; saída em PDF padrão e/ou resumo por WhatsApp. | Ferramenta do dia a dia dentro do sistema. | ✅ |
| D08 | 19/11/2026 | **Integração com tráfego** (Meta Ads + Conversions API) fica para depois, módulo opcional. | Exige API oficial. | ✅ |
| D09 | 19/11/2026 | **WhatsApp não oficial: Evolution API self-hosted**, atrás de camada de "provedor de canal". | Gratuita, sem limite de instâncias, já conhecida. Risco: versão 2.4 exige ativação. | 💡 |
| D10 | 19/11/2026 | **Infra inicial:** Hostinger KVM 2 (Brasil) + Coolify; Supabase Free; Cloudflare R2 para mídias e backup diário; upgrade com 2–3 pagantes. | Custo mínimo; Vercel Hobby não permite uso comercial; Supabase Free sem backup. | 💡 |
| D11 | 19/11/2026 | **Kanban em quadro único.** A IA cria e move cards até "contrato assinado → 2º atendimento"; depois nunca mais move o card. Visões salvas "Comercial" e "Pós-venda" + arquivamento. | Simplicidade; visões evitam bagunça com volume. | ✅ |
| D12 | 19/11/2026 | **Documentos pedidos antes do contrato**, porque alimentam o preenchimento. 💡 IA lê RG/CNH e comprovante e pede só confirmação. | O contrato depende desses dados. | ✅ |
| D13 | 19/11/2026 | **Receita prevista:** valor padrão por produto (ex.: salário-maternidade ≈ R$ 2.000; BPC ≈ R$ 10.000); IA preenche ao assinar; equipe edita o valor real. Calculadora de atrasados + regra de honorários (D23) refina a projeção. | Noção de quanto deve entrar, ajustável. | ✅ |
| D14 | 19/11/2026 | ~~Cobrança automática + liberação manual.~~ Substituída por D19. | — | 🔁 |
| D15 | 19/11/2026 | ~~Gateway Stripe × Asaas.~~ Adiado: cobrança automática fica para depois (ver D19). | Início com cobrança manual. | 🔁 |
| D16 | 19/11/2026 | **3 planos limitados por leads novos atendidos por mês** (faixas 200 / 500 / 1.000). Valores sugeridos: R$ 397 / 797 / 1.397 + excedente. | IA consome tokens por atendimento; limite por lead é fácil de entender. Valores a validar no piloto. | ✅ estrutura · 💡 valores |
| D17 | 19/11/2026 | **Previdenciário primeiro** (agentes, templates e calculadoras); templates iniciais: salário-maternidade, BPC, auxílio-acidente. | Área em que o dono mais atua. | ✅ |
| D18 | 19/11/2026 | **Identidade visual:** branco dominante, uma cor de destaque, pouco colorido, elementos arredondados. Cor detalhada em D30. | Sistema claro e moderno. | ✅ |
| D19 | 19/11/2026 | **Cobrança manual no início:** o dono gera a cobrança fora do sistema e **libera/bloqueia o acesso manualmente** no painel master, que permite definir plano e limites de cada escritório, registrar pagamentos e controlar "pago até". Cobrança automática depois. | Começar simples e com controle total. | ✅ |
| D20 | 19/11/2026 | **Implantação: R$ 1.500** (pagamento único). | Configuração individual de agentes, voz, contrato e treinamento. | ✅ |
| D21 | 19/11/2026 | **Base dos agentes:** roteiros de atendimento do dono + pesquisa de informações atualizadas. Primeiro roteiro recebido e mesclado: salário-maternidade (v0.2). | Agentes precisos e alinhados à prática real. | ✅ |
| D22 | 19/11/2026 | **Calculadoras em 3 fases:** (1) atendimento; (2) processo; (3) avaliar planejamento. Ver `08-calculadoras.md`. | Foco no que ajuda a vender e decidir o caso. | ✅ |
| D23 | 19/11/2026 | **Honorários padrão:** **30% dos atrasados** + **parcelas futuras cobradas mensalmente**. Configurável por escritório e produto. **Salário-maternidade:** 30% do **valor total** do benefício, pago só depois que o benefício cai na conta. | Prática do dono. Parcelas futuras de outros produtos: ver `PENDENCIAS.md`. | ✅ |
| D24 | 19/11/2026 | **A IA pode informar ao cliente o valor estimado do benefício**, sempre como estimativa, usando o motor das calculadoras. | Ajuda a converter; o motor evita valores inventados. | ✅ |
| D25 | 19/11/2026 | **Honorários sempre em linguagem simples:** "Você não paga nada agora. Só paga o escritório depois que começar a receber o benefício: {{percentual}} do valor total", mostrando a conta. Nunca "êxito". | Pedido do dono. | ✅ |
| D26 | 19/11/2026 | **Uma pergunta por vez** (camada Global); no máximo 2–3 mensagens curtas seguidas; documentos pedidos um por vez. | Pedido do dono: conversa natural e empática. | ✅ |
| D27 | 19/11/2026 | **Áudio por etapa do roteiro:** `[ÁUDIO GRAVADO]` (arquivo do escritório), `[ÁUDIO]` (voz configurada) ou `[TEXTO]`. Boas-vindas do anúncio = áudio gravado. | O script do dono usa áudios em pontos-chave. | 💡 |
| D28 | 19/11/2026 | **Gestante que ainda não contribui:** a IA explica que dá para garantir o direito contribuindo **antes do parto**; quem orienta a guia é o escritório. | Evita orientação errada de guia e promessa indevida. | 💡 |
| D29 | 20/11/2026 | **Nome do produto: Kora IA — CRM Jurídico.** Logo: "Kora" em azul-tinta, "IA" em verde. | Definido pelo dono. | ✅ |
| D30 | 20/11/2026 | **Cores:** tinta #2C4058 (principal: botões, menu ativo, títulos), folha #6AD990 (**exclusiva da IA**), grafite #5C646C; fundo névoa #F4F6F8 com painéis brancos. | Tiradas do logo; dá significado ao verde (D36). | 💡 |
| D31 | 20/11/2026 | **Tipografia:** Inter (toda a interface e números) + Montserrat (títulos, números grandes, marca). Lato fora. | Inter é a mais legível em telas pequenas; Montserrat conversa com o logo. | 💡 |
| D32 | 20/11/2026 | **Stack de construção:** monorepo pnpm + Turborepo; Next.js + Tailwind v4 + shadcn/ui; worker Node + Fastify + BullMQ/Redis; Supabase; Cloudflare R2; Vercel AI SDK; Biome; Vitest. Ver `CLAUDE.md`. | Custo baixo, tempo real, multi-provedor de IA, fácil de manter com o Claude Code. | 💡 adotada para começar |
| D33 | 20/11/2026 | **Base de conhecimento inteira no prompt, com cache**; sem busca vetorial (RAG) no início. | Bases de ~12 mil tokens: mais precisão e simplicidade; cache reduz o custo. | 💡 |
| D34 | 20/11/2026 | **Arquivos no R2** (não no Supabase Storage); **Evolution só na rede interna** da VPS. | Storage grátis do Supabase é pequeno; Evolution não precisa ficar exposta. | 💡 |
| D35 | 20/11/2026 | **Convenções:** banco em português sem acento; entidades de domínio em português no código; docs e commits em pt-BR; o Claude Code trabalha por `CLAUDE.md` → `ROADMAP.md` → `PROGRESSO.md`. | Mesmo vocabulário do dono, dos docs e do banco. | 💡 |
| D36 | 20/11/2026 | **"Fio da IA":** tudo que a IA controla ou produz ganha fio verde lateral, selo "Nome (IA)" e, enquanto gera resposta, um ponto pulsando. | O advogado vê na hora o que foi a IA e o que foi a equipe. | 💡 |
| D37 | 20/11/2026 | **Funil padrão com etapas de sistema identificadas por chave** (novo, qualificacao, documentos, reuniao, contrato_enviado, contrato_assinado, segundo_atendimento, followup, perdido) + etapas livres de pós-venda. A IA usa a chave, não o nome. | O escritório pode renomear sem quebrar a IA. | 💡 |
