# Card Animation System

A record of how cards are displayed, interacted with, and animated throughout their lifecycle — from sitting in hand to being played and resolved.

## Overall Architecture

### Class Hierarchy

```
Control (Godot)
├── NCard (IPoolable)                     — The card visual itself (card.tscn)
│   └── NCardHighlight                   — Shader-based glow overlay
│
├── NCardHolder (abstract)               — Base container (set card, hover/focus effects)
│   ├── NHandCardHolder                  — Holder in the player's hand
│   ├── NSelectedHandCardHolder          — Holder in the "selected" bar
│   ├── NPreviewCardHolder               — Holder in preview containers
│   └── NGridCardHolder                  — Holder in grid layouts
│
└── NCardPlay (abstract)                 — Orchestrates card play + targeting
    ├── NMouseCardPlay                   — Mouse drag-to-play
    └── NControllerCardPlay              — Controller select-target-play

Node2D
├── NCardFlyVfx                          — Card flying on Bezier curve
├── NCardFlyShuffleVfx                   — Shuffle animation flying card
├── NCardFlyPowerVfx                     — Power card swoosh-to-player
├── NCardTrailVfx                        — Character-specific trail particles
├── NCardTrail (Line2D)                  — Debug/utility trail line
├── NCardSmithVfx                        — Smithing hammer spark VFX
├── NCardTransformVfx                    — Card-to-card transform (shader morph)
├── NCardUpgradeVfx                      — Upgrade sparkle VFX
├── NCardRareGlow / NCardUncommonGlow    — Rarity glow overlays
└── VfxCmd (static)                      — Spawns scene-based VFX at positions

Control (Layout)
├── NPlayerHand                          — Manages all hand card holders
├── NCardPlayQueue                       — Queue of played cards awaiting execution
└── NSelectedHandCardContainer           — Container for selected cards
```

### Runtime Structure

```
NCombatUi
├── Hand (NPlayerHand)
│   ├── CardHolderContainer
│   │   └── NHandCardHolder[] (up to ~10)
│   ├── NEndTurnButton
│   └── NCardPlay (NMouseCardPlay / NControllerCardPlay)
├── PlayQueue (NCardPlayQueue)
│   └── NCard[] (cards played but not yet executed)
├── PlayContainer
└── CombatVfxContainer
```

### Scene File

The card visual itself is defined in a scene template at `scenes/cards/card.tscn`, loaded by the `NCard` class. Cards are pooled (30 pre-created via `NodePool`).

### Key Files

| File | Path | Purpose |
|------|------|---------|
| NCard.cs | `src/Core/Nodes/Cards/NCard.cs` | Card visual: layout, tweens, pooling reset |
| NCardHighlight.cs | `src/Core/Nodes/Cards/NCardHighlight.cs` | Shader glow: show/hide/flash, colors |
| NCardHolder.cs | `src/Core/Nodes/Cards/Holders/NCardHolder.cs` | Base holder: SetCard, hover/focus, pressed signals |
| NHandCardHolder.cs | `src/Core/Nodes/Cards/Holders/NHandCardHolder.cs` | Hand holder: smooth lerp animations, drag, highlight |
| NSelectedHandCardHolder.cs | `src/Core/Nodes/Cards/Holders/NSelectedHandCardHolder.cs` | Selected bar holder: tween-in creation |
| NPreviewCardHolder.cs | `src/Core/Nodes/Cards/Holders/NPreviewCardHolder.cs` | Preview container holder |
| NGridCardHolder.cs | `src/Core/Nodes/Cards/Holders/NGridCardHolder.cs` | Grid layout holder |
| NPlayerHand.cs | `src/Core/Nodes/Combat/NPlayerHand.cs` | Hand layout, holder pressed, card play start |
| NMouseCardPlay.cs | `src/Core/Nodes/Combat/NMouseCardPlay.cs` | Mouse drag-play: play zone, targeting, cancel |
| NControllerCardPlay.cs | `src/Core/Nodes/Combat/NControllerCardPlay.cs` | Controller card play |
| NCardPlay.cs | `src/Core/Nodes/Combat/NCardPlay.cs` | Abstract card play base: TryPlayCard, cancel |
| NCardPlayQueue.cs | `src/Core/Nodes/Combat/NCardPlayQueue.cs` | Played cards queue: entrance/cancel/execution tweens |
| HandPosHelper.cs | `src/Core/Helpers/HandPosHelper.cs` | Precomputed fan layout data |
| NTargetManager.cs | `src/Core/Nodes/Combat/NTargetManager.cs` | Targeting state machine, reticle management |
| NTargetingArrow.cs | `src/Core/Nodes/Combat/NTargetingArrow.cs` | Bezier targeting arrow from card to cursor |
| VfxCmd.cs | `src/Core/Commands/VfxCmd.cs` | Static VFX spawn methods, path constants |
| CardCmd.cs | `src/Core/Commands/CardCmd.cs` | Card command entry points (AutoPlay, Preview...) |
| CardModel.cs | `src/Core/Models/CardModel.cs` | Card model: OnEnqueuePlayVfx, EnqueueManualPlay |
| PileTypeExtensions.cs | `src/Core/Entities/Cards/PileTypeExtensions.cs` | Pile target positions |
| NCardFlyVfx.cs | `src/Core/Nodes/Vfx/NCardFlyVfx.cs` | Bezier card flight with trail |
| NCardFlyPowerVfx.cs | `src/Core/Nodes/Vfx/NCardFlyPowerVfx.cs` | Power card arc flight with screen shake |
| NCardFlyShuffleVfx.cs | `src/Core/Nodes/Vfx/NCardFlyShuffleVfx.cs` | Shuffle Bezier flight |
| NCardTrailVfx.cs | `src/Core/Nodes/Vfx/NCardTrailVfx.cs` | Character-specific particle trail |
| NCardTrail.cs | `src/Core/Nodes/Vfx/NCardTrail.cs` | Line2D debug trail |
| NCardTransformVfx.cs | `src/Core/Nodes/Vfx/NCardTransformVfx.cs` | Shader-based card morph transform |
| NCardUpgradeVfx.cs | `src/Core/Nodes/Vfx/NCardUpgradeVfx.cs` | Upgrade sparkle + scale-in |
| NCardSmithVfx.cs | `src/Core/Nodes/Vfx/NCardSmithVfx.cs` | Smithing hammer sparks + card wobble |
| NCardRareGlow.cs | `src/Core/Nodes/Cards/NCardRareGlow.cs` | Rare card overlay glow |
| NCardUncommonGlow.cs | `src/Core/Nodes/Cards/NCardUncommonGlow.cs` | Uncommon card overlay glow |
| NCombatUi.cs | `src/Core/Nodes/Combat/NCombatUi.cs` | Top-level combat UI orchestrator |
| NCombatUi.tscn | `scenes/screens/combat/combat_ui.tscn` | Scene: card zone layout |

---

## 1. Cards in Hand (Idle)

### Fan Layout — `HandPosHelper.cs`

Precomputed tables for 1–10 card hand sizes:

- `_cardPositionData[handSize-1][cardIndex]` — X,Y offsets relative to hand center
- `_cardAngleData[handSize-1][cardIndex]` — rotation in degrees (-15° to +15°)
- `GetScale(handSize)` — base scale 0.8, further shrinks at high hand sizes (>7)

The card angles and positions form a symmetric arch opening upward.

### Layout Refresh — `NPlayerHand.RefreshLayout()`

Called when hand size changes, a card is removed/added, or hover focus changes. For each active `NHandCardHolder`:

1. Get target position/angle/scale from `HandPosHelper`
2. **Hovered card** (line 408-418): angle=0 (flat), scale=1x, position shifted up by half hitbox height
3. **Neighbor cards** (line 403-405): pushed aside by `Mathf.Lerp(100f, 0f, distanceRatio)` — the closer to focused card, the more they move
4. **Dragged card**: hitbox disabled (line 426)

### Smooth Animation System — `NHandCardHolder`

Each holder runs three independent async Lerp loops, with per-animation `CancellationTokenSource`:

| Animation | Speed | Snap Threshold | Method |
|-----------|-------|----------------|--------|
| Position | 7f | 1px | `AnimPosition()` |
| Angle | 10f | 0.1° | `AnimAngle()` |
| Scale | 8f | 0.002 | `AnimScale()` |

**Cancellation pattern** (line 196-210): When a new target is set, the old `CancellationTokenSource.Cancel()` is called, a new one is created, and a new async loop starts. This ensures animations don't fight each other during rapid layout changes.

**Hitbox re-enable** (line 261-265): Position animation waits until card is within 200px of target before re-enabling the hitbox, preventing accidental clicks during rearrangement.

### Highlight (Glow) — `NCardHighlight.cs`

Shader-based width animation on a glow overlay:

| Method | Effect | Duration |
|--------|--------|----------|
| `AnimShow()` | Shader `width` 0 → 0.075 | 0.5s |
| `AnimHide()` | Shader `width` → 0 | 0.5s |
| `AnimFlash()` | Quick 0 → 0.15 → 0.075 | ~0.35s |

**Glow colors** (static):
- `playableColor`: cyan `(0, 0.957, 0.988)` — card can be played
- `gold`: gold `(1, 0.784, 0)` — special highlight
- `red`: red `(0.83, 0, 0.33)` — card cannot be played

**Glow state update** — `NHandCardHolder.UpdateCard()` (line 302-330): Called on refresh, checks `CanPlay()` and `ShouldGlowRed`/`ShouldGlowGold` to call the appropriate `AnimShow()` or `AnimHide()`.

### Idle Cost Flicker — `NCard.PlayRandomizeCostAnim()`

Random timer intervals trigger a brief visual effect where the energy cost number cycles through random values (or a `?`) before snapping back to the actual cost. Pure visual flair.

### Enter/Exit Hand Animation — `NPlayerHand`

| Method | Effect |
|--------|--------|
| `AnimIn()` (line 801) | Cards fly into hand from below |
| `AnimOut()` (line 809) | Cards fly out downward |
| `AnimDisable()` (line 818) | Cards fade + shrink and become non-interactive |

---

## 2. Card Click / Drag Interaction

### Input Flow

```
Mouse down on card
  ↓
NCardHolder emits "Pressed" signal
  ↓
NPlayerHand.OnHolderPressed()
  ↓ (checks Mode == Play)
  ↓
StartCardPlay()
  ↓
Creates NMouseCardPlay or NControllerCardPlay
  ↓
Starts async play coroutine
```

### Holder Press — `NCardHolder.AnimPress()`

Card briefly scales down to 0.95x (visual press feedback), then on release returns to 1x or begins drag.

### NMouseCardPlay — `NMouseCardPlay.StartAsync()`

The card play coroutine is a state machine running as an async method:

```
StartAsync()
├── StartCardDrag() loop:
│   └── LerpToMouse() each frame
│       └── card.position = lerp(card.position, mouseWorldPos, speed)
│   ├── If card Y < 75% screen height → enter "Play Zone"
│   │   (if CanPlay() == false → show thought bubble, cancel)
│   └── If card Y > 95% screen height → enter "Cancel Zone"
│
├── Play Zone entered:
│   ├── AnimFlash() on card highlight
│   ├── NMouseCardPlay.PlayAnimDragRelease() — card shrinks slightly
│   ├── If single-target card:
│   │   ├── Connect to NTargetManager signals
│   │   └── Wait for target selection (mouse release on valid creature)
│   ├── If multi-target card:
│   │   ├── Show reticles on valid targets (NTargetManager)
│   │   └── Wait for target selection
│   └── If no-target card:
│       └── Proceed directly to TryPlayCard(null)
│
├── Wait for:
│   ├── Target selected → TryPlayCard(target)
│   ├── Card dragged back to Cancel Zone → cancel
│   └── Card dragged to valid zone but mouse released → cancel
│
└── On TryPlayCard:
    ├── On success → card moves to bottom of screen, emit Finished(true)
    └── On failure → emit Finished(false), holder returns to hand
```

### Targeting Arrow — `NTargetingArrow.cs`

A Bezier curve arrow from card position to mouse cursor:

- 19 curve segments
- Arrow head at end
- Head scales up when hovering a valid target (visual confirmation)
- Position updated each frame in `_Process()`
- For controller mode: arrow snaps to nearest valid target

### Play Zone Threshold

The threshold is adaptive: `_dragReleaseThresholdY` (line 49-66 of NMouseCardPlay) adjusts based on the number of cards in hand. With more cards, the play zone triggers at a lower threshold to account for the larger visual footprint.

### Controller Card Play — `NControllerCardPlay`

Alternative to mouse play for controller/joystick input:

1. Left/right on D-pad moves focus between cards in hand
2. Press A to select → card rises up, targeting zone opens
3. Left/right/up/down moves a targeting reticle between creatures
4. Press A again to confirm target → `TryPlayCard(target)`
5. Press B to cancel → return to hand

No drag-to-zone mechanic; selection is purely menu-based.

---

## 3. Card Play (Release) Animation Pipeline

### Phase 1: Enqueue — `CardModel.EnqueueManualPlay()`

```
TryPlayCard(target) succeeds
  ↓
CardModel.EnqueueManualPlay(target)
  ↓ (async)
  └── await OnEnqueuePlayVfx(target)     — card-specific pre-play VFX hook
  └── new PlayCardAction(this, target)    — or Enqueue for auto-play
      └── ActionQueueSynchronizer.RequestEnqueue(action)
```

**`OnEnqueuePlayVfx`** (virtual, in CardModel line 1292): A hook for cards to play a custom VFX immediately when the card is "committed" (before the queue animation). Default returns `Task.CompletedTask`. Overrides:

| Card | VFX |
|------|-----|
| FanOfKnives | Spawns daggers VFX |
| Inflame | Fire burst |
| Hellraiser | Hellfire VFX |
| Pyre | Fire pillar |

### Phase 2: Queue Entrance — `NCardPlayQueue.OnLocalCardPlayed()`

After enqueue, the card leaves the hand and enters the play queue:

1. Card reparented from `NPlayerHand` to `NCardPlayQueue`
2. `TweenCardToQueuePosition()` (line 291-299):
   - Tween to preset queue slot position
   - Scale: 1x → 0.8x
   - Modulate: fade in (alpha transition)
   - Duration: 0.35s
3. Card appears as a "pending" card in the queue, waiting for its `GameAction` to reach the front of the `ActionQueue`

### Phase 3: Execution — `CardCmd.AutoPlay()` → `NCard.AnimCardToPlayPile()`

When the `PlayCardAction` is dequeued and executed by `ActionExecutor`:

```
PlayCardAction.ExecuteAction()
  ↓
card.OnPlayWrapper()  → card effects run
  ↓ (in parallel)
CardCmd.AutoPlay() is called
  ↓
NCard.AnimCardToPlayPile()  (line 925-932)
  └── Tween card to PileType.Play.GetTargetPosition()
  └── Scale: 0.8x (same)
  └── Duration: 0.2s
  └── Card then becomes invisible (moved to "played" zone)
```

### Phase 4: Card Effects Execute

The card's actual effects (damage, draw, apply power, etc.) run through `Commands` (e.g., `CreatureCmd.Damage`), which may trigger further VFX via `VfxCmd`. The card visual is no longer visible at this point — it's been moved off-screen.

### Multi-Card Play Sequences — `NCard.AnimMultiCardPlay()` (line 904-923)

For cards that play multiple copies (e.g., `Wrath` hits 6 times):

- Flash fade out/in
- Scale bounce
- Position rise with `Back` easing
- Each "hit" triggers a visual pulse

### Cancellation Tween — `NCardPlayQueue` (line 280-289)

If a card is removed from the queue before execution (due to cancel/interrupt):

- Tween: fade out (modulate a → 0)
- Tween: offset 30px upward
- Duration: ~0.3s
- Card then freed/destroyed

---

## 4. Card Fly VFX (Flight Between Piles)

### NCardFlyVfx — General Card Flight

Used for cards moving between any two locations (hand → draw pile, hand → discard pile, etc.):

**Creation** (line 51): `NCardFlyVfx.Create(cardVisual, startPos, endPos, character)`

**Bezier flight** (line 108):

1. **Phase 1** (0%–50%): Card rises along a Bezier curve with random control point offset, rotates to face direction of travel, character trail VFX starts following
2. **Phase 2** (50%–100%): Card arcs downward to target, trail fades out at ~85%

**Randomization per flight** (line 73-77):
- Control point offset: random arc height
- Arc direction: random left/right
- Speed: random range
- Acceleration: random range

**Visual details** (line 137-139): Card body scales down and darkens during flight. Card always rotates to face its direction of travel.

### NCardFlyPowerVfx — Power Card Swoosh

Used when a power card is played and flies to the player icon:

- Follows a `Path2D` curve (pre-defined path)
- Card shrinks progressively
- Spin rate increases as it approaches target
- On arrival: screen shake effect (`ScreenShakeCmd`)
- Targets player's `VfxSpawnPosition`

### NCardFlyShuffleVfx — Shuffle Flight

Used during shuffle animations (deck → hand, hand → discard):

- Same Bezier pattern as `NCardFlyVfx`
- Randomization used for shuffle "spread" effect
- Multiple cards can fly simultaneously for visual shuffle

### NCardTrailVfx — Character Trail

Attached to a flying card, follows it each frame:

- Loads character-specific trail scene from `owner.Character.TrailPath`
- Sprites scale down and fade in over 0.5s
- `FadeOut()` (line 55): stops following, fades out, reduces particle emission
- Destroyed 2.5s after fade-out starts

### NCardTrail (Line2D) — Debug Trail

(Line 2D) utility: draws a smooth trail behind the card with point aging (0.8s lifetime), quadratic Bezier interpolation between sparse input points. Used for debugging or occasional visual flair.

---

## 5. Card Transform / Upgrade / Smithing VFX

### NCardTransformVfx — Card Morph

Used when a card transforms into another card (e.g., through an event or relic):

1. Renders card in a `SubViewport` for independent rendering
2. Applies shader morph: brightness pulse → "boing" stretch/squeeze deformation (line 84-106)
3. After morph: flashes relic icons
4. Final: card flies to target pile via `NCardFlyVfx`

**Hand variant** (line 138): Can also play the animation on a card already in hand without removing it — useful for "transforms that replace cards in hand".

### NCardUpgradeVfx — Upgrade Sparkle

Used when a card upgrades (damage +1, block +1, etc.):

- Particle burst at card position
- Card scales from 0 → 1x (scale-in effect)
- Then flies to its target pile via flight VFX
- Gold/yellow color palette

### NCardSmithVfx — Smithing (Forge)

Used in the Regent's Sovereign Blade forging mechanic:

- 3 sequential spark particle bursts, each with screen shake
- Between bursts: card wobbles (rotate back and forth) with elastic easing (line 156-183)
- Final: card settles with sparkle overlay
- Hammer animation visible in background

---

## 6. VfxCmd — Static VFX Spawner

`VfxCmd` is a static utility class that provides the main interface for spawning scene-based visual effects:

| Method | Purpose |
|--------|---------|
| `PlayVfx(Vector2 pos, string path)` | Instantiate VFX scene at world position |
| `PlayOnCreatureCenter(Creature, string path)` | Center VFX on a creature's VfxSpawnPosition |
| `PlayOnSide(CombatSide, string path, CombatState)` | Center on all creatures of a side |
| `PlayOnPlayerPosition(string path, Player player)` | VFX at player position |

**VFX Path Constants** (partial list):
- `vfx/vfx_attack_slash`
- `vfx/vfx_attack_blunt`
- `vfx/vfx_attack_stab`
- `vfx/vfx_block`
- `vfx/vfx_heal`
- `vfx/vfx_buff`
- `vfx/vfx_debuff`
- `vfx/vfx_orb_evoke`

All VFX scenes are loaded relative to a base VFX directory.

---

## 7. Card Model Hooks (VFX Integration Points)

### OnEnqueuePlayVfx — Pre-queue VFX

Called when the card is enqueued for play. Implementations:

**Default** (`CardModel.cs` line 1292):
```csharp
protected virtual async Task OnEnqueuePlayVfx(Creature target)
{
    await Task.CompletedTask; // no VFX
}
```

**Overrides** trigger immediate VFX before the queue animation even starts. Uses `VfxCmd.PlayVfx()` or `VfxCmd.PlayOnCreatureCenter()`.

### CanPlay Visual Feedback

`CardModel.CanPlay()` returns true/false but does not directly trigger VFX. Instead, `NHandCardHolder.UpdateCard()` checks `CanPlay()` to decide highlight color:
- `true` → cyan glow (playable)
- `false` → check `ShouldGlowRed()` → red glow
- golden glow for special states

### OnPlayWrapper — Effect Execution

`CardModel.OnPlayWrapper(choiceContext, target, ...)` is called by `PlayCardAction.ExecuteAction()`. This is the card's effect entry point — it executes the card's effect logic (damage, block, draw, etc.) which in turn calls `Commands`, which may trigger further VFX via `VfxCmd`.

---

## 8. Key Design Patterns

### Fully Async Animation

All card animations use `async Task` with either:
- **Loop-based Lerp**: `while (!snapped)` { `position = Lerp(current, target, speed * delta)`; `await NextFrame();` } — used by NHandCardHolder smooth animations
- **Godot Tween**: `Tween.Create().TweenProperty(node, "position", target, duration)` — used by NCardPlayQueue, NCard.AnimCardToPlayPile

No `_Process` tight loops for animation. Each async coroutine is self-contained.

### Cancellable Lerp (CancellationTokenSource)

Each `NHandCardHolder` maintains separate `CancellationTokenSource[]` for position, angle, and scale animations. When layout changes:
1. Old CTS cancelled → in-progress Lerp loops exit
2. New CTS created → new Lerp loops start toward new targets

Prevents animation conflicts during rapid layout changes (hand size changes, fast hover switching).

### Drag-to-Play (Mouse)

Cards are played by dragging them to a "play zone" (upper 75% of screen), then targeting with the mouse. This is a single continuous gesture — no separate "select card → click target" two-step.

### Targeting State Machine — `NTargetManager`

`NTargetManager` manages three targeting modes:
- `ClickMouseToTarget` — click card, then click target
- `ReleaseMouseToTarget` — drag card to play zone, release on target (most common)
- `Controller` — menu-based selection

Coordinates with `NTargetingArrow` (Bezier arrow), reticle display, and creature hover signals.

### Card Pooling — `NodePool<NCard>`

`NCard` implements `IPoolable`. Pool initialized with `NodePool.Init<NCard>(cardScene, 30)`. Cards are fetched via `NodePool.Get<NCard>()` and returned via `NodePool.Return(card)`.

**Pool reset** — `OnReturnedFromPool()` (line 934-962):
- Resets position, rotation, scale, modulate
- Clears highlight state
- Cleans up children (particles, overlays)
- Removes all tweens
- Resets internal card state references

### Visual/Logical Separation

Card visual lifecycle:
```
Hand (in NPlayerHand) → Play Queue (in NCardPlayQueue) → Played (off-screen or discarded)
```

Card logical lifecycle (parallel):
```
CardModel in hand pile → PlayCardAction in ActionQueue → CardModel in discard/exhaust pile
```

The visual follows the logical with a delay (the queue waiting period). The card is still visually present in the play queue while its `GameAction` is queued. This means the visual is always slightly behind the logical — the player sees the card waiting, then it visually resolves.

---

## 9. Color / Visual Style Constants

| Element | Color |
|---------|-------|
| Playable glow | Cyan `(0, 0.957, 0.988)` |
| Unplayable glow | Red `(0.83, 0, 0.33)` |
| Special glow | Gold `(1, 0.784, 0)` |
| Base card scale | 0.8x |
| Hover card scale | 1.0x |
| Hover angle | 0° (flat) |
| Queue card scale | 0.8x |
| Play zone threshold | 75% screen height (adaptive) |
| Cancel zone threshold | 95% screen height |
| Trail fade out | 0.5s |
| Flight duration | Random range (per VFX class) |
