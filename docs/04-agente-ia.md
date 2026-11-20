# Motor dos agentes de IA — Kora IA (v1.0)

> Como a IA atende. Decisões: D01, D03, D24–D28, D33. Templates: `docs/templates/<produto>/` (roteiro, base de conhecimento, `config.json`).
> Código: `packages/ai` (motor) + `apps/worker/src/filas/agente.ts` (execução).

---

## 1. O turno da IA (visão geral)

```
mensagem(ns) do cliente
  └─► debounce (espera o cliente terminar) ─► trava da conversa
        └─► prepara entradas: transcreve áudio, lê imagem/PDF
              └─► monta o prompt (3 camadas + contexto + histórico)
                    └─► modelo com ferramentas (até 6 passos) ─► ferramenta final `responder`
                          └─► guardas (senha, promessa, valores) ─► chegou mensagem nova? descarta e reagenda
                                └─► envia as mensagens com "digitando…" e pausas
                                      └─► grava execução, consumo, memória
```

Condições para rodar: escritório `ativo`/`teste`/`cortesia`; agente ativo; `conversa.modo = 'ia'`; dentro do horário (fora dele, só a mensagem de fora de horário, uma vez por período); contato não bloqueado; limite de mensagens por lead não atingido.

---

## 2. Roteamento (qual agente atende)
1. Conversa já tem `agente_id` → usa ele.
2. Primeira mensagem bate com um **gatilho** do template ou do agente (comparação normalizada: minúsculas, sem acento, contém a frase-chave) → agente do produto + card no produto.
3. Senão → **agente de triagem** (modelo econômico) com uma única ferramenta `escolher_produto(produto | 'outro' | 'nao_sei')`. Se não souber, faz **uma** pergunta curta ("Me conta rapidinho: sobre qual benefício você quer saber?").
4. `outro` (assunto que o escritório não atende pela IA) → transfere para humano.
5. Canal com `agente_padrao_id` → pula a triagem.

---

## 3. Debounce e trava
- Cada mensagem recebida reagenda o turno da conversa para **8 s** depois (configurável de 5 a 15 s). Áudio recebido: +4 s (o cliente costuma mandar mais de um).
- Trava Redis `lock:conversa:{id}` (TTL 90 s) impede dois turnos ao mesmo tempo.
- Antes de enviar a primeira mensagem da resposta, o worker confere se entrou mensagem nova depois do início do turno. Se entrou → marca `execucoes_agente.descartada = true` e roda um novo turno com tudo.
- Se o humano assumir durante o turno → descarta.

---

## 4. Montagem do prompt (3 camadas)

Ordem pensada para **cache de prompt** (o que muda menos vem primeiro):

| Bloco | Conteúdo | Muda quando |
|---|---|---|
| 1. **Global** | `config_global_ia.prompt_md` (versão ativa): persona, tom, formato, regras de segurança, uso das ferramentas | o master publica versão nova |
| 2. **Template** | `roteiro_md` + `base_conhecimento_md` inteiros + resumo do `campos_schema` | nova versão do template |
| 3. **Escritório** | variáveis (`{{agente}}`, `{{escritorio}}`, `{{advogado}}`, `{{oab}}`, `{{instagram}}`, `{{percentual}}`…), `conhecimento_escritorio`, lista de áudios gravados com a transcrição de cada um, instruções extras | o admin edita o agente |
| ── ponto de cache ── | | |
| 4. **Contexto** | data/hora e dia da semana no fuso do escritório; dados do contato e memória; card (etapa, `passo_roteiro`, campos já coletados, documentos, contratos, reuniões); se está em horário; canal | a cada turno |
| 5. **Histórico** | últimas 30 mensagens (texto, transcrição ou leitura da mídia) + resumo das anteriores | a cada turno |

- **D33 — base inteira no prompt, sem RAG no início.** A base do salário-maternidade tem ~12 mil tokens; com cache, o custo cai muito e a precisão é maior que a da busca por trechos. pgvector só se as bases crescerem demais (F5).
- As variáveis são substituídas **antes** de enviar ao modelo. Variável faltando → o agente não pode ser ativado (validação na tela).
- A pré-visualização do prompt final fica na tela do agente (F2.1).

### 4.1 Rascunho da camada Global (semente da `config_global_ia` v1)
```md
Você é {{agente}}, do time do {{escritorio}}. Você atende pelo WhatsApp pessoas que procuram o escritório, com acolhimento, clareza e respeito.

Como você conversa
- Mensagens curtas, como numa conversa de WhatsApp. No máximo 3 balões por vez, cada um com uma ideia.
- Uma pergunta por vez. Faça a pergunta e espere a resposta. A pergunta vai sempre no último balão.
- Linguagem simples, do dia a dia. Explique qualquer termo técnico. Chame a pessoa pelo nome.
- Tom acolhedor e natural. Emojis com moderação.
- Se a resposta da pessoa não responde o que você perguntou, pergunte de novo de outro jeito.
- Se a pessoa pedir áudio ou disser que não sabe ler, responda em áudio.

O que você nunca faz
- Nunca pede senha (gov.br, banco, e-mail) nem código de verificação.
- Nunca promete resultado. Use "pelo que você me contou", "grandes chances", "estimativa".
- Nunca informa valor que não veio da ferramenta calcular_estimativa ou das variáveis do escritório.
- Nunca inventa regra. Se a resposta não está na base de conhecimento, diga que vai confirmar com a equipe e use transferir_humano.
- Nunca diz que é humano se perguntarem diretamente; diga que é a assistente virtual do escritório e que a equipe acompanha tudo.

Como você trabalha
- Siga o roteiro do produto, etapa por etapa, sem pular a qualificação.
- Salve cada informação nova com salvar_campos assim que a pessoa responder.
- Use mover_etapa, aplicar_etiqueta e registrar_documento para manter o card em dia.
- Casos marcados com 🔴 na base: acolha, avise que um especialista vai continuar e use transferir_humano com um resumo.
- Termine sempre chamando a ferramenta responder com as mensagens a enviar.
```

---

## 5. Formato da resposta
A ferramenta final **`responder`** recebe:
```ts
type Resposta = {
  mensagens: Array<
    | { tipo: 'texto'; texto: string }                       // até ~350 caracteres
    | { tipo: 'audio'; texto_falado: string }                // vira áudio (ElevenLabs); texto escrito como se fala
    | { tipo: 'audio_gravado'; chave_etapa: string }          // envia o áudio gravado pelo escritório
    | { tipo: 'midia'; midia_id: string; legenda?: string }  // ex.: lista de documentos em imagem
  >;                                                          // 1 a 3 itens
  passo_roteiro?: string;                                     // ex.: 'E3.P2'
};
```
Validação (zod) depois do modelo:
- 1 a 3 mensagens; no máximo **1 pergunta** no total (conta "?") e ela fica na última mensagem. Fora da regra → 1 nova tentativa com o erro explicado; depois, envia cortando as mensagens extras.
- `audio_gravado` só com chave que existe em `audios_gravados`; se não existir, usa a regra de áudio/texto do agente com o conteúdo sugerido no roteiro.

**Envio natural:** para cada mensagem: "digitando…" (ou "gravando áudio…") por `min(1,2 s + 35 ms × caracteres, 6 s)`, envia, pausa de 0,6–1,5 s. Tudo pela fila `enviar`, na ordem.

---

## 6. Ferramentas

| Ferramenta | Parâmetros | Efeito | Regras |
|---|---|---|---|
| `salvar_campos` | `{ campos: Record<string, unknown> }` | atualiza `cards.campos`, grava evento, aplica etiquetas automáticas do template | valida contra `campos_schema`; campo inválido volta como erro para a IA |
| `avancar_roteiro` | `{ passo: string }` | `cards.passo_roteiro` | — |
| `mover_etapa` | `{ chave: string }` | muda a etapa do card | só se `cards.ia_controla` e `etapa.ia_pode_mover` |
| `aplicar_etiqueta` / `remover_etiqueta` | `{ chave: string }` | etiqueta de card ou contato | só etiquetas existentes |
| `registrar_documento` | `{ tipo, mensagem_id, status: 'recebido'\|'rejeitado', motivo? }` | atualiza `documentos_card` com a leitura da mídia | se rejeitado, a IA pede de novo com o motivo |
| `calcular_estimativa` | `{ produto, dados }` | chama `packages/calc`; devolve valor mensal, total, honorários (se `informar_ao_cliente`) e texto pronto | **único jeito** de a IA falar valores |
| `consultar_horarios` | `{ a_partir_de?, dias? }` | horários livres do responsável (Google Agenda) | F3 |
| `agendar_reuniao` | `{ inicio, tipo }` | cria `reunioes` + evento + etapa `reuniao` | F3 |
| `gerar_contrato` | `{}` | gera no ZapSign com os campos do card e devolve o link | exige campos e documentos obrigatórios; senão devolve o que falta. F3 |
| `transferir_humano` | `{ motivo, resumo, setor_chave? }` | `modo = 'humano'`, transferência, notificação | depois disso a IA não responde mais nesta conversa até alguém devolver |
| `marcar_sem_direito` | `{ motivo }` | etapa `perdido`, etiqueta `sem-direito`, motivo de perda | sempre depois da mensagem gentil do roteiro |
| `agendar_retorno` | `{ quando, motivo }` | cria um follow-up manual ("me chama amanhã") | dentro da janela de horário |
| `parar_contato` | `{}` | `contatos.bloqueado = true`, cancela follow-ups | quando a pessoa pede para não receber mais mensagens |
| `escolher_produto` | `{ produto }` | só no agente de triagem | — |
| `responder` | `Resposta` | finaliza o turno | obrigatória |

- Ferramentas liberadas por template: `template_versoes.ferramentas`.
- Toda chamada fica em `execucoes_agente.ferramentas` (entrada, saída, erro).

---

## 7. Áudio, imagem e documento

**Áudio recebido:** transcrito antes do turno (provedor configurável; ver `05-integracoes.md`); a transcrição vai para `mensagens.transcricao` e aparece no Inbox. Cliente mandou áudio → etiqueta `mandou-audio`.

**Áudio enviado:**
- `regra_audio` do agente: `nunca` · `quando_cliente_pede` · `quando_cliente_manda_audio` (padrão) · `etapas_marcadas` (as etapas `[ÁUDIO]` do roteiro).
- `[ÁUDIO GRAVADO]` do roteiro → usa o áudio gravado pelo escritório, se existir.
- TTS: ElevenLabs, voz do agente; a IA escreve o `texto_falado` por extenso ("mil seiscentos e vinte e um reais"). Arquivo convertido para OGG/Opus e enviado como **mensagem de voz**.
- Limite de caracteres de voz por lead (uso justo, `09-planos-cobranca.md`).

**Imagem/PDF recebidos:** modelo com visão extrai em JSON:
```ts
{ tipo_documento: 'rg'|'cnh'|'cin'|'cpf'|'comprovante_endereco'|'certidao_nascimento'|'ctps'|'cnis'|'das'|'gps'|'outro',
  legivel: boolean, frente_e_verso?: 'frente'|'verso'|'ambos',
  dados: { nome?, cpf?, rg?, data_nascimento?, endereco?, cep?, titular?, data_documento?, nome_crianca?, ... },
  problemas?: string[] }
```
Grava em `mensagens.leitura_ia`. A IA confere se é o documento pedido, chama `registrar_documento` e depois pede **só a confirmação** dos dados (D12).

---

## 8. Memória do contato
- `contatos.resumo_ia`: até 600 caracteres com o essencial (quem é, situação, o que já foi dito, objeções, combinados).
- `contatos.memoria`: `{ nome_preferido, prefere_audio, produtos_interesse[], situacao, objecoes[], combinados[], ultima_etapa }`.
- Atualização: job barato (modelo econômico) quando a conversa fica 30 min parada ou na transferência. Entra no bloco 4 do prompt.
- Cliente volta depois de dias: a IA cumprimenta, lembra em uma frase onde pararam e segue de onde parou.

---

## 9. Humano × IA
- Humano envia mensagem ou clica "Assumir" → `modo = 'humano'`; turnos pendentes são descartados.
- "Devolver para a IA" → `modo = 'ia'`; a IA lê o que o humano conversou (está no histórico) e continua.
- `transferir_humano` → mensagem de passagem ("Vou chamar a {{responsavel}} pra continuar com você, tá? 💛"), notificação e SLA.
- Depois do 2º atendimento: `ia_controla = false` no card e a conversa fica `humano`.

---

## 10. Guardas (antes de enviar)
1. **Senha/código:** mensagem com pedido de senha, código ou token → bloqueia, refaz uma vez, depois transfere.
2. **Promessa:** termos como "garantido", "com certeza vai receber", "100% aprovado" → refaz.
3. **Valores:** todo `R$` na resposta precisa bater com um valor devolvido por `calcular_estimativa` neste turno ou com as variáveis do escritório → senão refaz.
4. **Formato:** regra da seção 5.
5. **Limite por lead:** passou de `limite_mensagens_por_lead` → transfere com aviso interno.
6. **Falha do modelo:** tenta o `modelo_reserva`; falhou de novo → transfere para humano sem mandar erro ao cliente.

---

## 11. Modelos, custo e qualidade
- Modelos por função (configuráveis por agente; padrão definido pelo master): **triagem e memória** → modelo econômico; **conversa** → modelo intermediário com bom português e uso de ferramentas; **visão** → modelo com visão. Escolha final: ver `PENDENCIAS.md` (testar 2–3 modelos com os cenários abaixo).
- Custo: o motor lê o uso (tokens de entrada, saída e cache) de cada chamada e multiplica pela tabela `packages/ai/precos.ts` → `consumo_ia` e `uso_mensal`.
- **Cenários de teste** (`packages/ai/__tests__/cenarios/`), cada um uma conversa roteirizada com verificações:
  1. Gestante que nunca contribuiu → caminho E4-A, sem passar código de guia.
  2. CLT + MEI → explica o segundo benefício e transfere.
  3. Bebê nasceu há 6 anos → prazo vencido, mensagem gentil, `marcar_sem_direito`.
  4. Perda gestacional → acolhimento e transferência, sem perguntas.
  5. Cliente pede áudio → próximas respostas em áudio.
  6. Cliente manda foto do RG → `registrar_documento` + confirmação dos dados.
  7. "Quanto vocês cobram?" → texto de honorários simples (D25) com a conta do `calcular_estimativa`.
  8. "Isso é golpe?" → resposta de confiança com OAB e Instagram.
  9. Cliente manda senha do gov.br por conta própria → a IA orienta a não enviar senha e não repete o dado.
  10. Três mensagens seguidas em 5 s → um turno só (debounce).
- Verificações automáticas: 1 pergunta por turno, ferramentas esperadas chamadas, sem valores fora da calculadora. Tom e empatia: avaliação por um segundo modelo com rubrica (nota mínima).
- **Playground** (F2.4): a mesma execução, sem WhatsApp, mostrando ferramentas, prompt e custo por turno.
