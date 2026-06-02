# EOSCore — Building a Working Multiplayer Game Project

Step-by-step recipe for wiring **EOSCore** into a real UE4.27 game project: `UGameInstance`, `AGameMode`, `APlayerController`, HUD/UMG, `USaveGame`, and the OSS purchase bridge.

> This is the **integration cookbook**. For the BP-node reference, see `Documentation/index.html`.

---

## TL;DR — minimum-viable flow

```
GameInstance::Init()       ─► EOS_Platform_Create  (via UEOSCoreSubsystem auto-init)
                              │
PlayerController::BeginPlay  ─► EOSLoginWithPlatform(EpicGames)
                              │   (or DevAuth in PIE, Steam/Apple/Google in shipping)
                              │
                              ▼
              [Bound to OnConnectLoginChanged]
                              │
                              ├─► EOSCreateLobby (Permission=PublicAdvertised, MaxMembers=4, EnableVoice=true)
                              │       │
                              │       └─► EOSJoinRTCRoom via GetLobbyRTCRoomName(LobbyId)
                              │
                              ├─► EOSQueryFriends, EOSQueryPresence — populate HUD
                              │
                              └─► EOSBeginClientSession(ClientServer)   (if AntiCheat enabled)
                                       │
                                       └─► forward bytes via PollOutboundServerMessages each Tick

OnGameEnd / Disconnect      ─► EOSLeaveRTCRoom -> EOSLeaveLobby -> EOSEndClientSession
SaveGame::Save              ─► UGameplayStatics::SaveGameToSlot
```

---

## 1. Project setup checklist

1. **Enable the plugin** — `EOSCore` is already in `PluginCore.uproject`. For a new project add it under `Plugins:`.
2. **Fill credentials** — Project Settings → Plugins → EOSCore → Credentials. (See `Documentation/index.html` Installation section.)
3. **Pick your default login platform** — Project Settings → Plugins → EOSCore → Login Defaults → Default Login Platform. For PC dev pick **Epic Games (Account Portal)**.
4. **(Optional) Anti-Cheat** — drop `base_public.cer` into `Build/NoRedist/`, flip Enable Anti-Cheat.
5. **(Optional) Cross-platform purchase OSS plugins** — enable `OnlineSubsystemSteam` / `OnlineSubsystemGooglePlay` / `OnlineSubsystemIOS` per shipping platform.

---

## 2. The four BP classes you create (or convert from C++)

### 2.1 `BP_EOSGameInstance` — extends `UGameInstance`

**Responsibility**: own EOS init early, wire delegates that survive level transitions, host the player-session-wide state.

**Why a GameInstance and not the GameMode?**
- Survives level transitions (GameMode is destroyed on map change).
- `UEOSCoreSubsystem` is a `UGameInstanceSubsystem` — it auto-creates inside any GameInstance.
- Auth tokens / Connect PUID / RTC room membership all want to outlive map travel.

**BP setup** (Blueprint graph in `BP_EOSGameInstance`):

```
Event Init
  └─► (Auto) UEOSCoreSubsystem::InitializePlatform fires from settings.bAutoInitialize
  └─► Get EOSCore Subsystem (Core proxy)
        │
        └─► Bind Event to OnInitialized → Custom Event "OnEOSReady"
        └─► Bind Event to OnAuthLoginChanged → Custom Event "OnAuthChanged"
        └─► Bind Event to OnConnectLoginChanged → Custom Event "OnConnectChanged"

Event OnEOSReady (Custom)
  └─► (log "EOS initialized")

Event OnConnectChanged (Custom) [ProductUserId param]
  └─► If settings.bAutoBeginPlayerSession:
        └─► EOSMetricsLibrary::BeginPlayerSession(DisplayName, ServerIp="", GameSessionId="")
  └─► If settings.bAntiCheatEnabled:
        └─► EOSAntiCheatLibrary::BeginClientSession(Mode=ClientServer)
  └─► Broadcast custom event "OnPlayerReady" to listeners (PlayerControllers, HUD, etc.)
```

Set in `Project Settings → Maps & Modes → Game Instance Class` = `BP_EOSGameInstance`.

### 2.2 `BP_EOSGameMode` — extends `AGameMode`

**Responsibility**: host-side session/lobby create + register joining players.

```
Event BeginPlay
  ├─► (host check) Is Server?
  │     │
  │     YES:
  │     ├─► EOSCreateLobby
  │     │     - MaxMembers: settings.DefaultLobbyMaxMembers
  │     │     - Permission: settings.DefaultLobbyPermission
  │     │     - BucketId: settings.DefaultLobbyBucketId
  │     │     - Presence: settings.bDefaultSessionPresenceEnabled
  │     │   ─OnSuccess─► (store LobbyId on GameMode)
  │     │             └─► EOSCreateSession  (optional — for matchmaking layer)
  │     │
  │     NO (client):
  │     └─► [wait for player controller to drive join flow via EOSJoinLobby]

Event OnPostLogin (override, fires per joining player)
  └─► If Is Server:
        └─► EOSSessionsLibrary::RegisterPlayer(SessionName, NewPlayer.GetProductUserId())
        └─► (broadcast lobby state change to all clients)

Event EndPlay
  └─► EOSLobbyLibrary::DestroyLobby(LobbyId)
  └─► EOSSessionsLibrary::DestroySession(SessionName)
```

### 2.3 `BP_EOSPlayerController` — extends `APlayerController`

**Responsibility**: auth, voice push-to-talk, per-player UI.

```
Event BeginPlay
  ├─► (client) Is Local Controller? then:
  │     └─► EOSLoginWithPlatform
  │           Platform: settings.DefaultLoginPlatform   (read from settings, NOT hardcoded)
  │           Token:    "" (Account Portal) OR steam/google/apple token from native SDK
  │           Id:       "" (defaults to Account Portal)
  │           DisplayName: PlayerState->GetPlayerName()
  │     ─OnSuccess─► (subsystem now has EpicAccountId + ProductUserId set)
  │             └─► Push HUD widget via Create Widget (BP_EOSHUD)
  │             └─► After RTC room is known, EOSJoinRTCRoom(RoomName, ...)
  │
  └─► Setup Input
        ├─► Bind "PushToTalk" (e.g., V key) → Pressed: TransmitToAllRooms(true), Released: TransmitToAllRooms(false)
        ├─► Bind "ToggleSocialOverlay" (Shift+Tab) → EOSUILibrary::ShowSocialOverlay(LocalUser)
        └─► Bind "OpenStore" (B key) → EOSPlatformPurchaseAvailable → branch
              YES → Push BP_StoreWidget (calls EOSPlatformQueryReceipts + EOSQueryOffers)
              NO  → toast "Store not available on this platform"

Event EndPlay
  ├─► EOSUpdateRTCSending(CurrentRoom, false)  — mute mic before leave
  ├─► EOSLeaveRTCRoom(CurrentRoom)
  └─► EOSAuthLibrary::Logout

Custom Event "ApplyPendingSaveData" [BP_SaveGame param]
  └─► Read save game properties → apply to character/inventory
```

### 2.4 `BP_EOSHUD` — extends `UUserWidget` (UMG)

**Responsibility**: login UI, friends list, voice indicators, store entry point.

Recommended layout (Vertical Box root):

```
WBP_EOSHUD
├── TopBar (HorizontalBox)
│   ├── PlayerDisplayName (TextBlock)        ← bind to Sub->GetLocalEpicAccountId, format via EOSUserInfoLibrary::GetCachedDisplayName
│   ├── ConnectionStatus  (TextBlock)        ← bind to "EOS: " + Sub->IsInitialized
│   ├── ButtonsBox (HorizontalBox)
│   │   ├── BtnFriends  (Button + OnClicked → EOSQueryFriends → push WBP_FriendsList)
│   │   ├── BtnStore    (Button + OnClicked → push WBP_Store)
│   │   ├── BtnVoice    (Button + OnClicked → push WBP_VoiceSettings — see below)
│   │   └── BtnOverlay  (Button + OnClicked → EOSUILibrary::ShowSocialOverlay)
│
├── LobbyPanel (VerticalBox, visible if in lobby)
│   ├── LobbyIdText
│   ├── PlayersList (ListView, item template = WBP_PlayerRow)
│   └── BtnLeave (Button → EOSLeaveLobby)
│
└── BottomBar
    ├── SpeakingIndicator (TextBlock — bind to "🔊" if any participant.bSpeaking)
    └── MuteToggle (CheckBox → EOSUpdateRTCSending)
```

WBP_PlayerRow: shows display name + speaking indicator + mute/block buttons (per-row), calls `EOSUpdateRTCReceiving` and `EOSBlockRTCParticipant`.

WBP_VoiceSettings (in-game runtime menu — separate from the editor debugger):
- Mic combo: `EOSRTCLibrary::GetAudioInputDevices` → `SetAudioInputSettings`. Save the chosen DeviceId to a runtime save (NOT to UEOSCoreSettings — that's editor-config only).
- Speaker combo: same shape with output devices.
- Noise suppression / AEC toggles → `EOSRTCLibrary::SetRTCSetting("DisableNoiseSuppression", "true"/"false")`.

### 2.5 `BP_EOSSaveGame` — extends `USaveGame`

Generated by the `save_game` MCP tool (see ClaudeProMCP Phase 2 docs). Recommended variables:

```cpp
// in BP_EOSSaveGame variables:
FString  LastEpicAccountId;
FString  LastProductUserId;
FString  PreferredMicrophoneDeviceId;
FString  PreferredSpeakerDeviceId;
bool     bMicMuteOnJoin;
int32    MasterVolume;
TArray<FInventoryItem> Inventory;
TArray<FString>        UnlockedAchievementIds;
```

GameInstance hooks:

```
Event OnPlayerReady (from BP_EOSGameInstance)
  └─► LoadGameFromSlot("DefaultSave", 0) → cast to BP_EOSSaveGame
        └─► If exists:
              ├─► Apply Mic/Speaker DeviceId via EOSRTCLibrary::SetAudioInput/OutputSettings
              ├─► Apply bMicMuteOnJoin into a GameInstance-level bool
              └─► Restore inventory by pushing to PlayerState
        └─► If not exists:
              └─► Spawn fresh BP_EOSSaveGame, populate with defaults from EOSCore settings

Event OnApplicationWillDeactivate (FCoreDelegates)
  └─► SaveGameToSlot(CurrentSave, "DefaultSave", 0)
```

---

## 3. Cross-platform purchase flow (OSS bridge)

The OSS bridge auto-routes through whichever store SDK is loaded. **You write the BP graph ONCE.**

```
WBP_Store::OnConstruct
  └─► EOSGetActivePlatformOSS — display the platform name in a text block (debug-handy)
  └─► EOSPlatformPurchaseAvailable — if false, disable buy buttons + show "Store not available"
  └─► EOSPlatformQueryReceipts(bRestoreReceipts=true) → fill OwnedItems list
  └─► EOSQueryOffers (for Epic catalog) OR EOSQueryOffersById([list]) (for Google Play)
       └─► populate offer cards

WBP_OfferCard::OnBuyClicked
  └─► EOSPlatformPurchase(OfferId, "", 1)
        ├─OnSuccess(Receipt)─►
        │     ├─► Add receipt.ReceiptOfferId to OwnedItems
        │     ├─► Grant item to player (e.g., add to inventory)
        │     ├─► EOSPlatformFinalizePurchase(Receipt.TransactionId)
        │     │       ↳ REQUIRED on Google Play within 3 days. No-op on Epic/Steam.
        │     └─► Save the save game
        └─OnFailure(reason)──► toast error
```

**Platform-specific gotchas**:
- **Google Play**: not calling `EOSPlatformFinalizePurchase` within 3 days auto-refunds the purchase. Wire it on success unconditionally.
- **App Store**: same shape, but receipt validation may need a server round-trip (use `EOSPlatformQueryReceipts(bRestoreReceipts=true)` on app launch to restore subscription state).
- **Steam**: `EOSPlatformFinalizePurchase` is a no-op (Steamworks acknowledges automatically); calling it is still safe.
- **Epic Games Store**: equivalent to `EOSRedeemEntitlements` from the EOS Ecom path. Use the OSS bridge for consistency.

---

## 4. Anti-Cheat wiring

Only relevant for multiplayer games shipped with EAC. EOSCore exposes a scaffold — you wire the bytes through your normal network channel.

```
GameMode (server) BeginPlay:
  └─► If is dedicated server and AC enabled:
        ├─► EOS_AntiCheatServer init (NOT exposed via BP yet — call from C++ subsystem extension)
        └─► (per-tick) drain server-side AC messages and broadcast to clients via custom RPC

PlayerController (client) BeginPlay (after Connect login):
  └─► EOSAntiCheatLibrary::BeginClientSession(Mode=ClientServer)
  └─► Bind Tick:
        ├─► For each msg in EOSAntiCheatLibrary::PollOutboundServerMessages:
        │       └─► Server RPC: SendAntiCheatBytesToServer(msg.Data)
        │              ↳ on server: EOS_AntiCheatServer_ReceiveMessageFromClient (C++)
        ├─► If is mesh P2P mode, also poll PollOutboundPeerMessages and forward to each peer

ServerRPC "SendAntiCheatBytesToClient" (server → client):
  └─► On client: EOSAntiCheatLibrary::ReceiveMessageFromServer(bytes)

PlayerController EndPlay:
  └─► EOSAntiCheatLibrary::EndClientSession
```

Drop `base_public.cer` from EOS Portal → Anti-Cheat → Download integrity keys → `Build/NoRedist/base_public.cer`. Flip `bAntiCheatEnabled` in settings.

---

## 5. Common pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Login node returns silent failure in PIE | Missing credentials or wrong default platform | Check Output Log for `LogEOSCore`. For PIE use `Developer` platform + DevAuthTool token. |
| Voice chat join succeeds but no audio | `bDefaultMicMuted` is true OR caller never called `TransmitToAllRooms(true)` | Check Voice Chat settings + push-to-talk binding |
| `GetLobbyRTCRoomName` returns empty | Lobby was created with `bEnableRTCRoom=false` | Either set it true on `EOSCreateLobby` or in Project Settings → Lobby Defaults |
| `EOSPlatformPurchase` returns "No active OnlineSubsystem" | The matching engine plugin isn't enabled | Enable `OnlineSubsystemSteam` / `OnlineSubsystemGooglePlay` etc. in `.uproject`, set DefaultPlatformService in DefaultEngine.ini |
| AC `BeginClientSession` returns false | Cert missing or Connect login not yet complete | `Action_ValidateAntiCheatCert` button in EOSCore settings; gate BeginClientSession on `OnConnectChanged` event |
| `LoadGameFromSlot` returns null | First-time launch — no save yet | Always null-check and spawn a fresh `BP_EOSSaveGame` |

---

## 6. Recommended folder layout in `Content/`

```
Content/
├── Blueprints/
│   ├── Core/
│   │   ├── BP_EOSGameInstance
│   │   ├── BP_EOSGameMode
│   │   └── BP_EOSPlayerController
│   ├── SaveGame/
│   │   └── BP_EOSSaveGame
│   └── Pawns/
│       └── BP_PlayerPawn
├── UI/
│   ├── WBP_EOSHUD
│   ├── WBP_FriendsList
│   ├── WBP_PlayerRow
│   ├── WBP_VoiceSettings
│   ├── WBP_Store
│   └── WBP_OfferCard
└── DataAssets/
    └── DA_GameConfig
```

Set in `Project Settings`:
- `Default GameMode` → `BP_EOSGameMode`
- `Game Instance Class` → `BP_EOSGameInstance`
- `BP_EOSGameMode → Player Controller Class` → `BP_EOSPlayerController`
- `BP_EOSGameMode → HUD Class` → leave default; spawn `WBP_EOSHUD` from `BP_EOSPlayerController::BeginPlay` via `Create Widget` → `Add To Viewport`

---

## 7. End-to-end PIE test checklist

1. Install Epic Games Launcher; sign in once.
2. Unzip `Source/ThirdParty/EOSSDK/DevAuth/EOS_DevAuthTool-win32-x64-1.2.1.zip`, run it, log in, name your dev credential `DevA`.
3. In Project Settings → EOSCore → Login Defaults → Default Login Platform = **DevAuth Tool**. Save.
4. PIE with 2 players (Number of Players = 2, Net Mode = Play As Client).
5. Player 1 logs in with `Token=DevA, Id=127.0.0.1:6547`. Player 2 with `Token=DevB`. (Create DevB in DevAuthTool first.)
6. Player 1 (host) creates the lobby; both see Player Count = 2 in HUD.
7. Both press **V** (push-to-talk) — speaking indicator lights up in HUD.
8. Press **Shift+F1** — EOS Social Overlay appears (browser-rendered).
9. **Window → EOS Voice Chat Debugger** (in the editor) — verify both players appear in the participants list with mic state visible.

If any step fails, the table in Section 5 maps symptom → fix.

---

## 8. Where each BP node lives

| What you need | Where to find it (BP search) | Category |
|---|---|---|
| Log in | `EOS Login With Platform` | EOS \| Auth |
| Get current player | `Get EOS Auth Subsystem → Is Ready` + `Get Local Epic Account Id` | EOS \| Auth \| Subsystem |
| Create lobby | `EOS Create Lobby` | EOS \| Lobby |
| Lobby voice room | `Get Lobby RTC Room Name` | EOS \| Lobby \| RTC |
| Friends | `EOS Query Friends` then `Get Friends List` | EOS \| Friends |
| Voice — global PTT | `Transmit To All Rooms` | EOS \| RTC |
| Voice — mute one player | `EOS Update RTC Receiving` | EOS \| RTC |
| Voice — open debugger | Window menu **EOS Voice Chat Debugger** | (editor-only) |
| Purchase | `EOS Platform Purchase` | EOS \| Platform |
| Receipts | `EOS Platform Query Receipts` + `EOS Platform Finalize Purchase` | EOS \| Platform |
| Save persistent state | `Save Game To Slot` (engine built-in) + your `BP_EOSSaveGame` |  |
| Anti-Cheat begin | `EOS Begin Client Session` | EOS \| AntiCheat |

---

See `Documentation/index.html` for the full 262-node reference, and `Documentation/GameIntegration/GAME_INTEGRATION.html` for an interactive version of this guide.
