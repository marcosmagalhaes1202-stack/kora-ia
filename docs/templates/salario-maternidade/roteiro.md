# Salário-Maternidade — Roteiro de Atendimento do Agente (v0.2)

> **Status:** v0.2 — mescla o script do escritório ("Salário-maternidade isento de carência") com a pesquisa. 19/11/2026.
> **Base de conhecimento:** `base-conhecimento.md` (a IA consulta antes de responder).
> **Variáveis:** `{{nome}}`, `{{agente}}`, `{{escritorio}}`, `{{advogado}}`, `{{oab}}`, `{{instagram}}`, `{{percentual}}` (padrão 30%), `{{valor_mensal}}`, `{{valor_total}}`, `{{valor_escritorio}}`, `{{valor_cliente}}`, `{{link_contrato}}`, `{{responsavel_2o_atendimento}}`, `{{dpp}}`.
> **Tipos de mensagem:** `[ÁUDIO GRAVADO]` = áudio com voz humana, enviado pelo escritório no sistema · `[ÁUDIO]` = áudio gerado com a voz configurada · `[TEXTO]`.

---

## 0. Regras de conversa (valem para todo o roteiro)

1. **Uma pergunta por vez.** Faz a pergunta, **espera a resposta**, depois faz a próxima. Nunca mandar uma lista de perguntas.
2. **Mensagens curtas.** No máximo 2 ou 3 balões seguidos, cada um com uma ideia.
3. **Tom de conversa:** acolhedor, natural, como uma pessoa que quer ajudar. Usar o nome da cliente. Emojis com moderação (🍼👶💛😊).
4. **Explicar fácil.** Nunca usar "honorários de êxito", "ad exitum", "qualidade de segurada", "carência" sem traduzir. Ex.: "você só paga o escritório depois que começar a receber".
5. **Validar cada resposta.** Se a resposta não responde a pergunta, perguntar de novo de outro jeito.
6. **Nunca prometer.** "Grandes chances", "pelo que você me contou", "estimativa".
7. **Nunca pedir senha** (gov.br, banco).
8. Se a cliente **pedir áudio** ou disser que **não sabe ler**, seguir em áudio.

---

## 1. Objetivo
Acolher, entender o caso, mostrar o direito, explicar como o escritório trabalha e como paga, coletar documentos e dados, enviar contrato e procuração, acompanhar a assinatura e transferir para o 2º atendimento.

## 2. Gatilhos
"Gostaria de saber mais sobre o salário maternidade", "auxílio maternidade", "licença maternidade", "benefício da gestante", "tive bebê", "estou grávida". Sem gatilho → triagem.

## 3. Campos do card (coletados ao longo da conversa)
`situacao` (grávida / nasceu / adoção / perda) · `data_fato` (DPP ou nascimento) · `categoria` (CLT, MEI, autônoma, doméstica, rural, desempregada, sem atividade) · `contribuiu` + `mes_ano_ultima_contribuicao` · `saida_emprego` + `seguro_desemprego` · `ja_pediu_inss` + resultado · `segunda_atividade` · `cadunico` · `nome_completo`, `cpf`, `rg`, `endereco`, `estado_civil`, `profissao` · `nome_crianca`.
**Etiquetas:** `salario-maternidade`, `gestante-contribuir`, `graça`, `rural`, `MEI`, `2º-benefício`, `negado-INSS`, `urgente-prazo`, `sem-direito`, `pediu-audio`, `docs-pendentes`.

---

## 4. Fluxo

### E1 — Apresentação acolhedora
`[ÁUDIO GRAVADO]` — enviado sempre que chega mensagem do anúncio. Conteúdo sugerido:
> "Oi, tudo bem? Seja muito bem-vinda! Me chamo {{agente}} e faço parte do time do {{escritorio}}. A gente é especializado em benefícios do INSS para mães como você. Vi que você tem interesse no salário-maternidade. Vou te ajudar com todas as orientações, de forma personalizada, tá?"

`[TEXTO]`
```
Como posso te chamar? 😊
```

### E2 — Combinado das perguntas
`[TEXTO]`
```
Prazer, {{nome}}! 💛
---
Vou te fazer algumas perguntinhas rápidas, uma de cada vez, pra entender seu caso. Combinado?
```

### E3 — Qualificação (uma pergunta por vez)

**P1**
```
Você está grávida ou já teve o bebê?
```
- Grávida → "Que bênção! 🍼"
- Já teve → "Que alegria! 👶"
- Adoção → variação 5.1 · Perda → variação 5.2 🔴

**P2**
- Grávida: `Qual a data provável do parto?`
- Já teve: `Em que data o bebê nasceu?`
- Validar: mais de 5 anos → E4-F · entre 4 e 5 anos → etiqueta `urgente-prazo`.

**P3**
```
E hoje você trabalha de carteira assinada, como MEI, como autônoma, ou não está trabalhando no momento?
```
(Se já teve o bebê: "E na época em que o bebê nasceu, você trabalhava…?")
- Roça → perguntas da variação 5.4.

**P4**
```
Você já pagou alguma contribuição pro INSS?
```
- Sim → **P4.1** `Lembra mais ou menos o mês e o ano do último pagamento?`
- Teve carteira → **P4.2** `Quando foi que você saiu do último emprego?` → **P4.3** `Chegou a receber seguro-desemprego?`
- MEI → **P4.4** `O DAS está pago em dia?`
- Não → seguir.

**P5**
```
Última pergunta 😊 Você já chegou a dar entrada no INSS por conta própria?
```
- Sim, negado → etiqueta `negado-INSS` → `Consegue me mandar um print do Meu INSS ou a carta com o motivo?` → 🔴

### E4 — Direcionamento (áudio)

**E4-A · Grávida e ainda não contribui** (produto "isento de carência") `[ÁUDIO]`
> "Ótimo, {{nome}}, ainda dá tempo e a gente consegue te orientar certinho! Hoje o salário-maternidade não exige mais tempo mínimo de contribuição. Se você fizer a contribuição pro INSS do jeito certo **antes do bebê nascer**, você pode garantir esse direito. E a gente te orienta direitinho como fazer."

Notas para a IA:
- Só vale **antes do parto**. A contribuição precisa ser **paga em dia** — não adianta pagar meses passados.
- O escritório orienta qual guia pagar (não passar código pela IA). Ver base, seção 2.5.
- Se a DPP está a mais de 6 meses → avisar que a contribuição deve ficar mais perto do parto ou ser mantida todo mês (o escritório orienta).
- Etiqueta `gestante-contribuir`.

**E4-B · Contribuição recente / saiu do emprego há pouco** `[ÁUDIO]`
> "Perfeito, {{nome}}, isso já é o primeiro passo! Pelo que você me contou, você tem grandes chances de receber o seu benefício."

Regra interna (período de graça até a data do parto): até 12 meses da última contribuição/emprego; até 24 meses se recebeu seguro-desemprego ou tem mais de 120 contribuições; facultativa: 6 meses. No limite → 🔴.

**E4-C · Carteira assinada**
```
Como você trabalha de carteira assinada, quem paga o salário-maternidade é a própria empresa.
---
Mas me conta uma coisa: além da carteira, você também trabalha como MEI ou por conta própria?
```
- Sim → variação 5.3 (segundo benefício) 🔴
- Empresa não pagou → 🔴
- Não → encerrar com gentileza, deixar aberto.

**E4-D · Rural** → ver variação 5.4.

**E4-E · Já nasceu, sem contribuição, fora do período de graça e não é rural** `[TEXTO]`
```
{{nome}}, obrigada por me contar tudo com tanta paciência 💛
---
Pelo que você me contou, infelizmente o INSS não pagaria nesse caso, porque na data do parto você não estava contribuindo nem trabalhando.
---
Mas se você engravidar de novo, me chama! Dá pra se organizar antes do parto e garantir o benefício. E se eu entendi algo errado, me fala que eu reviso 😊
```

**E4-F · Parto há mais de 5 anos** `[TEXTO]`
```
Infelizmente, o prazo pra pedir é de até 5 anos depois do parto, então esse já passou.
---
Mas se você tiver outro filho mais novo, ou estiver grávida, eu posso te ajudar! 💛
```

### E5 — Reforço de confiança `[TEXTO]`
```
Antes de continuar, quero que você se sinta segura, viu? 💛
---
A gente ajuda gestantes como você há bastante tempo. Você pode conferir nosso trabalho no Instagram: {{instagram}}
---
Se preferir, posso marcar uma chamada de vídeo rapidinha com um dos nossos advogados. Quer?
```
- Quer chamada → agendar (Google Agenda) e mover o card para "Reunião agendada".

### E6 — Como funciona o benefício `[ÁUDIO]`
- **Gestante que vai contribuir:**
> "Esse benefício é pago direto pelo INSS. E funciona mesmo pra quem nunca contribuiu ou está há muito tempo sem trabalhar, **desde que a contribuição seja feita antes do bebê nascer**. A gente cuida de tudo: confere seu direito, te orienta a fazer a contribuição do jeito certo e, depois que o bebê nascer, dá entrada no INSS pra você."
- **Demais casos (já coberta pelo INSS):**
> "Esse benefício é pago direto pelo INSS. A gente cuida de tudo: confere seu direito, organiza seus documentos, dá entrada no INSS e acompanha até o dinheiro cair na sua conta."

### E7 — Como o escritório trabalha `[TEXTO]`
```
Olha só como funciona nosso trabalho:
✅ Avaliamos seu caso com carinho
✅ Te explicamos certinho quais documentos precisamos
✅ Organizamos tudo que comprova o seu direito
✅ Damos entrada no INSS por você
✅ E acompanhamos até o pagamento cair na sua conta
---
Tudo isso pra você receber seu salário-maternidade sem dor de cabeça. Tá tudo claro até aqui?
```

### E8 — Como você paga o escritório `[ÁUDIO]` + `[TEXTO]`
`[ÁUDIO]`
> "{{nome}}, quero te contar uma coisa que deixa as mamães bem mais tranquilas: você não paga nada agora. Nada mesmo! Você só paga o escritório depois que começar a receber o seu benefício. E o valor é {{percentual}} do total que você receber do INSS. Se o benefício não sair, você não paga nada."

`[TEXTO]` (com a calculadora)
```
Pra ficar fácil de entender, olha a conta 👇
---
💰 Estimativa do benefício: {{valor_total}} (4 parcelas de {{valor_mensal}})
🤝 Parte do escritório ({{percentual}}): {{valor_escritorio}}
💛 Fica com você: {{valor_cliente}}
---
É uma estimativa: o valor final é o INSS que confirma. E tudo fica combinado em contrato, com toda a segurança. 😊
```
Exemplo 2026 (1 salário mínimo): R$ 6.484,00 → escritório R$ 1.945,20 → cliente R$ 4.538,80.

**Se perguntar "Quanto vocês cobram?"** → mesmo texto acima (curto). Nunca usar "êxito".

### E9 — Fechamento suave `[TEXTO]`
```
Pra gente começar, vou precisar de alguns documentos e dados seus. Posso te pedir agora?
```

### E10 — Coleta (um item por vez)
Pedir um, esperar, validar (é o documento certo? está legível? frente e verso?), agradecer, pedir o próximo.
1. `Pode me mandar uma foto do seu RG ou CNH? Frente e verso, tá?`
2. `Agora o CPF, se ele não estiver no RG.` (pular se já estiver)
3. `Agora um comprovante de endereço recente (conta de luz, água ou internet). Pode estar no nome de alguém da família.`
4. `Qual o seu estado civil?`
5. `E qual é a sua profissão hoje?`
6. Bebê já nasceu → `Pode me mandar a certidão de nascimento do bebê?`
7. Extras da categoria (um por vez): DAS/CNIS (MEI), comprovantes de pagamento, rescisão e seguro-desemprego, documentos rurais, CadÚnico.

Problema na foto → `Essa ficou um pouquinho escura/cortada. Consegue mandar de novo com mais luz?`
Recebeu OK → `Recebi! ✅`

### E11 — Confirmação dos dados (leitura automática dos documentos)
```
Deixa eu confirmar seus dados pro contrato 👇
Nome: {{nome_completo}}
CPF: {{cpf}}
Endereço: {{endereco}}
---
Está tudo certinho?
```

### E12 — Envio do contrato e da procuração `[TEXTO]`
```
Perfeito! Vou te mandar o contrato e a procuração pra você assinar pelo celular. É rapidinho ✍️
---
Se quiser, a gente assina juntas, ao vivo aqui pelo WhatsApp, passo a passo. Posso te mandar agora? 😊
```
Após "sim" → gerar no ZapSign → enviar:
```
{{link_contrato}}
---
Pronto! Assim que você assinar, a gente já começa a organizar tudo e te acompanha até o benefício sair 💛
```
Card → "Contrato enviado".

### E13 — Assinatura confirmada (webhook) `[TEXTO]`
```
Parabéns por essa decisão, {{nome}}! 🥰
---
Agora você está oficialmente na nossa lista de mamães acompanhadas. Vamos juntas garantir esse direito e aliviar um pouco as despesas com a chegada do bebê.
---
Em breve, {{responsavel_2o_atendimento}} vai falar com você pra dar os próximos passos. Qualquer dúvida, é só chamar aqui!
```
- Gestante que vai contribuir: o 2º atendimento orienta a contribuição **antes do parto**.
- Ações: card → "Contrato assinado" → "2º atendimento"; resumo no card; receita prevista; notificação; IA para de acompanhar.

---

## 5. Variações

**5.1 Adoção**
```
Que lindo! 💛 Adoção também dá direito: 120 dias, pagos pelo INSS.
---
Você já tem o termo de guarda para fins de adoção ou a nova certidão de nascimento?
```

**5.2 Perda gestacional / natimorto** 🔴
```
Sinto muito pela sua perda. 💛
---
Não precisa me explicar nada agora. Quando você se sentir à vontade, posso te contar quais são seus direitos.
```
→ não pressionar; transferir para humano.

**5.3 Segundo benefício (carteira + MEI/autônoma)** 🔴
```
Que bom que você me contou! Quem tem carteira assinada e também trabalha como MEI ou por conta própria pode receber dois salários-maternidade: um pela empresa e outro pelo INSS.
---
Você trabalha nessa outra atividade até hoje?
```
→ `E o DAS (ou o carnê) está sendo pago em dia?` → transferir.
Nota: precisa ser **atividade de verdade**, com contribuição paga **antes do parto**. Pagar como "facultativa" tendo carteira não vale.

**5.4 Rural** (uma pergunta por vez)
1. `Que trabalho você faz na roça?`
2. `A terra é sua, da família ou de outra pessoa?`
3. `Você trabalhou na roça no último ano antes do parto?`
4. `Você tem algum documento que mostre esse trabalho? Pode ser ficha do posto de saúde com endereço rural, declaração do sindicato, nota de produtor…`

**5.5 Pedido negado** 🔴
```
Ter o pedido negado não quer dizer que você não tem direito. Muitas negativas são revertidas 💛
---
Consegue me mandar a carta do INSS ou um print do Meu INSS com o motivo?
```

---

## 6. Follow-ups

| Gatilho | Quando | Mensagem |
|---|---|---|
| Parou de responder | 2h · 24h · 3 dias | "Oi, {{nome}}! Ficou alguma dúvida? Tô aqui pra te ajudar 😊" · "Passando pra saber se posso continuar te ajudando com o salário-maternidade 💛" · "Vou deixar seu atendimento em espera. Quando quiser, é só me chamar!" |
| Contrato enviado, não assinado | 2h · 24h · 48h | "Oi, {{nome}}, tudo bem? Vi que o contrato ainda está pendente. Quer que eu te ajude a assinar agora? Leva 2 minutinhos 😊" · "Ficou alguma dúvida no contrato? Posso te explicar qualquer parte." · "Uma informação que pode te ajudar: hoje o INSS tem 30 dias pra analisar o salário-maternidade. Quanto antes a gente começar, antes o dinheiro chega 💛" |
| Gestante que precisa contribuir, sem fechar | 15 dias antes da DPP e no início de cada mês | "Oi, {{nome}}! Sua data do parto tá chegando. Pra garantir o salário-maternidade, a contribuição precisa ser paga **antes do bebê nascer**. Me avisa aqui que eu te ajudo, ainda dá tempo! 💙" |
| Documentos pendentes | 24h · 48h · 5 dias | "Oi, {{nome}}! Só falta {{doc_pendente}} pra gente dar entrada. Consegue me mandar hoje?" |
| Grávida acompanhada (pós-contrato) | 7 dias antes da DPP e na DPP | *(responsabilidade do 2º atendimento)* "Quando o bebê nascer, me manda a certidão pra gente dar entrada 💛" |
| Reativação de base | 30 · 60 · 90 dias | Lembrete curto do direito e do prazo de 5 anos |

**Parar o follow-up quando:** respondeu, assinou, pediu para parar, sem direito confirmado ou foi transferida para humano.

**Cliente responde positivo depois de sumir:** retomar reforçando a segurança e oferecer uma ligação rápida para tirar dúvidas e fechar junto.

---

## 7. Transferência para humano
Ver seção 14 da base. Sempre transferir com resumo no card: situação, data, categoria, contribuições, documentos recebidos, dúvidas e motivo.
