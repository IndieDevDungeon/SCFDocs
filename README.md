# SuperCloudFight — AI System Technical Document

**Project:** SuperCloudFight (SCF)  
**Engine:** Unity (Mirror Networking)  
**Date:** April 03 2026  

---

## Table of Contents

1. Overview 
2. Architecture Summary
3. Core Components
   - 3.1 FlightController — AI Behaviour
   - 3.2 AIManager — Spawning & Coordination
   - 3.3 AIScore — Scoreboard & Lifecycle
   - 3.4 AIRegOnServer — Network Registration
4. AI State Machine
5. Difficulty Profiles
6. Target Assignment System
7. Team Balancing
8. Damage & Death Handling
9. Game Mode Integration
10. Networking Model
11. Roadmap — AI-Only Simulation Mode
    - 11.1 Feature Description
    - 11.2 Feature Description
    - 11.6 Edge Cases & Risk Areas
 
12. File Reference

---

## 1. Overview

SuperCloudFight is a multiplayer aerial combat (Dog fight) arcade style game built on Unity and use Mirror for networking. 
The AI system enables server-authoritative bot planes that participate in matches alongside or instead of human players. 
AI planes share the same `FlightController` component as human-controlled planes, with a flag (`isAI`) switching the control path from player input to an internal state machine.

The system currently supports two live game modes involving AI:

| Mode | Description |
|---|---|
| `PlayerVsAI` | One human player vs. a configurable number of AI opponents |
| `PlayerVsFriend` | Multiple human players; AI bots fill remaining slots to meet the chosen player count |
| `AIVsAI` | AI-Only Simulation | — is described in the Roadmap section of this document.


---

## 2. Architecture Summary

```
NetManager (Mirror NetworkManager)
    |- Game Scene
                ── AIManager (NetworkBehaviour, Server-only logic)
                    ── Spawns AI prefabs via NetworkServer.Spawn()
                    ── Manages target assignment pools
                    ── Runs TeamBalance() every frame on server
            
                  ── [Per AI Plane GameObject]
                        ── FlightController  (flight physics + AI state machine)
                        ── AIScore           (scoreboard data, SyncVars)
                        ── AIRegOnServer     (registers AI into GameManager.remotePlayers)
                        ── ServerDamageable  (HP, damage, death events)
                        ── NetDamageable     (client-side HP mirror)
                        ── NetworkShoot      (server-authoritative shooting)
```

All AI logic runs **server-side only**. `FlightController.aiUpdate()` and `FlightController.FixedUpdate()` both guard with `if (!isServer) return` for the AI code path. Physics forces are applied on the server and replicated to clients via Mirror's standard transform sync.

---

## 3. Core Components

### 3.1 FlightController — AI Behaviour

**File:** `Assets/Scripts/Game/Gameplay/FlightController.cs`  
**Base class:** `NetworkBehaviour`

The `FlightController` is the single component that drives both human and AI planes. The control path is selected by the `isAI` boolean field.

**Key fields relevant to AI:**

| Field | Type | Description |
|---|---|---|
| `isAI` | `bool` | Switches Update/FixedUpdate to AI code path |
| `manualTarget` | `Transform` | The current target the AI is steering toward or attacking |
| `aiState` | `AIState` (enum) | Current state: `Approach`, `Attack`, or `BreakOff` |
| `angleToTarget` | `float` | Angle between AI's heading and target — used for firing checks |
| `aiShootingPoint` | `Transform` | Optional override for the weapon origin transform |
| `aiShootingDistance` | `float` | Max engagement range (falls back to 200 units if unset) |
| `speedControl` | `float` | Throttle value, remapped from `speedControlRange` |
| `baseThrust` | `float` | Randomised per AI at `Awake()` between 100–200 for variance |

**Update loop (AI path):**

```
Update() [ClientCallback]
    └── if isAI -> aiUpdate()
                ── Guard: if !isServer → return
                ── FindClosestPlayer()          — proximity-based target Finding
                ── AIManager.UpdateMyTarget()   — assign and refresh target from pool
                ── State machine transitions    - Ai behavior for new state driven behavior
                ── Shoot() 

FixedUpdate() [ClientCallback] 
    └── if isAI
            ── AddRelativeForce (thrust)
            ── AddRelativeTorque (steering from PID result)
```

**PID Steering:**  
Rotation toward the target is computed using three `PIDController` instances (`px`, `py`, `pz`) operating on Euler angles per axis. The result vector is applied as relative torque each `FixedUpdate`. 
This is the same PID system used for human players, ensuring AI planes handle identically to player planes at the physics level.

---

### 3.2 AIManager — Spawning & Coordination

**File:** `Assets/Scripts/Game/AI/AIManager.cs`  
**Base class:** `NetworkBehaviour`  
**Singleton:** `AIManager.instance`

`AIManager` is the server-side orchestrator. It runs exclusively on the server (`if (!isServer) return` in `Update()`).

**Responsibilities:**

- Spawning AI prefabs at runtime via `AddAI(Team, int index)`
- Maintaining the `spawnedAIs` dictionary (`FlightController → Transform` target mapping)
- Managing the available/borrowed target pool (`AvailableTargets`, `BorrowedTargets`)
- Running `TeamBalance()` to fill vacant slots with bots
- Preventing AI overlap via `FixAIOverlap()`
- `CheckAISameTargetConflict()` -- Resolves same-target conflicts  


**Target pool:**  
`ItemPickup` (powerup pickups transforms) in the scene are collected at `OnStartServer()` into `AvailableTargets`for ai to engage on non combat behavior.  
When an AI needs a non-combat target (e.g. fly toward towards a pickup), it calls `BorrowTarget()`, which moves a transform from `AvailableTargets` to `BorrowedTargets`. Targets are returned via `ReturnBorrowed()` when the AI switches to an enemy target plane or dies.

---

### 3.3 AIScore — Scoreboard logic 

**File:** `Assets/Scripts/Game/AI/AIScore.cs`  
**Base class:** `NetworkBehaviour`

Mirrors the `PlayerScore` component for human players. Manages the AI's scoreboard entry (name, lives, kills, eliminations) using `SyncVar` for replication. Spawns `ScoreboardElement` network objects for each stat column.

**Key SyncVars:**

| SyncVar | Type | Description |
|---|---|---|
| `myId` | `string` | GUID assigned at spawn |
| `myName` | `string` | Randomly generated (e.g. "AI47") |
| `myLives` | `int` | Remaining lives, read by `ServerGameEnder` |
| `myTeamId` | `int` | Drives `myTeam` (Team enum) via hook `OnTeamIdChanged` |
| `myEliminations` | `int` | Elimination count |

On death, `AIManager.OnAIDeath()` checks `myLives`. If lives remain, it waits `GameConfiguration.config.respawnTime` seconds then calls `NetDamageable.ResetHP()`. If no lives remain, the GameObject is deactivated and `GameEvents.playerOutOfLives` is fired, which feeds into `ServerGameEnder.PlayerOutOfGame()`.

---

### 3.4 AIRegOnServer — Network Registration

**File:** `Assets/Scripts/Game/AI/AIRegOnServer.cs`  
**Base class:** `NetworkBehaviour`

Runs on `Start()` to register the AI into `GameManager.remotePlayers` (the same dictionary used for human players). This ensures the `ServerGameEnder` can count AI survivors when determining round end conditions, and that the kill feed system can reference AI entities by ID.

---

## 4. AI State Machine

Each AI plane runs a three-state machine managed in `FlightController.aiUpdate()`.

```
┌─────────────┐     in range + has angle      ┌─────────────┐
│   Approach  │ ─────────────────────────── ▶ │    Attack   │
│  (default)  │                               │             │
└─────────────┘                               └──────┬──────┘
       ▲                                             │
       │                                  too close OR
       │                              engage timer expired
       │                                             │
       │         break-off timer expired     ┌───────▼─────────┐
       └──────────────────────────────────── │    BreakOff     │
                                             └─────────────────┘
```

**State descriptions:**

| State | Steering Target | Shooting | Exit Condition |
|---|---|---|---|
| `Approach` | `manualTarget.position` | Off | In attack range AND has firing angle |
| `Attack` | `manualTarget.position` | Burst fire (see §5) | Too close (`breakOffDistance`) OR `attackEngageDuration` elapsed |
| `BreakOff` | `breakOffPoint` (lateral escape vector) | Off | `breakOffDuration` elapsed |

**Break-off point calculation:**  
A lateral escape vector is computed perpendicular to the AI-to-target direction, offset by `breakOffLateralDistance` (default 120 units) to one random side, plus `breakOffForwardDistance` (80 units) ahead. This creates a realistic evasive banking manoeuvre.

**Separation steering:**  
`GetAISeparationOffset()` samples all other AI planes within a configurable radius (default 30 units) and adds a weighted repulsion vector to the steering target each frame. This prevents AI planes from stacking on the same position.

**Thrust modulation:**  
`baseThrust` is dynamically scaled each frame based on 2D distance to the steering target, ranging from 100 (close) to 250 (far). This causes AI planes to accelerate toward distant targets and slow near them.

---

## 5. Difficulty Profiles

Difficulty is read from `GameModeManager.Instance.SelectedDifficulty` each time a profile is needed. Three profiles are defined as `AIDifficultyProfile` structs:

| Parameter | Easy | Medium (default) | Hard |
|---|---|---|---|
| Attack angle window | ----- | 170°–188° | 174°–186° |
| Attack range multiplier | ×0.8 | ×1.05 | ×1.15 |
| Burst off duration | 2.0–3.5 s | ------- | ------ |
| Shots per burst | 1–4 | 3–6 | 5–8 |
| Shot interval | 0.15–0.28 s | ----- | ------ |
| Attack engage duration | 3–5 s | ----- | ----- |
| Break-off duration | 5–8 s | ----- | ----- |
| Break-off distance | 40 units | 30  | 18  |
| Target switch interval | 60 s | 30 s | 20 s |

Above values are manually tweaked for optimal ai behavior gameplay on each difficulty, 
Note: A fallback profile (identical to Medium) is used if `GameModeManager.Instance` is null, ensuring AI always has valid parameters even in editor test runs.

**Burst fire system:**  
When entering `Attack` state, a burst is rolled: a random shot count, a random inter-shot interval, and a random off-time between bursts. The AI fires individual shots via `NetworkShoot.RequestShoot()`, passing a velocity-compensated direction. Between bursts, `NetworkShoot.OnReleaseShootButton()` is called to release the trigger.

**HP threshold:**  
AI planes stop firing when their own HP drops below specific value , simulating self-preservation humanized behaviour.

---

## 6. Target Assignment System

Target assignment is handled by `AIManager.UpdateMyTarget()`. Each call decides whether the AI should attack an enemy player or fly toward a pickup:

```
UpdateMyTarget(FlightController me)
        ── Roll: attacking = Random(0–10) > 3   (70% chance to attack)
    
        ── If attacking:
            ── Find me.team from players list
            ── Build enemies list (opposite team, distance > 50 units)
            ── me.manualTarget = random enemy player 
    
        ── If not attacking (or no enemies found):
                 If AvailableTargets not empty:
                     ── me.manualTarget = BorrowTarget()   (powerup pickup)
                 Else:
                     ── me.manualTarget = random other AI Player
```

`UpdateMyTarget` is called:
- When 
    -- the AI has no target
    -- the AI gets too close to its current target
    -- `targetSwitchInterval` seconds have elapsed since last assignment
    -- the current target's `ServerDamageable.HitPoints` reaches 0
    -- `CheckAISameTargetConflict` detects two AIs sharing a target at close range

---

## 7. Team Balancing

`TeamBalance()` runs every frame on the server (with a 3-second cooldown for performace optimization). 
It counts current players per team and calls the appropriate spawn method:

**`SpawnRemainingBotsVSAI`** (used in `PlayerVsAI` mode):
- Identifies the human player's team
- Spawns AI bots only on the opposing team

**`SpawnRemainingBotsVSFriends`** (used in `PlayerVsFriend` mode):
- Counts all present teams
- Distributes bots evenly across teams to reach `SelectedPlayerCount`
- Handles odd-number distribution with a +1 remainder pass.

Spawning is gated by `isSpawning` bool to prevent concurrent coroutine overlap. 
Notes: Each bot is added one frame apart via 

---

## 8. Damage & Death Handling

**Damage flow:**
1. Client-side hit detection calls `ServerDamageable.Damage(shooterID, amount, ...)`
2. `CmdDamage` relays to server
3. Server decrements `hitPoints` (SyncVar) and fires `wasHit` / `wasKilled` actions
4. `uWasKilled` UnityEvent triggers `AIManager.OnAIDeath()`

**Death flow:**
```
OnAIDeath(GameObject ai)
        ── If myLives > 1:
            ── Coroutine: wait respawnTime → NetDamageable.ResetHP() → UpdateLives(lives - 1)
        ── If myLives <= 1:
                ── ai.SetActive(false)
                ── GameEvents.playerOutOfLives?.Invoke()
                        ── ServerGameEnder.PlayerOutOfGame()
                                ── Count survivors across all teams
                                        ── If only 1 team remains → GameEnded(winner)

```

`ServerGameEnder` counts both human players (via `GameManager.playerLives`) and AI players (via `AIScore.myLives` + `gameObject.activeInHierarchy`) when evaluating round-end conditions. This means AI deaths directly contribute to match resolution.

---

## 9. Game Mode Integration

`GameModeManager` is a `DontDestroyOnLoad` singleton that persists from the [menu] scene and into the [game] scene. :

```csharp
public GameMode SelectedMode;        // PlayerVsFriend | PlayerVsAI
public MatchType SelectedMatchType;  // TimeSurvival | LastOneStanding | ScoringKills
public DifficultyLevel SelectedDifficulty; // Easy | Medium | Hard
public int SelectedPlayerCount;      // Total players including bots
```

`AIManager.TeamBalance()` reads `GameModeManager.Instance.SelectedMode` to decide which spawn strategy to use. `FlightController.GetDifficultyProfile()` reads `SelectedDifficulty` to configure the AI's combat parameters.

---

## 10. Networking Model

The AI system is fully server-authoritative:

| System | Authority |
|---|---|
| AI state machine (`aiUpdate`) | Server only |
| Physics forces (thrust, torque) | Server only |
| Shooting (`NetworkShoot.RequestShoot`) | Server → replicated |
| HP (`ServerDamageable.hitPoints`) | SyncVar, server writes |
| Scoreboard data (`AIScore`) | SyncVars, server writes |
| AI transform position | Mirror NetworkTransform component |

AI planes do Not have a `connectionToClient` in the traditional sense — they are spawned with the server's `connectionToClient` (the host connection). This is consistent with how Mirror handles server-owned objects.

---

## 11. Roadmap — AI-Only Simulation Mode:

### 11.1 Feature Description

The **AI-Only Simulation Mode** is a new game mode in which no human player is required. Pressing Play immediately starts a fully automated match: the host session is created programmatically From Menu to the Game, the game scene is loaded, and two teams of AI planes fight to completion without any player input. The session runs, resolves, and can loop or return to menu automatically.

This is sometimes referred to internally as *"simulation mode"* or *"AI taking over the player."*

---


### 11.2 Session Flow Diagram

```
[User presses "Simulation" button in menu]
                    │
                    ▼
        SimulationAutoLauncher.LaunchSimulation()
                    │
          ┌─────────┴────────────┐
          │  Set GameModeManager │
          │  SelectedMode=AIVsAI │
          │  SelectedCount=8     │
          └─────────┬────────────┘
                    │
          NetworkManager.StartHost()
                    │
          NetManager.ServerChangeScene("game")
                    │
          [Game scene loads additively]
                    │
          AIManager.OnStartServer()
                    │
          TeamBalance() fires after 3s cooldown
                    │
          SpawnRemainingBotsAIVsAI(0, 8)
                    │
          ┌─────────┴──────────┐
          │  4× AddAI(Team1)   │
          │  4× AddAI(Team2)   │
          └─────────┬──────────┘
                    │
          [8 AI planes spawn and begin flying]
                    │
          FlightController.aiUpdate() runs per plane
                    │
          [Combat proceeds: Approach → Attack → BreakOff]
                    │
          [Planes die, AIScore.myLives decrements]
                    │
          ServerGameEnder.PlayerOutOfGame()
                    │
          [Last team standing wins]
                    │
          GameEvents.endRound(winner) → RpcGameOver()
                    │
          [End screen shown / return to menu]
```

---

### 11.3 Edge Cases & Risk Areas

To be Addressed


## 12. File References

| File | Namespace / Class | Role |
|---|---|---|
| `Assets/Scripts/Game/Gameplay/FlightController.cs` | `SCF.Game.Gameplay.FlightController` | Flight physics + AI state machine |
| `Assets/Scripts/Game/AI/AIManager.cs` | `AIManager` | AI spawning, targeting, team balance |
| `Assets/Scripts/Game/AI/AIScore.cs` | `AIScore` | AI scoreboard data & respawn lifecycle |
| `Assets/Scripts/Game/AI/AIRegOnServer.cs` | `AIRegOnServer` | Registers AI into `GameManager.remotePlayers` |
| `Assets/Scripts/Game/System/GameManager.cs` | `SCF.GameManager` | Static game state, player dictionaries |
| `Assets/Scripts/Game/System/GameEvents.cs` | `SCF.GameEvents` | Global event bus |
| `Assets/Scripts/Game/System/PlayLevelLoader.cs` | `SCF.Game.System.PlayLevelLoader` | Additive scene loading for game level |
| `Assets/Scripts/Game/Networking/ServerDamageable.cs` | `SCF.Game.Networking.ServerDamageable` | Server-authoritative HP & damage |
| `Assets/Scripts/Game/Networking/ServerGameEnder.cs` | `SCF.Game.Networking.ServerGameEnder` | Round end detection & winner broadcasts
| `Assets/Scripts/Game/Networking/NetworkShoot.cs` | `SCF.Game.Networking.NetworkShoot` | Server-authoritative shooting |
| `Assets/Scripts/Networking/NetManager.cs` | `NetManager` | Mirror NetworkManager extension |
| `Assets/Scripts/Networking/Ready.cs` | `SCF.Networking.Ready` | Lobby ready-up and scene change trigger |
| `Assets/Sohail/Scripts/GameModeManager.cs` | `GameModeManager` | Mode, difficulty, player count config |

---

*Document prepared from SuperCloudFight Assets. All line references correspond to the codebase state as of March 2026.*
