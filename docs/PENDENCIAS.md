# Pendências e perguntas em aberto

> Lista viva do que falta definir. Quando algo for decidido, sai daqui e entra em `DECISOES.md`.
> **Bloqueia?** indica a primeira tarefa do `ROADMAP.md` que depende do item.
> Última atualização: 20/11/2026

## Com o dono do produto

| # | Pendência | Bloqueia? |
|---|---|---|
| 1 | **Confirmar as propostas D27 a D37** (áudios por etapa, guia da gestante, cores, fontes, stack, base no prompt, R2, convenções, "fio da IA", funil padrão). | — (o Claude Code segue com elas) |
| 2 | **Domínio** do Kora IA (ex.: `koraia.com.br`) e e-mail do remetente. | F0.7 |
| 3 | **Contas necessárias** (criadas pelo dono, no nome da empresa): GitHub, Supabase, Cloudflare (R2 + DNS), Hostinger (VPS), provedores de IA, ElevenLabs, ZapSign (sandbox do escritório piloto), Google Cloud (OAuth da Agenda), serviço de e-mail (Resend ou Brevo). | F0.7 / F2 / F3 |
| 4 | **Infraestrutura:** confirmar VPS Hostinger + Coolify e Evolution API (D09, D10). | F0.7 |
| 5 | **Ícone do app** (PWA): aprovar a proposta do `06-design-ux.md`, seção 13, ou enviar o oficial. | F0.6 |
| 6 | **Parcelas futuras dos honorários** (BPC, auxílio-acidente e outros): quantas e qual percentual. | F3.4 |
| 7 | **Roteiro do salário-maternidade v0.2:** revisar o texto final e os ajustes feitos no script original. | F2.1 |
| 8 | **Áudios gravados:** quais etapas terão áudio gravado; cada escritório grava os seus? | F2.3 |
| 9 | **Dados do escritório piloto:** `{{instagram}}`, chamada de vídeo (sim/horários), como é feito o acesso ao Meu INSS (a IA nunca pede senha), quem orienta a guia da gestante. | F2.1 |
| 10 | **Horário de atendimento da IA** (24h ou comercial?) e regras de transferência. | F2.4 |
| 11 | **Valores dos planos** e do excedente: validar com os 2–3 primeiros escritórios. | piloto |
| 12 | **Limite de leads atingido:** IA continua e cobra excedente (proposta) ou pausa para leads novos? | F1.8 |
| 13 | Próximos templates: **BPC/LOAS** e **auxílio-acidente** (mesmo formato do salário-maternidade). | F5 |

## Com advogado previdenciarista
**Base de conhecimento do salário-maternidade** (itens ⚠️ em `templates/salario-maternidade/base-conhecimento.md`):
- 13º (abono anual) no salário-maternidade pago pelo INSS — fontes divergem.
- Desconto de contribuição/IR sobre o benefício.
- Critério de semanas entre aborto (2 semanas) e natimorto (120 dias).
- Acúmulo com BPC, pensão por morte; aposentada que contribui.
- Teto no segundo benefício (CLT no teto + MEI).
- Data exata do fim do período de graça.
- **Gestante que vai contribuir (seção 2.5):** guia certa por situação; validação da baixa renda (CadÚnico); estratégia quando o parto está a mais de 6 meses; risco de negativa com primeira contribuição pouco antes do parto.

**Calculadoras** (ver `08-calculadoras.md`, seção 8):
- Média do contribuinte individual/MEI no salário-maternidade.
- Índice dos benefícios assistenciais (BPC) a partir de set/2025 no Manual CJF 2026.
- Critério do Decreto 12.534/2025 para a renda do BPC.
- Casos reais anonimizados para testes.

## Com advogado de proteção de dados
- Termos de uso, acordo de tratamento de dados (DPA) e política de privacidade do Kora IA (`10-seguranca-lgpd.md`, seção 5).
- Como avisar o lead sobre privacidade sem quebrar o roteiro (link na primeira mensagem? perfil do WhatsApp?).
- Prazo de retenção de leads que não contrataram (proposta: 24 meses).

## Técnicas (decidir durante a construção)
| Pendência | Quando |
|---|---|
| Modelo de IA padrão por função (triagem/memória, conversa, visão): testar 2–3 com os cenários de `04-agente-ia.md`, seção 11 | F2.2 |
| Provedor de transcrição de áudio (testar com áudios reais de WhatsApp) | F2.3 |
| Versão estável da Evolution API sem ativação | F1.1 |
| Verificação do app no Google (escopos da Agenda) — iniciar cedo | F3.2 |
| Bloqueio automático por atraso no pagamento (ligado ou desligado por padrão) | F0.5 |
