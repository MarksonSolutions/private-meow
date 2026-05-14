# private-meow — Referência de Métodos do Client

Fork de `tulir/whatsmeow`. Todas as assinaturas abaixo são do tipo `*Client`.

---

## Índice

- [Inicialização](#inicialização)
- [Conexão](#conexão)
- [Proxy](#proxy)
- [Event Handlers](#event-handlers)
- [Envio de Mensagens](#envio-de-mensagens)
- [Mensagens — Builders](#mensagens--builders)
- [Leitura e Recibos](#leitura-e-recibos)
- [Presença](#presença)
- [Usuários e Contatos](#usuários-e-contatos)
- [Grupos](#grupos)
- [Privacidade](#privacidade)
- [Download e Upload de Mídia](#download-e-upload-de-mídia)
- [Chamadas](#chamadas)
- [Newsletters](#newsletters)
- [App State](#app-state)
- [Pairing](#pairing)
- [Push Notifications](#push-notifications)
- [Utilitários](#utilitários)

---

## Inicialização

```go
func NewClient(deviceStore *store.Device, log waLog.Logger) *Client
```
Cria novo cliente WhatsApp. `deviceStore` obrigatório (via `sqlstore`). `log` pode ser nil (no-op).

```go
// Exemplo:
container, _ := sqlstore.New("postgres", dsn, nil)
device, _ := container.GetFirstDevice()
client := whatsmeow.NewClient(device, nil)
```

---

## Conexão

```go
func (cli *Client) Connect() error
func (cli *Client) ConnectContext(ctx context.Context) error
```
Conecta ao servidor WhatsApp Web. Usa `ConnectContext` para controle de timeout.

```go
func (cli *Client) Disconnect()
```
Desconecta limpamente.

```go
func (cli *Client) ResetConnection()
```
Fecha e reabre socket sem remover sessão.

```go
func (cli *Client) Logout(ctx context.Context) error
```
Faz logout e invalida sessão no servidor.

```go
func (cli *Client) IsConnected() bool
func (cli *Client) IsLoggedIn() bool
```
Verifica estado atual.

```go
func (cli *Client) WaitForConnection(timeout time.Duration) bool
```
Bloqueia até conectar ou timeout. Retorna `true` se conectou.

---

## Proxy

```go
func (cli *Client) SetProxyAddress(addr string, opts ...SetProxyOptions) error
```
Define proxy por URL (ex: `"socks5://127.0.0.1:1080"`). `SetProxyOptions`:
```go
type SetProxyOptions struct {
    NoWebsocket bool  // não usar proxy no WebSocket
    NoMedia     bool  // não usar proxy no download de mídia
}
```

```go
func (cli *Client) SetProxy(proxy Proxy, opts ...SetProxyOptions)
func (cli *Client) SetSOCKSProxy(px proxy.Dialer, opts ...SetProxyOptions)
```
Define proxy via interface ou dialer SOCKS5 diretamente.

```go
func (cli *Client) SetMediaHTTPClient(h *http.Client)
func (cli *Client) SetWebsocketHTTPClient(h *http.Client)
func (cli *Client) SetPreLoginHTTPClient(h *http.Client)
```
Configura HTTP clients customizados por uso.

---

## Event Handlers

```go
func (cli *Client) AddEventHandler(handler EventHandler) uint32
```
Registra handler de eventos. Retorna ID do handler.
```go
// EventHandler = func(evt interface{})
// Tipos de evento: *events.Message, *events.Connected, *events.Disconnected,
// *events.Receipt, *events.Presence, *events.GroupInfo, *events.JoinedGroup, etc.
client.AddEventHandler(func(evt interface{}) {
    switch v := evt.(type) {
    case *events.Message:
        fmt.Println("msg:", v.Message.GetConversation())
    case *events.Connected:
        fmt.Println("conectado")
    }
})
```

```go
func (cli *Client) AddEventHandlerWithSuccessStatus(handler EventHandlerWithSuccessStatus) uint32
```
Handler que retorna bool indicando sucesso (usado para acks síncronos).

```go
func (cli *Client) RemoveEventHandler(id uint32) bool
func (cli *Client) RemoveEventHandlers()
```
Remove handler por ID ou todos os handlers.

```go
func (cli *Client) ParseWebMessage(chatJID types.JID, webMsg *waWeb.WebMessageInfo) (*events.Message, error)
```
Converte `WebMessageInfo` (histórico) para `*events.Message`.

---

## Envio de Mensagens

```go
func (cli *Client) SendMessage(
    ctx context.Context,
    to types.JID,
    message *waE2E.Message,
    extra ...SendRequestExtra,
) (resp SendResponse, err error)
```
Envio principal. Funciona para DMs, grupos e newsletters.
```go
// Exemplo — texto simples:
resp, err := client.SendMessage(ctx, jid, &waE2E.Message{
    Conversation: proto.String("Olá!"),
})
```

```go
func (cli *Client) SendPeerMessage(ctx context.Context, message *waE2E.Message) (SendResponse, error)
```
Envia mensagem peer-to-peer (para próprios dispositivos vinculados).

```go
func (cli *Client) RevokeMessage(ctx context.Context, chat types.JID, id types.MessageID) (SendResponse, error)
```
Apaga mensagem para todos ("Delete for everyone").

```go
func (cli *Client) SetDisappearingTimer(
    ctx context.Context,
    chat types.JID,
    timer time.Duration,
    settingTS time.Time,
) error
```
Define timer de mensagens temporárias para um chat.

---

## Mensagens — Builders

Funções auxiliares para construir protobuf de mensagens.

```go
func (cli *Client) BuildRevoke(chat, sender types.JID, id types.MessageID) *waE2E.Message
```
Constrói mensagem de revogação (usar com `SendMessage`).

```go
func (cli *Client) BuildReaction(
    chat, sender types.JID,
    id types.MessageID,
    reaction string,
) *waE2E.Message
```
Constrói reação. `reaction` = emoji ou `""` para remover.

```go
func (cli *Client) BuildEdit(
    chat types.JID,
    id types.MessageID,
    newContent *waE2E.Message,
) *waE2E.Message
```
Constrói edição de mensagem.

```go
func (cli *Client) BuildHistorySyncRequest(
    lastKnownMessageInfo *types.MessageInfo,
    count int,
) *waE2E.Message
```
Constrói request de histórico (enviar com `SendPeerMessage`).

```go
func (cli *Client) BuildMessageKey(chat, sender types.JID, id types.MessageID) *waCommon.MessageKey
func (cli *Client) BuildUnavailableMessageRequest(chat, sender types.JID, id string) *waE2E.Message
```

```go
func (cli *Client) GenerateMessageID() types.MessageID
func GenerateMessageID() types.MessageID  // função livre
```
Gera ID único para mensagem.

---

## Leitura e Recibos

```go
func (cli *Client) MarkRead(
    ctx context.Context,
    ids []types.MessageID,
    timestamp time.Time,
    chat, sender types.JID,
    receiptTypeExtra ...types.ReceiptType,
) error
```
Marca mensagens como lidas (envia read receipt).

```go
func (cli *Client) SetForceActiveDeliveryReceipts(active bool)
```
Força envio de delivery receipts mesmo sem estar "ativo".

```go
func (cli *Client) SendProtocolMessageReceipt(
    ctx context.Context,
    id types.MessageID,
    msgType types.ReceiptType,
) error
```
Envia recibo para mensagens de protocolo (ex: revogação).

```go
func (cli *Client) SendHistorySyncServerErrorReceipt(
    ctx context.Context,
    msgID types.MessageID,
    mediaKey []byte,
) error
```
Sinaliza erro no download de mídia do history sync.

---

## Presença

```go
func (cli *Client) SendPresence(ctx context.Context, state types.Presence) error
```
Define presença global.  
`state`: `types.PresenceAvailable` | `types.PresenceUnavailable`

```go
func (cli *Client) SendChatPresence(
    ctx context.Context,
    jid types.JID,
    state types.ChatPresence,
    media types.ChatPresenceMedia,
) error
```
Envia indicador de digitação/gravando para um chat específico.  
`state`: `types.ChatPresenceComposing` | `types.ChatPresenceRecording` | `types.ChatPresencePaused`  
`media`: `types.ChatPresenceMediaText` | `types.ChatPresenceMediaAudio`

```go
func (cli *Client) SubscribePresence(ctx context.Context, jid types.JID) error
```
Subscreve atualizações de presença de um contato. Requer `SendPresence(Available)` antes.

---

## Usuários e Contatos

```go
func (cli *Client) IsOnWhatsApp(ctx context.Context, phones []string) ([]types.IsOnWhatsAppResponse, error)
```
Verifica se números estão no WhatsApp. `phones` = `["+5511999999999"]`.
```go
type IsOnWhatsAppResponse struct {
    Query string
    JID   types.JID
    IsIn  bool
    VerifiedName *types.VerifiedName
}
```

```go
func (cli *Client) GetUserInfo(ctx context.Context, jids []types.JID) (map[types.JID]types.UserInfo, error)
```
Retorna info de usuários (devices, push name, verified name).

```go
func (cli *Client) GetProfilePictureInfo(
    ctx context.Context,
    jid types.JID,
    params *GetProfilePictureParams,
) (*types.ProfilePictureInfo, error)
```
Busca URL da foto de perfil.
```go
// Parâmetros:
type GetProfilePictureParams struct {
    Preview    bool   // retorna thumbnail
    ExistingID string // retorna nil se ID não mudou
    IsCommunity bool
}
```

```go
func (cli *Client) GetBusinessProfile(ctx context.Context, jid types.JID) (*types.BusinessProfile, error)
```
Busca perfil business (descrição, endereço, email, site, categoria).

```go
func (cli *Client) SetStatusMessage(ctx context.Context, msg string) error
```
Atualiza status/bio da conta.

```go
func (cli *Client) GetBlocklist(ctx context.Context) (*types.Blocklist, error)
```
Retorna lista de contatos bloqueados.

```go
func (cli *Client) UpdateBlocklist(
    ctx context.Context,
    jid types.JID,
    action events.BlocklistChangeAction,
) (*types.Blocklist, error)
```
Bloqueia ou desbloqueia contato.  
`action`: `events.BlocklistChangeActionBlock` | `events.BlocklistChangeActionUnblock`

```go
func (cli *Client) GetContactQRLink(ctx context.Context, revoke bool) (string, error)
```
Retorna link QR do contato da própria conta. `revoke=true` gera novo link.

```go
func (cli *Client) ResolveContactQRLink(ctx context.Context, code string) (*types.ContactQRLinkTarget, error)
```
Resolve link QR de um contato externo.

```go
func (cli *Client) ResolveBusinessMessageLink(ctx context.Context, code string) (*types.BusinessMessageLinkTarget, error)
```
Resolve link de mensagem business (wa.me).

```go
func (cli *Client) GetUserDevices(ctx context.Context, jids []types.JID) ([]types.JID, error)
func (cli *Client) GetUserDevicesContext(ctx context.Context, jids []types.JID) ([]types.JID, error)
```
Retorna JIDs de todos os dispositivos vinculados de cada usuário.

```go
func (cli *Client) StoreLIDPNMapping(ctx context.Context, first, second types.JID)
```
Armazena mapeamento LID↔PN (usado internamente após resolve).

---

## Grupos

```go
func (cli *Client) CreateGroup(ctx context.Context, req ReqCreateGroup) (*types.GroupInfo, error)
```
Cria grupo.
```go
type ReqCreateGroup struct {
    Name          string
    Participants  []types.JID
    CreateKey     types.MessageID  // opcional
    LinkedParentJID types.JID      // para comunidades
}
```

```go
func (cli *Client) GetGroupInfo(ctx context.Context, jid types.JID) (*types.GroupInfo, error)
```
Retorna informações do grupo (participantes, admins, desc, configurações).

```go
func (cli *Client) GetJoinedGroups(ctx context.Context) ([]*types.GroupInfo, error)
```
Lista todos os grupos em que a conta está.

```go
func (cli *Client) LeaveGroup(ctx context.Context, jid types.JID) error
```
Sai de um grupo.

```go
func (cli *Client) UpdateGroupParticipants(
    ctx context.Context,
    jid types.JID,
    participantChanges []types.JID,
    action ParticipantChange,
) ([]types.GroupParticipant, error)
```
Adiciona, remove, promove ou rebaixa participantes.  
`action`: `ParticipantChangeAdd` | `ParticipantChangeRemove` | `ParticipantChangePromote` | `ParticipantChangeDemote`

```go
func (cli *Client) GetGroupRequestParticipants(ctx context.Context, jid types.JID) ([]types.GroupParticipantRequest, error)
func (cli *Client) UpdateGroupRequestParticipants(
    ctx context.Context,
    jid types.JID,
    participantChanges []types.JID,
    action ParticipantRequestChange,
) ([]types.GroupParticipant, error)
```
Gerencia solicitações de entrada (grupos com aprovação).

```go
func (cli *Client) SetGroupName(ctx context.Context, jid types.JID, name string) error
func (cli *Client) SetGroupTopic(ctx context.Context, jid types.JID, previousID, newID, topic string) error
func (cli *Client) SetGroupDescription(ctx context.Context, jid types.JID, description string) error
func (cli *Client) SetGroupPhoto(ctx context.Context, jid types.JID, avatar []byte) (string, error)
```
Atualiza metadados do grupo.

```go
func (cli *Client) SetGroupLocked(ctx context.Context, jid types.JID, locked bool) error
```
Restringe edição de info do grupo a admins.

```go
func (cli *Client) SetGroupAnnounce(ctx context.Context, jid types.JID, announce bool) error
```
Restringe envio de mensagens a admins (modo anúncio).

```go
func (cli *Client) SetGroupJoinApprovalMode(ctx context.Context, jid types.JID, mode bool) error
```
Exige aprovação de admin para novos membros.

```go
func (cli *Client) SetGroupMemberAddMode(
    ctx context.Context,
    jid types.JID,
    mode types.GroupMemberAddMode,
) error
```
Define quem pode adicionar membros.

```go
func (cli *Client) GetGroupInviteLink(ctx context.Context, jid types.JID, reset bool) (string, error)
```
Retorna link de convite. `reset=true` gera novo link (invalida o anterior).

```go
func (cli *Client) GetGroupInfoFromLink(ctx context.Context, code string) (*types.GroupInfo, error)
func (cli *Client) JoinGroupWithLink(ctx context.Context, code string) (types.JID, error)
```
Obtém info e entra em grupo via link de convite.

```go
func (cli *Client) GetGroupInfoFromInvite(
    ctx context.Context,
    jid, inviter types.JID,
    code string,
    expiration int64,
) (*types.GroupInfo, error)
func (cli *Client) JoinGroupWithInvite(
    ctx context.Context,
    jid, inviter types.JID,
    code string,
    expiration int64,
) error
```
Variante com JID e inviter explícitos (de notificação de convite).

```go
func (cli *Client) LinkGroup(ctx context.Context, parent, child types.JID) error
func (cli *Client) UnlinkGroup(ctx context.Context, parent, child types.JID) error
```
Vincula/desvincula subgrupo a uma comunidade.

```go
func (cli *Client) GetSubGroups(ctx context.Context, community types.JID) ([]*types.GroupLinkTarget, error)
func (cli *Client) GetLinkedGroupsParticipants(ctx context.Context, community types.JID) ([]types.JID, error)
```
Lista subgrupos e participantes de uma comunidade.

---

## Privacidade

```go
func (cli *Client) GetPrivacySettings(ctx context.Context) (settings types.PrivacySettings)
func (cli *Client) TryFetchPrivacySettings(ctx context.Context, ignoreCache bool) (*types.PrivacySettings, error)
```
Lê configurações de privacidade.
```go
type PrivacySettings struct {
    GroupAdd        PrivacySetting
    LastSeen        PrivacySetting
    Status          PrivacySetting
    Profile         PrivacySetting
    ReadReceipts    PrivacySetting
    Online          PrivacySetting
    CallAdd         PrivacySetting
    DisappearingTimer time.Duration
}
// PrivacySetting valores: "all", "contacts", "contact_blacklist", "nobody", "match_last_seen"
```

```go
func (cli *Client) SetPrivacySetting(
    ctx context.Context,
    name types.PrivacySettingType,
    value types.PrivacySetting,
) (settings types.PrivacySettings, err error)
```
Atualiza uma configuração de privacidade.

```go
func (cli *Client) SetDefaultDisappearingTimer(ctx context.Context, timer time.Duration) error
```
Define timer padrão de desaparecimento para novos chats.

```go
func (cli *Client) GetStatusPrivacy(ctx context.Context) ([]types.StatusPrivacy, error)
```
Retorna configurações de privacidade de status (quem vê).

---

## Download e Upload de Mídia

### Download

```go
func (cli *Client) Download(ctx context.Context, msg DownloadableMessage) ([]byte, error)
```
Download principal de mídia de uma mensagem (`ImageMessage`, `VideoMessage`, `AudioMessage`, `DocumentMessage`, etc.).

```go
func (cli *Client) DownloadAny(ctx context.Context, msg *waE2E.Message) (data []byte, err error)
```
Tenta download de qualquer tipo de mídia na mensagem.

```go
func (cli *Client) DownloadThumbnail(ctx context.Context, msg DownloadableThumbnail) ([]byte, error)
```
Download de thumbnail.

```go
func (cli *Client) DownloadMediaWithPath(
    ctx context.Context,
    directPath string,
    encFileHash, fileHash, mediaKey []byte,
    fileLength int,
    mediaType MediaType,
    mmsType string,
) ([]byte, error)
```
Download por path direto (útil para re-download de mídia expirada).

```go
func (cli *Client) DownloadHistorySync(
    ctx context.Context,
    notif *waE2E.HistorySyncNotification,
    synchronousStorage bool,
) (*waHistorySync.HistorySync, error)
```
Download de blob de history sync.

```go
func (cli *Client) FetchStickerPack(ctx context.Context, packID string) (*types.StickerPack, error)
```
Busca pacote de stickers por ID.

```go
func GetMediaType(msg DownloadableMessage) MediaType
```
Retorna o `MediaType` de uma mensagem.

### Upload

```go
func (cli *Client) Upload(ctx context.Context, plaintext []byte, appInfo MediaType) (resp UploadResponse, err error)
```
Upload de mídia. Retorna `UploadResponse` com URL, hash, key.
```go
// MediaType: MediaImage, MediaVideo, MediaAudio, MediaDocument, MediaHistory, MediaAppState
type UploadResponse struct {
    URL        string
    DirectPath string
    MediaKey   []byte
    FileSHA256 []byte
    FileEncSHA256 []byte
    FileLength uint64
    Handle     string
}
```

```go
func (cli *Client) UploadReader(
    ctx context.Context,
    plaintext io.Reader,
    tempFile io.ReadWriteSeeker,
    appInfo MediaType,
) (resp UploadResponse, err error)
```
Upload via reader (sem carregar tudo em memória).

```go
func (cli *Client) UploadNewsletter(ctx context.Context, data []byte, appInfo MediaType) (resp UploadResponse, err error)
func (cli *Client) UploadNewsletterReader(ctx context.Context, data io.ReadSeeker, appInfo MediaType) (resp UploadResponse, err error)
```
Upload para newsletters.

```go
func (cli *Client) DeleteMedia(
    ctx context.Context,
    appInfo MediaType,
    directPath string,
    encFileHash []byte,
    encHandle string,
) error
```
Remove mídia do servidor.

---

## Chamadas

```go
func (cli *Client) RejectCall(ctx context.Context, callFrom types.JID, callID string) error
```
Rejeita chamada recebida. Obter `callFrom` e `callID` do evento `*events.CallOffer`.

---

## Newsletters

```go
func (cli *Client) GetNewsletterInfo(ctx context.Context, jid types.JID) (*types.NewsletterMetadata, error)
func (cli *Client) GetNewsletterInfoWithInvite(ctx context.Context, key string) (*types.NewsletterMetadata, error)
```
Busca metadata de newsletter por JID ou link de convite.

```go
func (cli *Client) GetSubscribedNewsletters(ctx context.Context) ([]*types.NewsletterMetadata, error)
```
Lista newsletters em que a conta está inscrita.

```go
func (cli *Client) CreateNewsletter(ctx context.Context, params CreateNewsletterParams) (*types.NewsletterMetadata, error)
```
Cria newsletter.

```go
func (cli *Client) FollowNewsletter(ctx context.Context, jid types.JID) error
func (cli *Client) UnfollowNewsletter(ctx context.Context, jid types.JID) error
func (cli *Client) NewsletterToggleMute(ctx context.Context, jid types.JID, mute bool) error
```
Segue, deixa de seguir ou muta newsletter.

```go
func (cli *Client) NewsletterSubscribeLiveUpdates(ctx context.Context, jid types.JID) (time.Duration, error)
```
Subscreve updates em tempo real de newsletter. Retorna duração da subscrição.

```go
func (cli *Client) NewsletterMarkViewed(ctx context.Context, jid types.JID, serverIDs []types.MessageServerID) error
```
Marca mensagens de newsletter como vistas.

```go
func (cli *Client) NewsletterSendReaction(
    ctx context.Context,
    jid types.JID,
    serverID types.MessageServerID,
    reaction string,
    messageID types.MessageID,
) error
```
Envia reação em mensagem de newsletter.

```go
func (cli *Client) GetNewsletterMessages(
    ctx context.Context,
    jid types.JID,
    params *GetNewsletterMessagesParams,
) ([]*types.NewsletterMessage, error)
func (cli *Client) GetNewsletterMessageUpdates(
    ctx context.Context,
    jid types.JID,
    params *GetNewsletterUpdatesParams,
) ([]*types.NewsletterMessage, error)
```
Busca mensagens e atualizações de newsletter.

```go
func (cli *Client) AcceptTOSNotice(ctx context.Context, noticeID, stage string) error
```
Aceita termos de serviço (required para criar newsletters).

---

## App State

```go
func (cli *Client) FetchAppState(
    ctx context.Context,
    name appstate.WAPatchName,
    fullSync bool,
    onlyIfNotSynced bool,
) error
```
Sincroniza app state (contatos, configurações, etc.).  
`name`: `appstate.WAPatchName("regular")`, `"regular_low"`, `"critical_block"`, `"critical_unblock_low"`, `"nse_vod"`

```go
func (cli *Client) SendAppState(ctx context.Context, patch appstate.PatchInfo) error
```
Envia mutação de app state (ex: arquivar chat, fixar, marcar como não lido).

```go
func (cli *Client) MarkNotDirty(ctx context.Context, cleanType string, ts time.Time) error
```
Marca app state como limpo após sync.

```go
func BuildFatalAppStateExceptionNotification(collections ...appstate.WAPatchName) *waE2E.Message
func BuildAppStateRecoveryRequest(collection appstate.WAPatchName) *waE2E.Message
```
Builders para mensagens de recuperação de app state.

---

## Pairing

```go
func (cli *Client) PairPhone(
    ctx context.Context,
    phone string,
    showPushNotification bool,
    clientType PairClientType,
    clientDisplayName string,
) (string, error)
```
Inicia pairing por código (sem QR). Retorna código de 8 dígitos.
```go
// PairClientType: PairClientChrome, PairClientEdge, PairClientFirefox,
//                 PairClientIE, PairClientOpera, PairClientSafari,
//                 PairClientCustom
code, err := client.PairPhone(ctx, "+5511999999999", true, whatsmeow.PairClientChrome, "Chrome (Linux)")
```

---

## Push Notifications

```go
func (cli *Client) RegisterForPushNotifications(ctx context.Context, pc PushConfig) error
```
Registra para push notifications.
```go
// Implementações de PushConfig:
type FCMPushConfig  struct { Token string }
type APNsPushConfig struct { Token string; VoIPToken string }
type WebPushConfig  struct { Endpoint, Auth, P256DH string }
```

```go
func (cli *Client) GetServerPushNotificationConfig(ctx context.Context) (*waBinary.Node, error)
```
Busca configuração de push do servidor.

---

## Utilitários

```go
func (cli *Client) SetMaxParallelRetryReceiptHandling(n int64)
```
Limita concorrência no processamento de retry receipts.

```go
func ParseDisappearingTimerString(val string) (time.Duration, bool)
```
Converte string (`"off"`, `"24h"`, `"7d"`, `"90d"`) para `time.Duration`.

```go
func GetLatestVersion(ctx context.Context, httpClient *http.Client) (*store.WAVersionContainer, error)
```
Busca versão mais recente do WhatsApp Web (usar para manter `SetWAVersion` atualizado).

```go
func (cli *Client) DangerousInternals() *DangerousInternalClient
```
Expõe métodos internos para uso avançado (testes, extensões). Não usar em produção.

---

## Eventos Principais

| Evento | Descrição |
|--------|-----------|
| `*events.Message` | Nova mensagem recebida |
| `*events.Receipt` | Recibo de entrega/leitura |
| `*events.Presence` | Mudança de presença de contato |
| `*events.ChatPresence` | Digitando/gravando em chat |
| `*events.Connected` | Conexão estabelecida |
| `*events.Disconnected` | Desconectado |
| `*events.LoggedOut` | Sessão encerrada (ban/logout remoto) |
| `*events.QR` | QR code para scan |
| `*events.PairSuccess` | Pairing concluído |
| `*events.GroupInfo` | Mudança em grupo |
| `*events.JoinedGroup` | Entrou em grupo |
| `*events.HistorySync` | Sync de histórico recebido |
| `*events.AppStateSyncComplete` | App state sincronizado |
| `*events.PrivacySettings` | Configurações de privacidade alteradas |
| `*events.CallOffer` | Chamada recebida |
| `*events.CallTerminate` | Chamada encerrada |
| `*events.NewsletterMessageMeta` | Metadata de mensagem de newsletter |
| `*events.StreamError` | Erro de stream (connection replaced, etc.) |
| `*events.KeepAliveTimeout` | Timeout de keepalive |
| `*events.KeepAliveRestored` | Keepalive restaurado |
| `*events.Blocklist` | Blocklist atualizada |
| `*events.PushNameSetting` | Push name alterado |
| `*events.MediaRetry` | Retry de mídia |

---

## Customizações neste Fork

Diferenças em relação ao `tulir/whatsmeow` original:

| Customização | Descrição |
|--------------|-----------|
| `SetProxyAddress(addr, opts)` | Aceita URL string diretamente (tulir exige `SetProxy`) |
| `SetProxyOptions.NoWebsocket` | Controle granular — proxy só na mídia ou só no WebSocket |
| `SetProxyOptions.NoMedia` | Idem |
| Fingerprint Chrome/Windows | `store.BaseClientPayload` pré-configurado para parecer Chrome no Windows |
| `store.SetOSInfo("Windows", ...)` | OS fingerprint aplicado no init |
