# Calculadoras Jurídicas — Previdenciário (v0.2)

> **Status:** escopo em 3 fases aprovado (19/11/2026). Regras legais ainda precisam ser conferidas por advogado previdenciarista antes de implementar.
> **Convenção:** ✅ decidido · 💡 proposta · ⚠️ conferir/risco.
> **Importante:** toda regra legal abaixo deve ser **conferida por advogado** e coberta por **testes com casos reais** antes de ir para produção. Parâmetros mudam todo ano.

---

## 1. Estratégia

### 1.1 O que o mercado usa hoje
| Ferramenta | O que oferece (segundo os próprios sites) |
|---|---|
| **Cálculo Jurídico (CJ)** | Maior software de cálculos para advogados do Brasil (+105 mil advogados já usaram). +15 tipos de cálculos previdenciários: planejamento, RMI, revisões, liquidação de sentença, tempo de contribuição, regras de transição, incapacidade, pensão, BPC/LOAS. Importa CNIS; IA lê documentos (CNIS, PPP, sentenças). A partir de ~R$ 72/mês (plano anual). |
| **Previdenciarista (Prev)** | Importação do CNIS, cálculo de benefícios pré e pós-Reforma, comparação de cenários, tempo de contribuição, qualidade de segurado, RMI, planejamento, valor da causa e **liquidação de sentença atualizada ao Manual CJF 2026** (Res. CJF 990/2026), abatimentos via HISCRE, honorários, RPV/precatório, "Evoluir RMI". |
| **IEPREV Calc** | Importação do CNIS, conferência de vínculos, comparação de regras, liquidação e valor da causa. |
| **Tramitação Inteligente** | Planilha de cálculo previdenciário e planejamento a partir do CNIS. |

### 1.2 Posicionamento ✅
- Nossas calculadoras ficam **dentro do atendimento**: calculam rápido, **ligadas ao card do cliente**, e alimentam a **receita prevista** e a resposta da IA.
- **Fase 1 — atendimento:** calculadoras dos produtos vendidos (salário-maternidade, BPC, auxílio-acidente) + qualidade de segurado + motor de atrasados/valor da causa/honorários. Foco: **decidir rápido se vale pegar o caso e quanto ele vale**.
- **Fase 2 — processo:** liquidação completa, CNIS, RMI, tempo de contribuição.
- **Fase 3 — avaliar:** planejamento de aposentadoria (CJ e Prev são muito fortes aqui).
- Princípios: **memória de cálculo auditável**, **parâmetros versionados por data e fonte**, aviso **"estimativa — conferir antes de protocolar"**.

---

## 2. Parâmetros legais (tabela versionada no banco)

| Parâmetro | Valor | Vigência | Fonte |
|---|---|---|---|
| Salário mínimo | R$ 1.621,00 | desde 01/01/2026 | Decreto 12.797/2025 |
| Teto do RGPS (benefício e salário de contribuição) | R$ 8.475,55 | desde 01/01/2026 | Portaria Interministerial MPS/MF nº 13/2026 |
| Piso de benefício e de salário de contribuição | R$ 1.621,00 | desde 01/01/2026 | Portaria Interministerial MPS/MF nº 13/2026 |
| Reajuste de benefícios acima do mínimo | 3,90% (INPC 2025) | jan/2026 | Portaria Interministerial MPS/MF nº 13/2026 |
| Limite de renda per capita do BPC | ¼ do SM = R$ 405,25 | 2026 | LOAS |
| Teto do JEF / limite de RPV federal | 60 SM = R$ 97.260,00 | 2026 | Lei 10.259/2001, art. 3º |

- ⚠️ Esses valores mudam todo ano. **Nunca fixar no código**: tabela `parametros_legais` com vigência e fonte; o painel master permite atualizar.
- **Índices mensais** (INPC, IPCA-E, Selic, taxa legal): tabela `indices_mensais`, atualizada todo mês por um job do worker (Banco Central – SGS; IBGE – SIDRA), com alerta se faltar algum mês.

---

## 3. Fase 1 — calculadoras do atendimento

### 3.1 Salário-maternidade
**Entradas:** categoria da segurada; data do parto/adoção; salários de contribuição (quando necessário); data do requerimento; última contribuição (para checar qualidade de segurada).

**Regras:**
- **Carência: não é exigida** para nenhuma categoria — STF, ADIs 2.110 e 2.111 (efeitos a partir de 05/04/2024), incorporado pelo INSS na IN PRES/INSS 188/2025. Basta a **qualidade de segurada** (ver 3.4).
- **Valor por categoria:**
  - Empregada e avulsa: remuneração integral (paga pela empresa).
  - Empregada doméstica: último salário de contribuição.
  - Contribuinte individual / MEI / facultativa: média dos últimos 12 salários de contribuição. ⚠️ conferir regra do período de apuração.
  - Segurada especial (rural): 1 salário mínimo.
  - Desempregada em período de graça: média das contribuições, com piso do SM.
  - Limites: piso R$ 1.621,00; teto R$ 8.475,55 (exceto empregada, cuja remuneração integral é paga pela empresa).
- **Duração:** 120 dias (4 parcelas).
- ⚠️ Conferir: abono anual (13º) proporcional sobre o salário-maternidade.
- Prescrição: parcelas dos últimos 5 anos.

**Saídas:** valor mensal, total do benefício, atrasados corrigidos (se for ação judicial), honorários, receita prevista.

**Exemplo (segurada especial, 2026):** 4 × R$ 1.621,00 = R$ 6.484,00; 30% de honorários ≈ R$ 1.945 → bate com a estimativa de ~R$ 2.000 por contrato.

### 3.2 BPC/LOAS — verificador de renda + estimativa
**Entradas:** espécie (idoso 65+ ou pessoa com deficiência); membros do grupo familiar que moram na mesma casa e renda de cada um; benefícios de até 1 SM recebidos por idoso/PcD do grupo; despesas comprovadas (medicamentos, fraldas, alimentação especial); data do requerimento (DER).

**Regras:**
- Valor: sempre **1 salário mínimo**. **Não paga 13º.**
- Renda per capita até **¼ do SM** (R$ 405,25 em 2026).
- Benefício previdenciário ou outro BPC de até 1 SM de membro idoso/PcD **não entra** no cálculo da renda.
- Despesas de saúde, medicamentos, alimentação especial e fraldas **podem ser consideradas**, conforme o regulamento. ⚠️ Conferir critério de apuração do Decreto 12.534/2025.
- PcD: impedimento de longo prazo (efeitos por no mínimo 2 anos) — avaliação médica e social do INSS.

**Saídas:** renda per capita; status **"dentro do critério" / "acima — avaliar despesas e vulnerabilidade (análise do advogado)" / "fora do critério"**; atrasados desde a DER; honorários; receita prevista.

### 3.3 Auxílio-acidente
**Entradas:** data do acidente; data de cessação do auxílio por incapacidade temporária (se houve); salários de contribuição desde 07/1994 (digitados ou, na fase 2, importados do CNIS).

**Regras:**
- Valor = **50% do salário de benefício (SB)**.
- SB para acidente a partir de 13/11/2019 (pós-EC 103): média de **100%** dos salários de contribuição desde 07/1994. Antes disso: média dos **80% maiores**.
- **Pode ficar abaixo do salário mínimo** (natureza indenizatória). Máximo: 50% do teto.
- ⚠️ Conferir: data de início (dia seguinte à cessação do auxílio por incapacidade), pagamento até a aposentadoria, reajustes anuais.
- Prescrição: parcelas dos últimos 5 anos.

**Saídas:** valor inicial, valor atual (reajustado), atrasados, honorários, receita prevista.

### 3.4 Qualidade de segurado e período de graça
**Entradas:** data da última contribuição; categoria; número de contribuições sem perda da qualidade; desemprego comprovado.

**Regras (Lei 8.213/91, art. 15):** ⚠️ conferir antes de implementar
- 12 meses após a última contribuição (regra geral).
- +12 meses se tiver mais de 120 contribuições sem perda da qualidade.
- +12 meses se comprovar desemprego.
- Facultativo: 6 meses.
- Data exata da perda conforme a regra legal do mês de recolhimento.

**Saída:** "mantém a qualidade de segurado até DD/MM/AAAA". Usada pelo salário-maternidade (desempregada) e pelo auxílio-acidente.

### 3.5 Motor comum: atrasados, valor da causa e honorários
**Entradas:** valor mensal (ou RMI), data de início (DIB/DER), data do cálculo, valores já recebidos.

**Regras:**
- Parcelas mês a mês, com reajustes anuais; 13º quando a espécie tiver (BPC não tem).
- Prescrição quinquenal.
- **Correção e juros — Manual de Cálculos da Justiça Federal (Res. CJF 990/2026):**
  - Até nov/2021: INPC (benefício previdenciário) ou IPCA-E (benefício assistencial) + juros da poupança.
  - Dez/2021 a ago/2025: **Selic** (correção e juros juntos).
  - A partir de set/2025: **INPC + juros pela taxa legal** (Selic menos INPC) para benefícios previdenciários.
  - ⚠️ Conferir no Manual a regra dos **benefícios assistenciais (BPC)** a partir de set/2025.
- **Compensação** de valores recebidos na via administrativa: **mês a mês**, limitada ao valor devido em cada mês, sem resultado negativo (STJ, Tema 1.207, incorporado ao Manual).
- **Valor da causa:** parcelas vencidas + 12 vincendas. Comparar com 60 SM (R$ 97.260,00): até esse valor → JEF; acima → vara federal ou renúncia expressa ao excedente.
- **RPV × precatório:** até 60 SM → RPV.
- ✅ **Honorários contratuais (padrão do dono):** **30% dos atrasados** + **parcelas futuras cobradas mensalmente**. Configurável por escritório e por produto:
  - `percentual_atrasados` (padrão 30%)
  - `parcelas_futuras_qtd` e `parcelas_futuras_percentual` (⚠️ valores a definir — ver `PENDENCIAS.md`)
- **Receita prevista** gerada em duas partes: **valor dos atrasados** (recebimento único, previsto para a data estimada de pagamento) + **parcelas mensais futuras** (uma previsão por mês).

**Saídas:** memória de cálculo mês a mês, totais, enquadramento JEF/RPV, honorários (atrasados + parcelas futuras), cronograma de receita.

---

## 4. Fase 2 — cálculos de processo ✅
- Liquidação de sentença completa pelo Manual CJF 2026: abatimentos (HISCRE), honorários de sucumbência, multas, acordos, RPV/precatório.
- **Importação do CNIS (PDF) com IA:** vínculos, salários de contribuição, indicadores e pendências → base para RMI e auxílio-acidente.
- Tempo de contribuição.
- Evolução da RMI (reajustes anuais).

## 5. Fase 3 — avaliar ✅
- Planejamento de aposentadoria e regras de transição da EC 103 e comparador de cenários.
- ❌ **Não construir "revisão da vida toda":** o STF (ADIs 2.110 e 2.111) definiu que a regra de transição do fator previdenciário é de aplicação obrigatória.
- Depois: calculadoras trabalhista e criminal.

---

## 6. Saídas e uso no sistema
- **PDF padrão:** logo e dados do escritório, dados do cliente, parâmetros usados (com data e fonte), memória de cálculo, aviso de estimativa.
- **Resumo para WhatsApp:** texto curto e simples. Ex.: "Pelo que você me contou, o valor estimado do seu salário-maternidade é de R$ X por mês, por 4 meses. Esse valor é uma estimativa e depende da análise do INSS."
- **Salvar no card** e **atualizar a receita prevista** automaticamente.
- ✅ **A IA pode informar o valor estimado ao cliente** durante o atendimento, pela ferramenta `calcular_estimativa(produto, dados)` que usa o mesmo motor. Regras: a IA **nunca inventa valor** (só repassa o que o motor calculou), apresenta sempre como **estimativa** e **não promete resultado**. Os honorários **não** são informados pela IA, salvo se o escritório configurar.

---

## 7. Notas técnicas 💡
- **Motor de cálculo puro em TypeScript**, separado da interface, com **testes automatizados** contra casos de referência (validados por advogado e/ou comparados com CJ/Prev).
- Tabelas: `parametros_legais` (nome, valor, início/fim de vigência, fonte), `indices_mensais` (índice, competência, valor, fonte), `calculos` (escritório, card, tipo, entradas, resultado, versão do motor e dos parâmetros, link do PDF), `regras_honorarios` (escritório, produto, percentual dos atrasados, parcelas futuras).
- Cada cálculo guarda a **versão dos parâmetros** usada, para poder ser reproduzido depois.
- PDF gerado no servidor a partir de um modelo HTML.

---

## 8. A confirmar
**Com o dono:**
1. Parcelas futuras: quantidade e percentual de cada (por produto).

**Com advogado previdenciarista (antes de programar):**
2. Regra exata da média para contribuinte individual/MEI no salário-maternidade.
3. Abono anual (13º) no salário-maternidade.
4. Índice dos benefícios assistenciais a partir de set/2025 no Manual CJF 2026.
5. Critério do Decreto 12.534/2025 para a renda do BPC.
6. Regras exatas do período de graça e da data de início do auxílio-acidente.
7. Casos reais (anonimizados) para os testes do motor.

---

## Fontes consultadas (19/11/2026)
- Cálculo Jurídico — calculojuridico.com.br (páginas de cálculos previdenciários e planos)
- Previdenciarista — "Prev atualiza liquidação de sentença ao Manual CJF 2026" e "O que avaliar em um sistema de cálculo previdenciário"
- IEPREV — "CJF atualiza Manual de Cálculos: o que muda em 2026"; "Teto do JEF em 2026"; "BPC/LOAS em 2026"; "Salário-maternidade em 2026"
- IBDP — "Novo Manual de Cálculos da Justiça Federal muda forma de calcular atrasados previdenciários" (14/09/2026)
- Gilberto Melo — Resolução CJF nº 990, de 3 de julho de 2026
- Portaria Interministerial MPS/MF nº 13/2026 (via Portal da Indústria, Diário de Pernambuco, NSC Total)
- Barbieri Advogados — salário mínimo 2026 e salário-maternidade
- Âmbito Jurídico — auxílio-acidente em 2026
- Correio 24 Horas / Previdenciarista — critério de renda do BPC em 2026
