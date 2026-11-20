# Planos, limites, consumo e painel master — Kora IA (v1.0)

> Decisões: D02, D16, D19, D20. Valores dos planos ainda 💡 (validar no piloto).

---

## 1. Planos

| Plano | Leads novos/mês | Usuários | Números de WhatsApp | Preço de referência 💡 |
|---|---|---|---|---|
| Essencial | 200 | 3 | 1 | R$ 397 |
| Profissional | 500 | 6 | 2 | R$ 797 |
| Escritório | 1.000 | 10 | 3 | R$ 1.397 |

- **Implantação:** R$ 1.500, pagamento único (D20).
- `escritorios.limites_personalizados` sobrepõe qualquer limite do plano (negociação, cortesia).
- Limite efetivo = `coalesce(limites_personalizados.x, plano.x)` — calcular numa função única (`app.limites_do_escritorio(id)`), usada pela web e pelo worker.

## 2. O que é um lead
- **Lead = contato que recebeu a primeira resposta da IA** e que não foi contado nos últimos 30 dias (função `app.contabilizar_lead`, `03-banco-de-dados.md`).
- Conversas só com humanos **não** contam.
- O mesmo contato voltando depois de 30 dias conta de novo.
- O contador do mês fica em `uso_mensal.leads` (competência no fuso de São Paulo).

## 3. O que acontece em cada limite

| Limite | Comportamento |
|---|---|
| Usuários | Convite bloqueado ao atingir o limite, com aviso "Seu plano permite X usuários". |
| Números | Botão "Conectar WhatsApp" bloqueado ao atingir o limite. |
| Leads — 80% | Notificação para o admin e para o master. |
| Leads — 100% | 💡 **A IA continua atendendo** e os leads acima do limite ficam registrados como excedente (cobrado à parte, ~R$ 1,50/lead 💡). Notificação para admin e master. O master pode mudar para "pausar a IA para leads novos" por escritório. |
| Mensagens da IA por lead | `agentes.limite_mensagens_por_lead` (padrão 80). Passou → transfere para humano. |
| Voz por lead | 💡 até 3.000 caracteres de áudio gerado por lead/mês; depois, só texto. |

## 4. Consumo de IA
- Cada chamada (texto, visão, transcrição, voz) grava `consumo_ia` com tokens/caracteres/segundos e **custo estimado em US$** (tabela `packages/ai/precos.ts`).
- `uso_mensal` soma tudo por escritório e mês.
- Painel master mostra por escritório: leads, mensagens da IA, custo estimado do mês, **custo por lead** e **margem** (preço do plano − custo). Serve para validar os preços no piloto.

## 5. Status do escritório

```
 teste ──► ativo ◄──► bloqueado
   │         │
   └──► cortesia      cancelado (dados mantidos por 90 dias, depois apagados — ver 10-seguranca-lgpd)
```
- **teste:** com `teste_ate`; faixa com dias restantes; ao vencer, vira `bloqueado` automaticamente.
- **ativo:** normal. O "pago até" é informativo.
- 💡 **Bloqueio automático por atraso:** desligado por padrão; se ligado, bloqueia X dias após o "pago até" (configurável no master).
- **bloqueado:** usuários veem "Acesso suspenso"; a IA e os follow-ups param; mensagens continuam chegando e ficam guardadas.
- Lembretes para o master: "pago até" vencendo em 5 dias e vencido.

## 6. Painel master — telas
- **Escritórios:** tabela com nome, plano, status, "pago até" (em vermelho se vencido), leads do mês / limite (barra), custo de IA do mês, números conectados, último acesso. Filtros por status e plano.
- **Detalhe do escritório:** dados, plano e limites personalizados, status (bloquear/liberar com motivo), "pago até", pagamentos, uso mês a mês, canais, membros, observações, **Entrar como** (auditado).
- **Planos:** criar e editar.
- **Pagamentos:** registrar (data, valor, forma, referência, competência) e, opcionalmente, avançar o "pago até" em 1 mês. Histórico com filtro por mês (receita do SaaS).
- **Templates e camada Global:** editar, ver diferenças entre versões, publicar.
- **Parâmetros legais e índices:** ver `08-calculadoras.md`.
- **Consumo:** visão geral de custo de IA por escritório e por modelo.
- **Filas:** link para o Bull Board.

## 7. Cobrança automática (F5)
- Comparar Asaas (Pix/boleto baratos) e Stripe. Webhook de pagamento confirmado → registra o pagamento e avança o "pago até" sozinho. O fluxo manual continua existindo.
