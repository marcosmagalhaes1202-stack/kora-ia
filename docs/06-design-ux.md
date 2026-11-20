# Design e UX — Kora IA (v1.0)

> Base para o design system do app e para o Claude Design. Decisões: D18, D29–D31, D36.
> Implementação: tokens em `apps/web/src/styles/globals.css` (Tailwind v4 `@theme`), componentes shadcn ajustados em `components/ui`.

---

## 1. A marca

**Kora IA — CRM Jurídico.** Logo em duas cores: **"Kora" em azul-tinta** e **"IA" em verde-folha**, com "CRM JURÍDICO" em grafite.

**Ideia central do visual: o verde é a IA.** No logo, o verde marca só o "IA". No sistema é igual: **o verde aparece onde a IA está agindo** (conversa que a IA atende, mensagem que a IA escreveu, card que a IA controla, campo que a IA preencheu). O resto da interface é branco, tinta e cinzas. Assim a cor tem significado: o advogado bate o olho e sabe o que foi a IA e o que foi a equipe.

**Personalidade:** sério e confiável (é jurídico), mas leve e acolhedor (o cliente final é uma mãe, um idoso, um trabalhador). Nada de "tribunal" (sem marrom, dourado, serifas pesadas, martelo).

---

## 2. O que tiramos das referências

| Referência | O que levar |
|---|---|
| **Project M** (Kanban roxo) | Sidebar dupla: trilho de ícones + menu com nomes, recolhível. Colunas do Kanban como "trilhos" suaves com contador. Card arrastado levemente inclinado, com o lugar vazio tracejado. Menu "…" no card. |
| **Closewise** (pipeline) | **Cabeçalho da coluna com soma de valores.** Faixa de indicadores acima do quadro. Barra "dias na etapa × tempo normal". Chips de filtro (responsável, período, etiquetas, "+ filtro"). Prazo atrasado em vermelho. Etiquetas pequenas e cinzas. |
| **Monty** (dashboard, settings, tabelas) | Fundo cinza bem claro com **painéis brancos bem arredondados**. Topbar flutuante com busca. Item ativo do menu em **pílula escura**. Botão principal em pílula com ícone. Abas em pílula (segmented). Tabelas limpas com filtros. Configurações com submenu à esquerda e "zona de perigo". |
| **Orven** (onboarding) | Onboarding em passos com cartões "Conectar" — usar (em versão clara) para a **configuração inicial do escritório**. |
| **Fynix** (verde) | Verde associado ao assistente de IA — confirma a ideia do verde = IA. |

---

## 3. Cores

### 3.1 Base (tirada do logo)
| Token | Hex | Uso |
|---|---|---|
| `tinta` | **#2C4058** | Marca, botão principal, item ativo do menu, títulos, links |
| `folha` | **#6AD990** | **Somente IA**: selo, fio lateral, ponto "IA atendendo", destaque de campos preenchidos pela IA |
| `grafite` | **#5C646C** | Texto secundário, ícones |
| `texto` | **#1B2733** | Texto principal |
| `papel` | **#FFFFFF** | Painéis, cards, campos |
| `nevoa` | **#F4F6F8** | Fundo do app (atrás dos painéis) |
| `linha` | **#E4E8ED** | Bordas e divisórias |

### 3.2 Escalas
| | 50 | 100 | 200 | 300 | 500 | 700 | 800 | 900 |
|---|---|---|---|---|---|---|---|---|
| **tinta** | #F0F3F7 | #DDE4EC | #BCC8D6 | #8FA2B9 | #4C6380 | **#2C4058** | #233447 | #1A2736 |
| **folha** | #EDFBF2 | #D3F5DF | #B0EDC6 | **#6AD990** (400) | #3FBF6E | #24874B | #1C6A3B | #134A29 |
| **cinza** | #F4F6F8 | #EBEEF2 | #E4E8ED | #C9CFD6 | #8A929B | #5C646C | #3D454E | #1B2733 |

### 3.3 Contraste (AA) — regras
- `folha` **nunca** como cor de texto ou ícone sobre branco (1,8:1). Sobre `folha`, texto em `tinta` (6:1).
- Texto verde, quando necessário: `folha-700` #24874B (≥ 4,5:1 no branco).
- `grafite` no branco: 6:1 (ok para texto secundário). `cinza-500` #8A929B só para placeholders e texto desabilitado (3,2:1, nunca para informação importante).
- Botão principal: fundo `tinta`, texto branco (10,6:1).

### 3.4 Estados
| Estado | Texto | Fundo | Ícone |
|---|---|---|---|
| Sucesso (ex.: contrato assinado) | #1C6A3B | #E9F7EF | check |
| Alerta (prazo perto, limite 80%) | #9A5B00 | #FEF4E2 | relógio/triângulo |
| Erro / atrasado | #B42318 | #FDEDEC | alerta |
| Informação | #2C4058 | #F0F3F7 | info |

Sucesso usa verde **escuro com ícone de check**; a IA usa **verde-folha com o ícone de faísca**. Os dois nunca dependem só da cor.

### 3.5 Etiquetas (tons pastel, sem verde — o verde é da IA)
| Token | Fundo | Texto |
|---|---|---|
| `ceu` | #E6F0FB | #24507F |
| `lavanda` | #EEEAFB | #4B3C8F |
| `rosa` | #FBE9F1 | #8C2F5B |
| `pessego` | #FDEEE4 | #8A4316 |
| `areia` | #F5F0E6 | #6B5634 |
| `sol` | #FCF5D7 | #7A5B00 |
| `petroleo` | #E3F2F3 | #1F5E63 |
| `cinza` | #EEF1F4 | #3D454E |

---

## 4. Tipografia

- **Inter** — toda a interface: textos, tabelas, chat, formulários, números (com `font-variant-numeric: tabular-nums` em valores e tabelas). É a mais legível em tamanhos pequenos e telas de celular.
- **Montserrat** (600/700) — **só** títulos de página, nome do card no detalhe, números grandes de indicadores e a marca. Conversa com o logo e dá personalidade sem cansar a leitura.
- Lato fica de fora para não ter três famílias.
- Carregar com `next/font` (subconjunto latin + latin-ext, `display: swap`).

| Estilo | Fonte | Tamanho / altura | Peso | Uso |
|---|---|---|---|---|
| `titulo-1` | Montserrat | 28 / 34 | 700 | Título da página |
| `titulo-2` | Montserrat | 20 / 26 | 600 | Título de painel, nome no detalhe do card |
| `numero-grande` | Montserrat | 30 / 36 | 600 | Indicadores (KPI) |
| `titulo-3` | Inter | 16 / 24 | 600 | Título de seção, cabeçalho de coluna |
| `corpo` | Inter | 14 / 20 | 400 | Texto padrão do app |
| `corpo-forte` | Inter | 14 / 20 | 500 | Nomes nas listas, botões |
| `chat` | Inter | 15 / 22 (16 no celular) | 400 | Bolhas de mensagem |
| `pequeno` | Inter | 13 / 18 | 400 | Metadados, prévia da conversa |
| `legenda` | Inter | 12 / 16 | 500 | Etiquetas, horário, contadores |

- Sem rótulos em CAIXA ALTA (só o "CRM JURÍDICO" do logo). Títulos e botões em **frase normal** ("Enviar contrato").
- Campos de formulário com 16 px no celular (evita o zoom do iPhone).

---

## 5. Forma, espaço e profundidade

**Raio por hierarquia** (não é um raio só para tudo):
| Elemento | Raio |
|---|---|
| Painéis grandes (lista, conversa, colunas do Kanban, blocos do dashboard) | 20 px |
| Cards, itens de lista selecionados, popovers, diálogos | 14 px |
| Campos de texto, selects | 12 px |
| Botões, abas (segmented), chips, etiquetas | pílula (999 px) |
| Bolhas de chat | 18 px, com o canto do lado de quem enviou em 6 px |
| Avatares | círculo |

- **Espaçamento:** base 4 px; escala 4, 8, 12, 16, 20, 24, 32, 40, 48. Padding de painel 20–24 px; entre painéis 16 px (12 no celular).
- **Profundidade:** bordas `linha` de 1 px por padrão; **sombra só no que flutua** (popover, menu, card sendo arrastado, sheet): `0 12px 32px -12px rgb(27 39 51 / .22)`. Nada de sombra cinza em todo card.
- Sem gradientes decorativos.
- **Ícones:** lucide, traço 1,75, 18 px (16 em chips).

---

## 6. O elemento assinatura: "fio da IA"

Um único recurso visual forte, usado sempre do mesmo jeito:

- **Fio verde de 3 px** na borda esquerda de tudo que a IA controla ou produziu:
  - card do Kanban com `ia_controla = true`;
  - bolha de mensagem escrita pela IA (as da equipe não têm fio);
  - linha da conversa no Inbox enquanto a IA atende;
  - evento da linha do tempo feito pela IA.
- **Selo IA:** pílula `folha-100` com texto `folha-800`, ícone de faísca e o nome do atendente virtual, no formato "Ana (IA)". Variante "Humano: Paula" em `tinta-50`.
- **Pulso:** um ponto `folha` que pulsa devagar **só** enquanto a IA está gerando a resposta ("Ana está digitando…"). É a única animação contínua do sistema; desligada com `prefers-reduced-motion`.
- Campo do card preenchido pela IA: pequeno ponto `folha` ao lado do rótulo, com tooltip "Preenchido pela IA em 19/11, 14:32".

---

## 7. Layout e telas

### 7.1 Shell
```
┌──────┬──────────────┬──────────────────────────────────────────────────────┐
│ trilho│ menu (232)   │  topbar flutuante: busca ⌘K · número/canal · 🔔 · eu │
│ (64)  │ Kora IA      ├──────────────────────────────────────────────────────┤
│  ◉    │ Inbox     12 │                                                      │
│  ▦    │ Funil        │              conteúdo (fundo névoa,                  │
│  ✓    │ Tarefas    3 │              painéis brancos)                        │
│  …    │ Agenda       │                                                      │
│       │ Agentes      │                                                      │
│       │ Dashboard    │                                                      │
│       │ Calculadoras │                                                      │
│       │ ──────────── │                                                      │
│  ⚙    │ Configurações│                                                      │
│       │ [uso: 132/500 leads]                                                │
└──────┴──────────────┴──────────────────────────────────────────────────────┘
```
- Menu recolhe para o trilho (só ícones) — no Inbox ele começa recolhido para dar espaço.
- Item ativo: pílula `tinta` com texto branco.
- Rodapé do menu: medidor de leads do mês (plano), discreto; vira alerta em 80%.
- **Celular:** barra inferior com 5 itens — Inbox, Funil, Tarefas, Avisos, Mais. Topbar com título e busca.

### 7.2 Inbox (tela principal)
```
┌ conversas (340) ─────────┐┌ conversa ───────────────────────────────┐┌ cliente (360) ──────┐
│ [Todas|Minhas|IA|Humano] ││ Maria Souza  · Comercial 1   [Assumir]  ││ Maria Souza          │
│ 🔍 buscar   [filtros +]   ││ selo: Ana (IA) atendendo                ││ Salário-maternidade  │
│▌● Maria Souza    14:32  2 ││                                          ││ Etapa: Documentos    │
│▌  "Já mandei o RG"        ││   ┌───────────────┐                      ││ Resumo da IA …       │
│   João Lima      13:05    ││   │ msg do cliente │                     ││ Campos ●DPP 12/02    │
│   "Ok, obrigado"          ││   └───────────────┘                      ││ Documentos 2/3       │
│ …                          ││                    ┌────────────────┐▌  ││ Etiquetas            │
│                            ││                    │ msg da IA       │▌  ││ Linha do tempo       │
│                            ││                    └────────────────┘   ││ [Enviar contrato]    │
│                            ││ [ / respostas · 📎 · 🎤 · nota interna ]  ││                      │
└────────────────────────────┘└──────────────────────────────────────────┘└──────────────────────┘
```
- Linha da lista: avatar 40 px, nome (`corpo-forte`), prévia (`pequeno`, 1 linha), horário, contador de não lidas (pílula `tinta`), ícone do canal, fio verde se a IA atende, etiquetas (máx. 2 + "+n").
- Bolhas: cliente à esquerda (fundo `cinza-100`); equipe à direita (fundo `tinta-50`); **IA à direita em branco com borda `linha` e fio verde**; nota interna em `sol` com ícone de cadeado, largura total.
- Áudio: player com onda e velocidade 1×/1,5×/2×; transcrição recolhível abaixo.
- Painel do cliente recolhível; no celular vira uma tela.
- Barra de ação contextual: **Assumir conversa** (quando a IA atende) / **Devolver para a IA** / **Transferir**.

### 7.3 Funil (Kanban)
```
 Funil      [Comercial | Pós-venda]   [Quadro | Lista]           [+ Novo card]
 [Responsável ▾] [Produto ▾] [Etiquetas ▾] [Período ▾] [+ filtro]     🔍
 ┌ indicadores: Em aberto 48 · R$ 93,4 mil previsto · Assinados no mês 12 · Conversão 24% ┐
 ┌ Novo contato 12 ─┐ ┌ Em qualificação 9 ┐ ┌ Documentos 7 ─┐ ┌ Contrato enviado 4 ┐
 │ R$ 23.342        │ │ R$ 17.506         │ │ R$ 13.616     │ │ R$ 7.780          │
 │▌┌──────────────┐ │ │                   │ │               │ │                   │
 │▌│ Maria Souza  │ │ │                   │ │               │ │                   │
 │▌│ Sal.-matern. │ │ │                   │ │               │ │                   │
 │▌│ [MEI][áudio] │ │ │                   │ │               │ │                   │
 │▌│ R$ 1.945 ·2d │ │ │                   │ │               │ │                   │
 │▌│ ◕ Ana (IA)   │ │ │                   │ │               │ │                   │
 │ └──────────────┘ │ │                   │ │               │ │                   │
```
- Coluna = painel `nevoa`-escuro (`cinza-100`) com raio 20; cards brancos raio 14 por cima.
- Cabeçalho: nome, contador, **soma do valor previsto**, botão "+".
- Card: nome (`corpo-forte`), produto (`pequeno`, grafite), até 3 etiquetas, valor previsto (tabular), **barra de tempo na etapa** (fica âmbar/vermelha quando passa do normal), avatar do responsável ou selo da IA, próximo passo/prazo.
- Arrastar: card levanta (sombra + 2° de inclinação) e o lugar de destino aparece tracejado.
- Detalhe do card: **sheet à direita** (largura 560) com abas Resumo · Conversa · Documentos · Contrato · Receita · Linha do tempo.

### 7.4 Outras telas
- **Dashboard:** faixa de 4 indicadores (`numero-grande`), gráfico de contratos por semana, funil de conversão por etapa, receita prevista × realizada, consumo de IA. Gráficos com `tinta` e cinzas; `folha` só para séries da IA (ex.: "atendidos pela IA").
- **Agentes:** cartões por produto (ativo/inativo, conversas no mês, conversão), editor em abas: Identidade · Roteiro (leitura) · Variáveis · Áudios gravados por etapa · Voz e áudio · Horários · Pré-visualização · Playground.
- **Configurações:** submenu à esquerda (Escritório, Equipe, Canais, Integrações, Funil, Etiquetas, Respostas rápidas, Contratos, Honorários, Notificações) + painel; "zona de perigo" no fim de Canais e Escritório.
- **Configuração inicial (onboarding do escritório):** passos — 1 Dados do escritório · 2 Conectar WhatsApp (QR) · 3 Ativar agente · 4 ZapSign · 5 Convidar equipe — com cartões "Conectar" e progresso.
- **Painel master:** mesmo shell, menu próprio, faixa superior em `tinta` escrito "Painel master" para nunca confundir com um escritório. "Entrar como" mostra faixa âmbar fixa "Você está vendo como Escritório X — Sair".

---

## 8. Componentes (lista para a F0.4)
Button (principal `tinta`, secundário contorno, fantasma, perigo; tamanhos 32/40/44), IconButton, Input, Textarea, Select, Combobox, DatePicker, Checkbox, Switch, SegmentedControl (pílula), Tabs, Badge, **Etiqueta** (8 tons), **SeloIA** / SeloHumano, Avatar (+ grupo), Card, Painel, Dialog, Sheet, Popover, DropdownMenu, Tooltip, Toast (Sonner), Skeleton, EmptyState, Tabela (ordenação, seleção, paginação), FiltroChip, BarraDeTempoNaEtapa, PlayerDeAudio, BolhaMensagem, Composer, LinhaDoTempo, KPI, MedidorDeUso.

---

## 9. Movimento
- Só para responder a uma ação: abrir sheet/popover (180 ms, ease-out), arrastar card, confirmar envio (bolha entra com leve subida de 4 px).
- Sem animação de entrada em seções ao carregar a página.
- Pulso da IA (seção 6) é a única animação contínua.
- `prefers-reduced-motion`: tudo instantâneo.

---

## 10. Textos da interface
- Português simples, voz ativa, frase normal. O botão diz o que acontece: **"Enviar contrato"**, **"Assumir conversa"**, **"Devolver para a IA"**, e o aviso confirma com o mesmo verbo ("Contrato enviado").
- Nome das coisas pelo que o advogado entende: "Canais" e "Conectar WhatsApp", não "instância" ou "webhook".
- Vazio = convite para agir: "Nenhuma conversa ainda. Conecte um número de WhatsApp para começar a receber mensagens." + botão.
- Erro diz o que houve e como resolver, sem pedir desculpas: "O número Comercial 1 desconectou. Escaneie o QR code de novo para voltar a receber mensagens."
- Datas relativas nas listas ("há 5 min", "ontem, 14:32"); valores em `R$ 1.945,20`.

---

## 11. Acessibilidade
Contraste AA; foco visível (anel de 2 px `tinta-300` com afastamento de 2 px); navegação por teclado no Inbox (↑/↓ entre conversas, Enter abre, Esc fecha painel) e no Kanban (mover card pelo menu "Mover para…", não só arrastando); áreas de toque ≥ 44 px; textos alternativos; nada transmitido só por cor.

---

## 12. Tokens (Tailwind v4)

```css
@import "tailwindcss";

@theme {
  --font-sans: "Inter", ui-sans-serif, system-ui, sans-serif;
  --font-display: "Montserrat", "Inter", ui-sans-serif, sans-serif;

  --color-tinta-50:#F0F3F7; --color-tinta-100:#DDE4EC; --color-tinta-200:#BCC8D6; --color-tinta-300:#8FA2B9;
  --color-tinta-500:#4C6380; --color-tinta-700:#2C4058; --color-tinta-800:#233447; --color-tinta-900:#1A2736;

  --color-folha-50:#EDFBF2; --color-folha-100:#D3F5DF; --color-folha-200:#B0EDC6; --color-folha-400:#6AD990;
  --color-folha-500:#3FBF6E; --color-folha-700:#24874B; --color-folha-800:#1C6A3B; --color-folha-900:#134A29;

  --color-cinza-50:#F4F6F8; --color-cinza-100:#EBEEF2; --color-cinza-200:#E4E8ED; --color-cinza-300:#C9CFD6;
  --color-cinza-500:#8A929B; --color-cinza-700:#5C646C; --color-cinza-800:#3D454E; --color-cinza-900:#1B2733;

  --color-papel:#FFFFFF; --color-nevoa:#F4F6F8; --color-linha:#E4E8ED;
  --color-texto:#1B2733; --color-grafite:#5C646C;

  --color-sucesso:#1C6A3B; --color-sucesso-fundo:#E9F7EF;
  --color-alerta:#9A5B00;  --color-alerta-fundo:#FEF4E2;
  --color-erro:#B42318;    --color-erro-fundo:#FDEDEC;

  --radius-painel:20px; --radius-card:14px; --radius-campo:12px; --radius-bolha:18px;

  --shadow-flutuante: 0 12px 32px -12px rgb(27 39 51 / .22);
}
```
- Mapear para as variáveis do shadcn: `--primary` = tinta-700, `--primary-foreground` = branco, `--background` = nevoa, `--card` = papel, `--border` = linha, `--ring` = tinta-300, `--muted-foreground` = grafite.
- Etiquetas: classes geradas a partir da tabela 3.5.

---

## 13. Ícone do app (PWA) — a definir
Proposta: quadrado arredondado `tinta` com "K" branco em Montserrat Bold e um ponto `folha` no canto (a IA). Gerar 192, 512 e versão "maskable". O dono pode trocar quando tiver o ícone oficial.
