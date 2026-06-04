# EOSpro — Sample Game classes (`EOSproExample` module)

A ready-to-use set of **C++ gameplay-framework classes** that demonstrate the EOSpro plugin
end-to-end. They ship **inside the EOSpro plugin** (Runtime module `EOSproExample`), so they
appear in **Project Settings → Maps & Modes** of *any* project that enables EOSpro — no separate
project, no `.uproject`, nothing to copy.

The interactive UI is a fully **native UMG widget** (built in C++), so the sample runs with
**zero content assets**.

## Classes you can select in Project Settings → Maps & Modes

| Setting | Pick this class | Notes |
|---|---|---|
| Game Instance Class | `EOSExampleGameInstance` | Ensures the EOS platform is initialized at startup. |
| Default GameMode | `AEOSExampleGameMode` | Wires up all of the classes below. |
| &nbsp; ↳ Default Pawn Class | `AEOSExamplePawn` | Empty pawn (menu experience). |
| &nbsp; ↳ HUD Class | `AEOSExampleHUD` | Spawns the interactive menu widget on BeginPlay. |
| &nbsp; ↳ Player Controller Class | `AEOSExamplePlayerController` | Shows the mouse cursor. |
| &nbsp; ↳ Game State Class | `AEOSExampleGameState` | Example replicated `PlayersInSession`. |
| &nbsp; ↳ Player State Class | `AEOSExamplePlayerState` | Example replicated `EOSProductUserId`. |

`AEOSExampleGameMode` already sets the Pawn/HUD/Controller/GameState/PlayerState defaults, so
selecting it is enough — but every class is independently selectable too, since each is a native
C++ class. All are `Blueprintable` if you'd rather extend them in Blueprint.

## The interactive widget

`UEOSExampleMenuWidget` (created by `AEOSExampleHUD`) is a native `UUserWidget` whose buttons each
drive a real EOSpro call:

- **Auth** — Login (DevAuth), Login (Account Portal), Login (Device ID), Logout
- **Sessions** — Create / Find / Join (first result) / Destroy
- **Stats** — Ingest +10, Query
- **Leaderboards** — Query ranks
- **Achievements** — Query definitions, Unlock
- **Friends** — Query list

Results stream into an on-screen log (and the Output Log: `LogEOSproExample`, `LogEOSpro`).

## Run it (any project with EOSpro enabled)

1. *Edit → Project Settings → Plugins → EOSpro* — enter your Product/Sandbox/Deployment/Client
   IDs + Client Secret from the Epic Developer Portal. Leave **Auto Initialize** ticked.
2. *Project Settings → Maps & Modes*:
   - **Game Instance Class** → `EOSExampleGameInstance`
   - **Default GameMode** → `AEOSExampleGameMode` (or set it as a map's GameMode Override in
     World Settings)
3. Open any level and **Press Play** → the EOS demo menu appears. Click a **Login** button, then
   drive the other sections.

### Logging in during PIE
- **DevAuth tool**: launch the EOS DevAuth tool (`EOSpro/Source/ThirdParty/EOSSDK/DevAuth/`),
  create a credential, then use **Login (DevAuth)** with the matching `host:port` and name.
- **Account Portal**: opens a browser for Epic sign-in.
- **Device ID**: anonymous; needs a Device ID first (enable auto-create in EOSpro settings).

### Understanding the DevAuth `host:port` field — `127.0.0.1:6547`

**`127.0.0.1:6547` is NOT a browser link.** It is the local address of the **EOS Dev Auth
Tool** — a small desktop app you run on your own PC during development. Do not paste a URL here.

- **`127.0.0.1`** = "localhost" — a special IP that always means *this same computer*. It never
  goes to the internet; it loops back to your PC.
- **`6547`** = a **port number** — a numbered "door" a program listens on. It isn't special; it's
  just the port *you choose* when you start the tool.
- Together: *"the program on **my** machine listening on door **6547**"* — i.e. the Dev Auth Tool.

**Why it exists:** so you don't have to do the full Epic browser sign-in every time you press
Play. You sign in **once** in the tool and reuse it instantly across PIE sessions. Dev/testing only.

**How to use it:**
1. Run `EOS_DevAuthTool.exe` (unzip `…/ThirdParty/EOSSDK/DevAuth/EOS_DevAuthTool-win32-x64-*.zip`).
2. In the tool, enter a **Port** (e.g. `6547`) and click **Login** → a browser opens **once** to
   sign in with your real Epic account.
3. Give that credential a **Name** (e.g. `Context_1`).
4. In the widget's **Identity & Auth** card:
   - **Host field** = `127.0.0.1:6547` → *where* the tool is (the port must match the tool's port).
   - **Name field** = `Context_1` → *which* credential in the tool to use (must match the name).
5. Click **Login (DevAuth)**. The game connects to the tool at `127.0.0.1:6547`, gets a one-time
   code for `Context_1`, and signs you in — no in-game browser, no password typing.

**Notes:**
- If port `6547` is taken or you started the tool on a different port, change the number to match
  (e.g. `127.0.0.1:7777`).
- DevAuth still uses Epic login (EOS_Auth), so it needs Epic Account Services configured, exactly
  like Account Portal.
- If you just want the browser flow instead, skip DevAuth entirely and use **Login (Account
  Portal)** — that opens the Epic sign-in directly and needs no `127.0.0.1` address.

## Package it as a game

1. Do the setup above (project-wide GameInstance + GameMode + a default map).
2. *Project Settings → Maps & Modes → Game Default Map* → your level.
3. *File → Package Project → <platform>*. EOSpro auto-stages the EOS SDK runtime, the XAudio2
   redist, and `EOSBootstrapper.exe`. The packaged build boots straight into the EOS demo menu.

## Reskinning / extending

- **Reskin the UI**: create a Widget Blueprint, *reparent* it to `EOSExampleMenuWidget`, lay it
  out in the designer, then set `AEOSExampleHUD → Menu Widget Class` to your WBP.
- **Extend the framework**: create Blueprint children of any of the C++ classes and select those
  instead.

## Removing the sample from a shipping plugin

Delete the `EOSproExample` entry from `EOSpro.uplugin`'s `Modules` array (and optionally the
`Source/EOSproExample/` folder). Nothing in the core `EOSpro` runtime depends on it.
