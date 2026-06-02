# EOSCore — Documentation

UE4.27 plugin wrapping Epic Online Services SDK 1.19.0.7 with a 262-node Blueprint surface plus an editor-side voice chat debugger and a cross-platform OSS purchase bridge.

## Quick links

- **`index.html`** — full reference docs (open in any browser, Bootstrap-styled, with sidebar / topbar / FAQ).
- **Source** — `Plugins/EOSCore/Source/EOSCore/Public/<Category>/` and `Private/<Category>/`.
- **Editor module** — `Plugins/EOSCore/Source/EOSCoreEditor/` (voice chat debugger Slate widget).

## What's in this plugin

| Area | BP nodes | Notes |
|---|---:|---|
| Auth | 10 | EpicGames / Steam / Google / Apple / Device / DevAuth via `EOSLoginWithPlatform` |
| Connect | 10 | Product user mapping, external account linking, device id |
| Sessions | 15 | Create / Find / Join / Update / Invite + invite query |
| Lobby | 18 | Same surface as Sessions + RTC voice room introspection |
| Friends | 10 | Query / Send / Accept / Reject + status |
| Presence | 3 | Set + Query presence |
| Achievements | 5 | Unlock + query defs + query player progress |
| Stats | 4 | Ingest + Query |
| Leaderboards | 6 | Query defs + Query user scores + ranks |
| Ecom | 13 | Offers (all / by-id), Checkout, Ownership, Entitlements, Receipts |
| Storage / TitleStorage | 14 | Player file CRUD + title file read |
| P2P | 2 | Send / Receive packet |
| UI | 3 | EOS Overlay show / hide |
| Metrics | 2 | Begin / End player session |
| CustomInvites | 13 | Send / Accept / Reject + event receiver |
| Reports / Sanctions | 4 | Player behavior reports + sanctions query |
| RTC (voice) | 19 | Join / Leave / Sending / Receiving / Devices + global PTT |
| AntiCheat | 9 | Begin / End session + peer register + message bridge |
| Mods | 9 | Enumerate / Install / Uninstall / Update |
| IntegratedPlatform | 4 | Steam / EGS login status reporting |
| ProgressionSnapshot | 4 | Submit progression snapshots |
| Platform (OSS bridge) | 6 | Cross-platform purchases via engine's IOnlinePurchase |
| Core + Subsystems | 46 | Subsystem accessors + 21 per-category proxy subsystems |

**Total: 262 Blueprint nodes** across 26 categories.

## Five-minute setup

1. Enable the plugin in `<Project>.uproject` (already done for PluginCore).
2. Open **Project Settings → Plugins → EOSCore**. Fill in:
   - `ProductId`, `SandboxId`, `DeploymentId` — from the Epic Developer Portal.
   - `ClientId`, `ClientSecret` — the build's credential pair.
   - `EncryptionKey` — 64 hex chars for player-data-storage encryption.
3. (Optional) **Voice Chat section** — pick default Mic/Speaker DeviceId, AEC toggle, RTC background mode.
4. (Optional) **Anti-Cheat section** — drop `base_public.cer` from EOS Portal into `Build/NoRedist/`, flip `bAntiCheatEnabled = true`.
5. Build via `Build.bat PluginCoreEditor Win64 Development -Project=PluginCore.uproject -WaitMutex -FromMsBuild`.
6. In BP, call `EOSLoginWithPlatform(EpicGames)` — opens the Account Portal browser. Subsystem populates `GetLocalEpicAccountId` / `GetLocalProductUserId` on success.

## PIE / Editor testing

- Run `EOS_DevAuthTool.exe` from `Source/ThirdParty/EOSSDK/DevAuth/`, log in once, name your dev credential `DevUser1`.
- In PIE call `EOSLoginWithPlatform(Developer, Token="DevUser1", Id="127.0.0.1:6547")` — that hits the dev tool on the local port.
- Open the voice debugger: **Window → EOS Voice Chat Debugger** (or `EOSCore.ShowVoiceDebugger` in console). Pick mic/speaker, see live participants once you join a room.

## Cross-platform purchases (OSS bridge)

Use `EOSPlatformPurchase(OfferId, OfferNamespace, Quantity)` — it auto-routes through the engine's `IOnlinePurchase` implementation for whichever OSS is loaded:

- **Win64 Epic Games Store** — `OnlineSubsystemEOS` (already wired via EOSCore).
- **Steam** — enable engine plugin `OnlineSubsystemSteam` + set Steam AppId in `DefaultEngine.ini`.
- **Android** — enable engine plugin `OnlineSubsystemGooglePlay` + set up Play Console product IDs.
- **iOS** — enable engine plugin `OnlineSubsystemIOS` + App Store Connect IAP products.

No BP branching required. Inspect with `EOSGetActivePlatformOSS()` to see which is active.

## Folder structure (after reorganization)

```
Source/EOSCore/
├── Public/
│   ├── Core/             Module, Settings, Subsystem, Types
│   ├── Auth/             AuthLibrary, AuthSubsystem, LoginWithPlatform, VerifyIdToken, …
│   ├── Connect/          ConnectLibrary, CreateConnectUser, CreateDeviceId, …
│   ├── Sessions/         SessionsLibrary, Find/Join/Update/Invite nodes
│   ├── Lobby/            LobbyLibrary + same shape + RTC-room helpers
│   ├── Friends/ Presence/ UserInfo/
│   ├── Achievements/ Stats/ Leaderboards/
│   ├── Ecom/             Offers, Checkout, Ownership, Receipts
│   ├── Storage/ TitleStorage/
│   ├── P2P/ UI/ Metrics/
│   ├── CustomInvites/ Reports/ Sanctions/
│   ├── RTC/              All voice nodes in EOSRTCLibrary.h
│   ├── AntiCheat/
│   ├── Mods/ IntegratedPlatform/ Progression/
│   └── Platform/         OSS bridge for cross-platform purchases
└── Private/              (mirrors Public)
```

## Open issues (heads-up)

- **Continuance token flow is broken from pure Blueprints** — `EOS_ContinuanceToken` is opaque with no `_FromString`. The `EOSCreateConnectUser` / `EOSLinkAccount` BP nodes fail with a clear error when fed a string. Fix needs a wrapper UObject on `UEOSConnectLoginAsync::OnContinuanceRequired`.
- **`FEOSEcomCheckoutEntry::Quantity`** is currently a no-op on the SDK side (1.19.0.7 has no Quantity field in the SDK struct). Submit duplicate entries if you need quantity > 1.

## License

Authored files are `// Copyright Alpha XP 2026. All Rights Reserved.` Epic SDK headers (under `Source/ThirdParty/EOSSDK/Include/`) retain their original `// Copyright Epic Games, Inc.` notice and must not be modified.
