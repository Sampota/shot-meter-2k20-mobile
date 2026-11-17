# NBA 2K20 Mobile Native Hook Project — Handoff Notes

## Current state (important)

**No modified or working-hook `.so` is present in the current project files.** The only native library currently available is `libnba2k20_clean_opt.so`, which is the clean/original baseline. It has not been patched with GREEN GIANT!, custom audio playback, or a second meter. Do not install or distribute it as a modded build.

Verified baseline metadata:
- File: `libnba2k20_clean_opt.so`
- Size: 46,350,512 bytes
- Format: ELF64, Android AArch64 shared object
- Build ID: `e8317c4f978861414000a5851641ad5543f7fc6b`
- SHA-256: `f0b31d2050700e3d49ba53b0c6a1ce4896137e07e77b7aeed2066ebbf1de4d09`
- Built for Android 27; NDK r16b

The addresses and structural observations below are from the project's earlier static-analysis work notes. They are leads for continued disassembly and runtime validation, not proof that a hook is implemented or safe.

## Project goals

### 1. GREEN GIANT! release feedback
- Show a GREEN GIANT! visual announcement and play the supplied sound effect together.
- Trigger only for a confirmed green release by the user-controlled player.
- Require an actual ball release as a shot, not merely a green-looking meter state.
- Exclude AI/opponents, passes, and non-green releases.
- Play once per qualifying release.
- Do not wait for the ball to enter the basket; a green shot can miss.
- Do not alter release timing, shot chance, accuracy, thresholds, or any gameplay mechanics.

### 2. Independent second shot meter
- Add a separate visual meter that mirrors the original game's live meter/timing state.
- Preserve the original meter unchanged and visible as before.
- Read actual game state; do not use a separate timer or approximated animation.
- Do not change shooting logic, timing windows, thresholds, or gameplay.

## Main supplied project files

- `libnba2k20_clean_opt.so` — clean native baseline.
- `GAMEDATA.IFF` — game resource container; user identified it as containing Player Indicator, shooting-meter, and stamina assets.
- `Ho Ho Ho Green Giant Sound Effect_1.mp3` — requested sound effect (24,587 bytes).
- `NBA2K14_ShotMeter_V2_1b_Fire_and_Clean.rar` — design/behavior reference only; its binaries are Windows x86 and are not compatible with Android ARM64.
- `NBA 2K27_98.0.2.apk` — file currently present under this name; verify its package/version before treating it as the NBA 2K20 APK.
- Large main/patch OBB archives were previously examined, but are not present in the current `/mnt/data` listing.

## Native component findings

### ShotMeterComponent

| Symbol | Address |
|---|---:|
| `ShotMeterComponent::Init()` | `0x1B25A10` |
| `ShotMeterComponent::Update(float)` | `0x1B25B00` |
| `ShotMeterComponent::GetNumBars(float,bool)` | `0x1B25DBC` |
| `ShotMeterComponent::GetShotChance(AI_NBA_ACTOR*)` | `0x1B25E54` |
| `ShotMeterComponent::Draw()` | `0x1B25EE8` |
| `ShotMeterComponent::ShouldShow(AI_NBA_ACTOR*)` | `0x1B26340` |
| `g_ShotMeterComponent` | `0x3C70B40` |

Earlier structure observation for per-player meter records:
- `+0x08`: actor pointer
- `+0x10`: boolean/state
- `+0x14`: float associated with shot chance
- `+0x18`: another float, exact meaning unresolved

The update path calls `GetShotChance(actor)`, compares values against thresholds, and updates meter state. The draw path uses the player record and rendering helpers. Do not assume the `+0x14` shot-chance float is the actual release-timing progress.

### PlayerIndicatorComponent

| Symbol | Address |
|---|---:|
| `PlayerIndicatorComponent::Init()` | `0x1B22490` |
| `PlayerIndicatorComponent::Update(float)` | `0x1B22554` |
| `PlayerIndicatorComponent::Draw()` | `0x1B22834` |
| `PlayerIndicatorComponent::DoDraw(int)` | `0x1B228FC` |
| `PlayerIndicatorComponent::ShouldShow(AI_NBA_ACTOR*)` | `0x1B22EF8` |
| `g_PlayerIndicatorComponent` | `0x3C70730` |

### ShotFeedbackComponent

| Symbol | Address |
|---|---:|
| `ShotFeedbackComponent::Init()` | `0x1B25248` |
| `ShotFeedbackComponent::Update(float)` | `0x1B252D0` |
| `ShotFeedbackComponent::Draw()` | `0x1B25510` |
| `ShotFeedbackComponent::DoDraw(int)` | `0x1B255D8` |
| `ShotFeedbackComponent::ShouldShow(AI_NBA_ACTOR*)` | `0x1B25750` |
| `g_ShotFeedbackComponent` | `0x3C70838` |

### ShotIndicatorComponent and release timing

Known event-related functions:
- `ShotIndicatorComponent::OnShowShotFeedback(AI_PLAYER*,bool)` at `0x1B3C940`
- `ShotIndicator_OnShowShotFeedback(AI_PLAYER*,bool)` at `0x1B3CA08`

Earlier analysis suggested the component tracks a player, live value, animation/progress, state, release-timing classification/value, and scene/resource. The exact field offsets and meanings need re-verification before implementation. The normal jump-shot update path was observed calling:
- `MVS_AcquirehotReleaseTimingError()`
- `HUR_GetReleaseTimingThresholdValues()`

These are promising sources for a synchronized second meter, but the exact values and update semantics must be confirmed.

## Shot-release and grade investigation

### Candidate shot event
- `EVT_BallShot(...)` at `0x11E68DC` (size previously recorded as 1,768 bytes).
- Handles ball and shooter data, calls `MVS_GetShotData`, stores the shooter pointer at `[ball+0x140]`, calls `SFX_HandleBallShotEvent`, and later calls `JumpshotFeedback_ShotReleased` and `CCH_POE_SUCCESS_GRADE::HandleShotReleased`.
- Strong candidate for an actual ball-shot event hook, but the exact local-player and shot-direction checks remain to be proven.

### User release event
- `EVT_UserShotReleased(...)` at `0x11EAA38` (size previously recorded as 456 bytes).
- Calls profile, shot-data, jump-shot feedback, challenge, and success-grade functions.
- Strong candidate for the user release path, but still needs confirmation of the authoritative green classification and release semantics.

### Feedback grade
- `JumpshotFeedback_ShotReleased(...)` at `0x14D822C` (size previously recorded as 312 bytes).
- Earlier disassembly notes show controller-local checks through `Lockstep_IsControllerLocal(index)` and a feedback-grade calculation using threshold pairs at `0x0202F9A0`.
- `JumpshotFeedback_TranslatePercentToGrade(float,float)` at `0x14D8364`.
- 13 threshold pairs previously recorded at `0x0202F9A0`:
  `(0.10,0.08)`, `(0.18,0.12)`, `(0.25,0.16)`, `(0.32,0.20)`, `(0.39,0.24)`, `(0.46,0.28)`, `(0.53,0.32)`, `(0.60,0.35)`, `(0.67,0.38)`, `(0.74,0.41)`, `(0.81,0.43)`, `(0.88,0.45)`, `(0.95,0.47)`.
- These are grade thresholds, but the exact GREEN label/value mapping is not yet conclusively established. Do not hardcode a guessed timing enum as GREEN.

Other useful functions:
- `Hur_IsShotAJumpshot(AI_PLAYER const*)` at `0x120917C`
- `MVS_GetShotType(actor)` at `0x131BA48`
- `PHY_LaunchShot` at `0x1354D30`
- `Phy_InitShotSimulation` at `0x135B2A4`
- `Lockstep_IsControllerLocal(int)` at `0x0F7B380`

Shot direction toward the basket is not yet conclusively validated. A release event alone is not enough to prove a genuine attempt toward the hoop.

## Audio investigation

Functions identified:
- `AudioMacro_Register(AUDIO_MACROS*)` at `0x0D26464`
- `AudioMacro_Play(...)` at `0x0D27244`
- `AUDIOMACRORESOURCE_HANDLER::Init(...)` at `0x0D278BC`
- `Sound_PlaySoundByHash(string_crc,...)` at `0x0D3AE98`
- Possible stream functions: `VCAudioStream_AddFile()` at `0x1096C44`, `VCAudioStream_AddDataReference()` at `0x1096B74`

The game audio path has not yet been proven to accept and play the supplied MP3 safely. Confirm function prototypes, decoder/stream requirements, ownership, and lifetime before integrating it. Do not pass raw MP3 bytes to `Sound_PlaySoundByHash` without evidence that this is supported.

## Rendering helpers identified

- `CPlayerIconManager::DrawTexture(VCTEXTURE*,matrix const*)` at `0x0FDE7E0`
- `CPlayerIconManager::DrawTextureWithColor(...)` at `0x0FDE7E8`
- `CPlayerIconManager::DrawScene2DIn3D(...)` at `0x0FDE5E4`
- `CPlayerIconManager::SetMaterialEnabled(VCSCENE*,string_crc,bool)` at `0x0FDD5F0`
- `CPlayerIconManager::DrawSubSprite(...)` at `0x0FDEFDC`
- `CPlayerIconManager::DrawSubSpriteWithColor(...)` at `0x0FDF0EC`
- `CPlayerIconManager::DrawSceneRotY(...)` at `0x0FDDB88`
- `VCMATERIAL2::SetRenderState(...)` at `0x1E61EBC`

These are potential rendering building blocks. No second-meter rendering implementation has been built.

## GAMEDATA.IFF and OBB notes

- `GAMEDATA.IFF` was observed to contain two ZLIB-compressed sections; together they decompressed to roughly 35 MB in the earlier investigation.
- The first section included readable shader source, including a frontend shader path. The second section was mostly binary.
- User says this IFF contains Player Indicator, shooting meter, and stamina assets. Exact resource-key-to-asset mapping remains unresolved.
- Several resource-key assignments from earlier notes conflict; treat keys as unverified until matched against component `Init()` calls and the IFF directory.
- OBB header/table parsing was investigated, but the TOC field interpretation was not settled. Do not use previous guessed offsets for extraction. Reconfirm `VCBINFILEDEVICE` parsing and table layout first.
- Prefer selective extraction of only needed resources; do not ask the phone user to unpack multi-gigabyte OBBs unnecessarily.

## NBA2K14 reference archive

The supplied archive contains FIRE and CLEAN variants of a Windows 32-bit x86 NBA 2K14 mod. Its documentation describes a timing/coverage/distance HUD and shot-type support. The DLL/ASI binaries are not compatible with NBA 2K20 Mobile ARM64. Use it only as a visual/behavior reference; do not port its binaries or gameplay-changing behavior.

## Next work sequence

1. Re-open the exact baseline `.so` and re-verify symbols, addresses, and ARM64 call sites.
2. Resolve the authoritative GREEN grade mapping from release timing and grade/localization data.
3. Establish robust local-player, actual-release, shot-type, and basket-direction validation.
4. Confirm the ShotIndicator's live state fields and timing source; design a separate draw path without modifying the original meter.
5. Prove custom MP3 playback through a supported game audio/stream API, including lifetime and cleanup.
6. Implement isolated hooks in a separate working copy; preserve the clean original.
7. Build and test each condition in-game, including green misses, non-green shots, passes, AI shots, and other release types.

## Non-negotiable safeguards

- Never overwrite the clean original library.
- Never label an unmodified copy as a working mod.
- Never claim a hook/build/test succeeded unless it actually did.
- Keep the original meter and shooting mechanics intact.
- Treat unresolved offsets, resource keys, grade values, and function prototypes as hypotheses until re-verified.
