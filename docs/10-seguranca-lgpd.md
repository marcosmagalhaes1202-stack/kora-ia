# Segurança e LGPD — Kora IA (v1.0)

> O sistema guarda documentos pessoais e, em vários produtos, **dados de saúde** (gestação, deficiência, incapacidade). Isso exige cuidado extra. Itens jurídicos marcados com ⚠️ devem ser validados por advogado especialista em proteção de dados.

---

## 1. Autenticação e sessão
- Supabase Auth com e-mail e senha; senha mínima de 10 caracteres; confirmação de e-mail; recuperação por e-mail com SMTP próprio.
- Sessão por cookie seguro (`@supabase/ssr`), renovação automática.
- 💡 Verificação em duas etapas (TOTP) obrigatória para **master** e opcional para admins (F1).
- Limite de tentativas de login (configuração do Auth) e captcha se houver abuso.

## 2. Autorização
- **RLS em todas as tabelas** (`03-banco-de-dados.md`, seção 3) + checagem de papel nas Server Actions.
- `service_role` só no servidor (worker e rotas do master), nunca no navegador.
- Worker sempre filtra por `escritorio_id` (helpers de `packages/db`).
- Master: toda ação grava `auditoria`; "Entrar como" grava início e fim e mostra faixa na tela.
- Testes automáticos de isolamento entre escritórios para cada tabela.

## 3. Segredos e cifragem
- Segredos do sistema em variáveis de ambiente do Coolify (nunca no repositório; `.env*` no `.gitignore`).
- Credenciais por escritório (token ZapSign, refresh token Google, token da instância) cifradas com **AES-256-GCM** (`ENCRYPTION_KEY`), formato `v1:<iv>:<tag>:<dados>` em base64 — o prefixo de versão permite trocar a chave.
- Segredos de webhook: guardar só o **hash**; comparar em tempo constante.
- Dados em trânsito: HTTPS em tudo; Evolution só na rede interna.
- Dados em repouso: Supabase e R2 cifram em disco.

## 4. Proteções da aplicação
- Cabeçalhos de segurança (CSP, HSTS, X-Frame-Options, Referrer-Policy) no Next.js.
- Validação zod em toda entrada; upload só com tipos permitidos e tamanho máximo; nome de arquivo nunca usado como caminho.
- URLs de mídia sempre assinadas e curtas (5 min).
- Limite de requisições nas rotas públicas (`/webhooks`, `/api/*`).
- Dependências: Dependabot + `pnpm audit` no CI.
- Logs sem dados pessoais (sem CPF, documento, conteúdo de mensagem, token).

## 5. LGPD

### 5.1 Papéis ⚠️
- **Escritório = controlador** dos dados dos seus clientes e leads.
- **Kora IA = operador** (trata os dados em nome do escritório).
- Precisa de: **Termos de uso** + **Acordo de tratamento de dados (DPA)** entre Kora e escritório, e **Política de privacidade** do Kora. (Pendência: redigir com advogado.)

### 5.2 Bases legais (orientação a validar) ⚠️
- Atendimento e contratação: procedimentos preliminares e execução de contrato (art. 7º, V).
- Dados de saúde para pedir benefício ou entrar com ação: exercício regular de direitos em processo administrativo/judicial (art. 11, II, "d").
- Leads que não contratam: legítimo interesse limitado (retorno ao contato que a pessoa iniciou), com opção de parar (`parar_contato`).

### 5.3 Transparência
- 💡 Primeira resposta da IA ou mensagem de perfil do WhatsApp com link curto para a política de privacidade do escritório (pendência: decidir o formato sem quebrar o roteiro).
- A IA se identifica como assistente virtual se perguntarem.

### 5.4 Suboperadores (listar na política e no DPA)
Supabase (banco, possivelmente fora do Brasil conforme região/backup), Cloudflare (R2, DNS), Hostinger (VPS no Brasil), provedores de IA (EUA), ElevenLabs, ZapSign, Google (Agenda), serviço de e-mail. ⚠️ Transferência internacional (IA): cláusulas no DPA; não enviar mais dados do que o necessário ao modelo.

### 5.5 Minimização
- A IA recebe só o necessário: histórico recente, resumo e campos do card. Documentos vão como imagem só no turno em que são lidos.
- Não usar dados dos clientes para treinar modelos (usar APIs com retenção zero/sem treino quando o provedor oferecer).

### 5.6 Retenção
| Dado | Prazo |
|---|---|
| `webhook_eventos` (payload bruto) | 7 dias |
| Mensagens, mídias, cards de **leads que não contrataram** | 💡 24 meses sem interação → anonimizar |
| Dados de **clientes** (contrato assinado) | enquanto o escritório for cliente do Kora (o escritório tem dever de guarda) |
| Escritório cancelado | exportação disponível por 90 dias; depois, exclusão definitiva |
| Backups | 30 dias |
| `auditoria` | 2 anos |

### 5.7 Direitos do titular
- No contato: **Exportar dados** (JSON + mídias em ZIP) e **Excluir/anonimizar** (só admin, com confirmação e auditoria; contratos assinados ficam guardados por obrigação legal do escritório).
- Atendimento de pedido do titular é feito pelo escritório (controlador); o Kora fornece a ferramenta.

### 5.8 Incidentes
- Roteiro: conter → avaliar dados afetados → avisar os escritórios afetados sem demora → apoiar a comunicação à ANPD e aos titulares quando necessário → registrar.

## 6. Ética da advocacia (OAB) na IA
- Sem promessa de resultado; valores como estimativa; sem captação agressiva (follow-up só com quem procurou o escritório; parar quando pedirem).
- Identificação do escritório e do advogado responsável nas mensagens de confiança (`{{advogado}}`, `{{oab}}`).
