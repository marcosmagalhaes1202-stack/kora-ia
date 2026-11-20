# Requisitos — Kora IA (v1.0)

> Requisitos funcionais por módulo, com critérios de aceite. Visão e contexto: `00-visao-geral.md`. Ordem de construção: `ROADMAP.md`.
> **Convenção:** RF = requisito funcional · RNF = não funcional · ✅ decidido · 💡 proposta.

---

## 1. Papéis e permissões

| Papel | Quem é | Resumo |
|---|---|---|
| **master** | Dono do SaaS (Kora IA) | Tudo em todos os escritórios: planos, status, pagamentos, templates, camada Global, parâmetros legais, suporte. |
| **admin** | Dono/sócio do escritório | Tudo no próprio escritório: equipe, canais, agentes, integrações, funil, financeiro. |
| **advogado** | Advogado da equipe | Atende, vê todos os cards, contratos, calculadoras e receita. Não mexe em equipe nem integrações. |
| **comercial** | Atendente/SDR | Atende, move cards, envia contrato. Vê só a receita dos próprios cards. |
| **atendimento** | 2º atendimento / pós-venda | Igual ao comercial, com foco na visão Pós-venda. |
| **financeiro** | Financeiro | Receita e contratos; lê conversas, mas não atende. |

| Ação | admin | advogado | comercial | atendimento | financeiro |
|---|:-:|:-:|:-:|:-:|:-:|
| Ver e responder conversas | ✓ | ✓ | ✓ | ✓ | só ver |
| Assumir, devolver e transferir | ✓ | ✓ | ✓ | ✓ | — |
| Mover cards e editar campos | ✓ | ✓ | ✓ | ✓ | — |
| Editar etapas do funil | ✓ | — | — | — | — |
| Enviar contrato | ✓ | ✓ | ✓ | ✓ | — |
| Ver e editar receita | ✓ | ✓ | só os seus | só os seus | ✓ |
| Calculadoras | ✓ | ✓ | ✓ | ✓ | ✓ |
| Agentes (configurar) | ✓ | — | — | — | — |
| Canais e integrações | ✓ | — | — | — | — |
| Equipe e setores | ✓ | — | — | — | — |
| Dashboard | ✓ | ✓ | só os seus | só os seus | ✓ |

- 💡 MVP: cada usuário pertence a **um** escritório (o master não pertence a nenhum).
- O master pode "entrar como" um escritório para suporte. Isso é registrado em `auditoria` e mostra uma faixa visível na tela.

---

## 2. Acesso e conta

- **RF-01** Login com e-mail e senha; recuperação de senha por e-mail.
- **RF-02** O admin convida membros por e-mail, definindo papel e setor. O convite expira em 7 dias.
- **RF-03** Escritório com status `bloqueado` → os usuários veem a tela "Acesso suspenso" com o contato do suporte. A IA para de responder, mas o sistema **continua recebendo** as mensagens.
- **RF-04** Status `teste` mostra uma faixa com os dias restantes.
- **Aceite:** um usuário bloqueado não acessa nenhuma rota de dados (testado também via API do Supabase).

## 3. Canais
- **RF-10** Conectar número de WhatsApp por QR code (Evolution); ver status (conectado, desconectado, aguardando QR, erro) em tempo real.
- **RF-11** Limite de números por plano (padrão: 1 / 2 / 3).
- **RF-12** Cada canal tem nome, cor, agente padrão (ou triagem) e limites de envio.
- **RF-13** Ao desconectar, notificar o admin (push + sino).
- 💡 F5: API oficial, Instagram e Messenger (mesma interface de canal).

## 4. Inbox
- **RF-20** Lista única de conversas de todos os canais, ordenada pela última mensagem; filtros por canal, número, status, modo (IA/humano), responsável, setor, etiqueta, produto e "sem resposta há mais de X min". Visões salvas por usuário.
- **RF-21** Busca por nome, telefone e texto da mensagem.
- **RF-22** Enviar e receber texto, áudio (gravar no navegador), imagem, vídeo, documento; ver status (enviada, entregue, lida).
- **RF-23** Notas internas (não vão para o cliente), com @menção.
- **RF-24** Respostas rápidas digitando `/`.
- **RF-25** Selo visível: **IA atendendo** / **Humano: Nome** / **Pausada**.
- **RF-26** Painel lateral: dados do contato, card, resumo da IA, documentos, etiquetas, reuniões, contratos e linha do tempo.
- **RF-27** Não lidas por conversa e total no menu; marcar como não lida.
- **RF-28** Mensagens enviadas pelo celular do escritório (fora do sistema) aparecem na conversa como "enviada pelo celular".
- **Aceite:** mensagem recebida aparece em menos de 2 s na tela aberta; nenhuma mensagem duplicada após reenvio de webhook.

## 5. Atendimento humano × IA
- **RF-30** "Assumir" pausa a IA na conversa. Enviar mensagem como humano também assume.
- **RF-31** "Devolver para a IA": a IA retoma a partir do estado do roteiro salvo no card.
- **RF-32** Transferir para membro ou setor com motivo; a IA também transfere sozinha nos casos 🔴 do template, sempre com resumo.
- **RF-33** 💡 Pausa automática: se o humano não responder em X min após a transferência, notificar de novo (alerta de SLA).

## 6. Funil (Kanban)
- **RF-40** Um quadro por escritório, com etapas editáveis (nome, cor, ordem, tipo, se a IA pode mover).
- **RF-41** A IA cria o card no primeiro contato e move até "2º atendimento". Depois, `ia_controla = false` e só a equipe move.
- **RF-42** Arrastar e soltar; soma de valor previsto e contagem por coluna; tempo na etapa.
- **RF-43** Visões "Comercial" e "Pós-venda"; modo lista; filtros iguais aos do Inbox; arquivar.
- **RF-44** Detalhe do card: campos coletados, checklist de documentos, contratos, reuniões, tarefas, cálculos, receita e linha do tempo (quem fez o quê: IA, membro, sistema).
- **RF-45** Motivo de perda obrigatório ao mover para "Perdido / Sem direito".
- **Aceite:** um mesmo contato pode ter mais de um card (ex.: salário-maternidade e BPC).

## 7. Etiquetas, tarefas e organização
- **RF-50** Etiquetas de contato e de card, cores pastel, manuais e automáticas (aplicadas pela IA).
- **RF-51** Tarefas por card com responsável e prazo; "Minhas tarefas" com atrasadas em destaque.
- **RF-52** Visões salvas: "Meus atendimentos", "Contratos aguardando assinatura", "Sem resposta > 2h", "Documentos pendentes", "Reuniões de hoje".

## 8. Agentes de IA (detalhes em `04-agente-ia.md`)
- **RF-60** Central de agentes: ativar produtos a partir dos templates; personalizar nome do atendente, variáveis do escritório, horários, modelo, voz e regras de áudio.
- **RF-61** Subir **áudios gravados** por etapa do roteiro.
- **RF-62** Base de conhecimento extra do escritório (texto livre).
- **RF-63** Pré-visualizar o prompt final e testar no playground.
- **RF-64** A IA responde em mensagens curtas, **uma pergunta por vez**, entende áudio, imagem e documento, e responde em áudio conforme a regra.
- **RF-65** A IA preenche campos do card, aplica etiquetas, move etapas, pede e valida documentos, calcula estimativas, agenda reunião, gera contrato e transfere.
- **RF-66** Memória do contato: ao voltar, a IA retoma do ponto em que parou.
- **Aceite:** suíte de cenários do salário-maternidade passa (ver `04-agente-ia.md`, seção 11).

## 9. Follow-up
- **RF-70** Sequências com gatilho, passos (atraso + tipo de mensagem), janela de horário e regras de parada.
- **RF-71** Gatilhos: sem resposta, contrato pendente, documentos pendentes, data relativa a um campo (ex.: DPP), reativação de base.
- **RF-72** Para quando: o cliente responde, assina, pede para parar, é transferido para humano ou o card é perdido.
- **RF-73** Limite diário de envios por número (anti-banimento).

## 10. Contratos (ZapSign)
- **RF-80** Modelos por produto com mapeamento de variáveis → campos do card; envio pela IA ou manual.
- **RF-81** Status em tempo real (gerado, enviado, visualizado, assinado, recusado); PDF assinado salvo no R2.
- **RF-82** Ao assinar: boas-vindas, card → "Contrato assinado" → "2º atendimento", notificação, receita prevista criada.

## 11. Agenda
- **RF-90** Reunião com data, hora, tipo (vídeo, ligação, presencial) e responsável; evento no Google Agenda do responsável.
- **RF-91** Lembretes para o cliente (WhatsApp) e para a equipe (push).

## 12. Comunicação interna e notificações
- **RF-100** Chat interno: direto, por setor e geral; @menção.
- **RF-101** Notificações (sino + push): nova conversa atribuída, transferência, menção, contrato assinado, reunião próxima, canal desconectado, limite de leads (80% / 100%).
- **RF-102** Preferências de notificação por usuário.

## 13. Financeiro
- **RF-110** Receita prevista por card: atrasados (único) e parcelas mensais futuras, conforme `regras_honorarios`; salário-maternidade: 30% do total (D23).
- **RF-111** Editar valor real; marcar como recebida; filtro por mês e produto.

## 14. Dashboard
- **RF-120** Leads novos, conversas, contratos gerados e assinados (por produto, canal, período), conversão por etapa, tempo de primeira resposta, motivos de perda, receita prevista × realizada, consumo de IA.

## 15. Calculadoras
Ver `08-calculadoras.md` (fase 1 na F4 do roadmap).

## 16. Painel master (detalhes em `09-planos-cobranca.md`)
- **RF-130** Planos, escritórios, status, "pago até", limites personalizados, pagamentos manuais, contador de leads, consumo de IA por escritório.
- **RF-131** Templates e camada Global com versões.
- **RF-132** Parâmetros legais e índices.
- **RF-133** "Entrar como" escritório, com auditoria.

---

## 17. Requisitos não funcionais

| RNF | Meta |
|---|---|
| **Fluidez** | Inbox interativo em < 1,5 s (4G); trocar de conversa < 300 ms com cache; listas virtualizadas. |
| **Tempo real** | Mensagem recebida na tela em < 2 s. |
| **Resposta da IA** | Primeira mensagem em até ~15 s depois que o cliente para de digitar (debounce + geração). |
| **Disponibilidade** | Webhooks nunca perdem mensagem: gravar antes de processar; reprocessar em caso de falha. |
| **Escala inicial** | 100 usuários simultâneos, ~20 escritórios, ~30 números, sem degradar. |
| **Mobile** | PWA instalável; Inbox e Kanban completos no celular; push. |
| **Segurança** | RLS em todas as tabelas; segredos cifrados; auditoria; ver `10-seguranca-lgpd.md`. |
| **Acessibilidade** | Contraste AA, foco visível, navegação por teclado no desktop, toque ≥ 44 px. |
| **Ética OAB** | Sem promessa de resultado; valores como estimativa; publicidade sóbria. |
