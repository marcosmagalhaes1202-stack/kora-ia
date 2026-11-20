# Integrações — Kora IA (v1.0)

> Regra geral: cada integração fica atrás de uma interface nossa (provedor), para trocar de fornecedor sem mexer no resto. Credenciais por escritório ficam em `integracoes`/`canais` **cifradas**; as do dono do SaaS ficam em variáveis de ambiente.
> ⚠️ Endpoints e nomes de eventos abaixo devem ser **conferidos na documentação da versão em uso** antes de implementar. Registre a versão em `PROGRESSO.md`.

---

## 1. WhatsApp não oficial — Evolution API (F1)

**Interface (`packages/channels/tipos.ts`):**
```ts
interface ChannelProvider {
  conectar(canal): Promise<{ qrcode?: string; status: StatusCanal }>;
  desconectar(canal): Promise<void>;
  status(canal): Promise<StatusCanal>;
  enviarTexto(canal, para, texto, opcoes?: { respondeA?: string }): Promise<{ idExterno: string }>;
  enviarMidia(canal, para, midia: { url: string; mime: string; nome?: string; legenda?: string }): Promise<{ idExterno: string }>;
  enviarAudio(canal, para, urlOggOpus: string): Promise<{ idExterno: string }>; // mensagem de voz
  presenca(canal, para, tipo: 'digitando' | 'gravando' | 'parado'): Promise<void>;
  marcarLida(canal, para, idsExternos: string[]): Promise<void>;
  baixarMidia(canal, mensagemBruta): Promise<{ buffer: Buffer; mime: string; nome?: string }>;
  normalizarWebhook(payload: unknown): EventoNormalizado[];   // mensagem | status | conexao | qrcode
}
```

**Evolution (v2.x) — referência:**
| Ação | Endpoint (header `apikey`) |
|---|---|
| Criar instância | `POST /instance/create` (`instanceName`, `integration: "WHATSAPP-BAILEYS"`, `qrcode: true`, webhook) |
| QR / conectar | `GET /instance/connect/{instancia}` |
| Estado | `GET /instance/connectionState/{instancia}` |
| Desconectar / apagar | `DELETE /instance/logout/{instancia}` · `DELETE /instance/delete/{instancia}` |
| Webhook | `POST /webhook/set/{instancia}` → URL `{WEBHOOK_BASE_URL}/webhooks/evolution/{canalId}` com header secreto |
| Texto | `POST /message/sendText/{instancia}` |
| Mídia | `POST /message/sendMedia/{instancia}` |
| Voz | `POST /message/sendWhatsAppAudio/{instancia}` |
| Digitando | `POST /chat/sendPresence/{instancia}` |
| Lida | `POST /chat/markMessageAsRead/{instancia}` |
| Baixar mídia | `POST /chat/getBase64FromMediaMessage/{instancia}` |

**Eventos usados:** `MESSAGES_UPSERT` (nova mensagem, inclusive `fromMe` enviada pelo celular), `MESSAGES_UPDATE` (status), `CONNECTION_UPDATE`, `QRCODE_UPDATED`. Desligar os demais para economizar.

**Cuidados:**
- Uma instância por número; nome da instância = `kora_{canalId curto}`.
- Grupos: **ignorar** mensagens de grupo (`@g.us`) no MVP.
- Mensagens `fromMe` sem registro nosso = enviadas pelo celular → `autor = 'celular'` e **assume a conversa** (modo humano).
- ⚠️ **Anti-banimento:** limites por canal (`canais.limites`); nunca enviar em massa para quem não falou com o escritório; follow-ups só dentro da janela de horário; variar levemente as pausas; parar quando o contato pede.
- ⚠️ Versão 2.4 exige ativação: usar a **última estável sem ativação** (D09) e manter o provedor trocável.
- A Evolution roda com **Postgres próprio** na VPS (não usar o banco do Supabase) e Redis opcional.

## 2. WhatsApp API oficial, Instagram e Messenger (F5)
- `MetaProvider` com a mesma interface. Webhook único `/webhooks/meta` (verificação `hub.challenge` + assinatura `X-Hub-Signature-256`).
- API oficial: fora da janela de 24 h, só **modelos aprovados** (`tipo_passo = 'template_oficial'`). Política Meta 2026: atendimento do próprio negócio é permitido; IA de propósito geral não.
- Instagram/Messenger: app Meta verificado + revisão de permissões; tokens de página por escritório.
- Meta Ads + Conversions API: guardar `ctwa_clid` em `contatos.origem` já na F1 (quando vier), para usar depois.

## 3. ZapSign — contratos (F3)
- Credencial: token de API do **escritório** (cada um tem a sua conta). Sandbox para testes (`ZAPSIGN_API_URL`).
- **Criar documento a partir de modelo:** `POST /api/v1/models/create-doc/` com `template_id`, dados do signatário e `data: [{ de: "{{NOME}}", para: "Maria..." }]` (montado a partir de `modelos_contrato.mapeamento`).
- Contrato + procuração: modelo com os dois documentos ou documento principal + anexo (conferir na conta do escritório).
- **Webhooks** (configurados na conta ZapSign do escritório apontando para `/webhooks/zapsign/{escritorioId}` com segredo): documento criado, signatário visualizou, assinou, recusou.
- ⚠️ **Os links de arquivo expiram em 60 min:** ao receber "assinado", baixar o PDF na hora e salvar no R2.
- O link de assinatura vai na conversa pela IA (não usar o envio automático por WhatsApp da ZapSign, para não duplicar).

## 4. Voz — ElevenLabs (F2)
- Interface `ProvedorVoz.sintetizar(texto, vozId) → Buffer (mp3)`; converter para **OGG/Opus mono** com ffmpeg (no container do worker) e enviar como mensagem de voz.
- Modelo multilíngue com bom português; voz escolhida por agente (`agentes.voz_id`); biblioteca de vozes aprovadas pelo master.
- Medir `caracteres` por chamada → `consumo_ia` (tipo `voz`).

## 5. Transcrição de áudio (F2)
- Interface `ProvedorTranscricao.transcrever(buffer, mime) → { texto, segundos }`.
- Opções: ElevenLabs (speech-to-text), OpenAI ou outro com bom português. **Escolha pendente** (testar com áudios reais de WhatsApp: ruído, sotaque, áudios longos).
- Áudios acima de 10 min: transcrever só os primeiros 10 e avisar a equipe.

## 6. Provedores de IA (F2)
- **Vercel AI SDK**: provedores Anthropic, Google, OpenAI e xAI; o modelo é escolhido por texto `provedor:modelo` (`agentes.modelo`).
- Usar: chamadas com ferramentas, saída estruturada, **cache de prompt** (quando o provedor oferece), leitura de uso de tokens.
- Chaves do dono do SaaS (D02). Tabela de preços em `packages/ai/precos.ts`, revisada quando trocar de modelo.

## 7. Google Agenda (F3)
- OAuth 2.0 por **membro** (advogado), acesso offline (refresh token cifrado em `integracoes`).
- Escopos: criar/editar eventos e consultar livre/ocupado. Callback em `apps/web/app/api/oauth/google/callback`.
- Criar evento com convidado (e-mail do cliente, se houver) e link do Meet opcional; `reunioes.google_event_id`.
- ⚠️ App do Google em "modo teste" aceita poucos usuários; para produção, publicar o app e passar pela verificação de escopos sensíveis. Iniciar o processo cedo.

## 8. Web Push (F0)
- Biblioteca `web-push` (worker) com chaves VAPID. Inscrição pelo service worker (Serwist) → `push_inscricoes`.
- Resposta 404/410 do serviço de push → apagar a inscrição.
- iPhone: só funciona com o app **adicionado à tela de início** (iOS 16.4+). Mostrar um guia de instalação no primeiro acesso pelo celular.

## 9. Cloudflare R2 (F0/F1)
- API compatível com S3 (`@aws-sdk/client-s3` + `s3-request-presigner`), endpoint `https://{R2_ACCOUNT_ID}.r2.cloudflarestorage.com`.
- Buckets privados: `kora-midias`, `kora-backups`. Nada público.
- Regras de ciclo de vida: backups com mais de 30 dias apagados automaticamente.

## 10. E-mail transacional (F0)
- Auth do Supabase (convite, recuperação de senha) com **SMTP próprio** (o SMTP padrão do Supabase tem limite baixo). Opções com plano grátis: Resend, Brevo. Remetente no domínio do Kora (SPF/DKIM).

## 11. Índices econômicos (F4)
- Banco Central (SGS) para Selic e poupança; IBGE (SIDRA) para INPC e IPCA-E. Job mensal na fila `rotinas`, com alerta se faltar mês. Ver `08-calculadoras.md`.
