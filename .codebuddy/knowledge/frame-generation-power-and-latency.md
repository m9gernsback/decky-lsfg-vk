# LSFG-VK Present Modes and Frame Caps: Power, Latency, and Cross-API Behavior

Scope: Steam Deck (60 Hz panel), `multiplier = 2`, i.e. 30 base frames → 60 output frames.

Sources verified by direct source read, not inference:

- **lsfg-vk layer** — `/mnt/e/GitHub/lsfg-vk`, branch `develop`, HEAD `8b0da26`, plus git history for the removed present-mode flag
- **vkd3d-proton** — `libs/vkd3d/swapchain.c` at upstream master
- **this plugin** — `py_modules/`, `src/`, `shared_config.py`

## 0. Terminology correction (important)

The plugin's **"Base FPS Cap" (FrameCap) slider is not an lsfg-vk feature.** It does exactly one thing: writes a single line into `~/lsfg`

```bash
export DXVK_FRAME_RATE=<n>
```

See `py_modules/lsfg_vk/config_schema_generated.py:101-103`; the env-var name mapping is hardcoded in `scripts/generate_python_boilerplate.py:33`.

`DXVK_FRAME_RATE` is read by DXVK only, which implements D3D9/10/11. **For DX12, native Vulkan, and OpenGL that line is inert**, so a config that looks like "cap set" silently degenerates into "no cap".

Note the env var and the config option have *different* reach: `DXVK_FRAME_RATE` (env) does not affect DX12, but `dxvk.maxFrameRate` (config) does, via a cross-project interface. See section 4 — this distinction is easy to get wrong.

**Present Mode (FIFO / Mailbox)**, by contrast, is the `experimental_present_mode` field in `conf.toml` (`shared_config.py:70-76`), applied by the lsfg-vk layer to the real swapchain.

The two live at completely different levels: one is a pre-render frame limiter, the other is a present-stage queue policy.

## 0.5 The layer's present sequence (source of truth for everything below)

Per game frame the layer issues **`multiplier` presents on the game's own swapchain**. Current `develop` (`lsfg-vk-layer/src/swapchain.cpp:205-301`) and the shipped fp16 build (`src/context.cpp:169-232`, function starts at `:125`) are structurally identical here:

1. Blit the game's frame into a backend source image (`swapchain.cpp:172-203`)
2. Loop `multiplier - 1` times — per interpolated frame: `vkAcquireNextImageKHR` (`swapchain.cpp:212`, `UINT64_MAX` timeout) → blit interpolated result into the acquired image → `vkQueuePresentKHR` (`swapchain.cpp:281`)
3. **Then** present the original, real frame (`swapchain.cpp:289-301`)

Two consequences, both load-bearing:

- **The interpolated frame is presented FIRST, the real frame SECOND.**
- **The presents are separated by microseconds, not milliseconds.** They are serialized on the GPU via semaphores (`lastPCS.second`, `swapchain.cpp:294`), not spaced by any timer. Nothing in this code paces them.

FIFO's blocking behavior is what supplies the missing pacing. That is the single most important fact in this document.

## 0.6 Upstream deleted the Mailbox option

`git log -S "e_present"` gives the flag's full lifetime:

- Added 2025-07-18 in `8b29b95` ("add experimental flags for present mode and fps limit"), replacing a hardcoded `VK_PRESENT_MODE_FIFO_KHR` — the comment it deleted read `// enforce vsync`
- Survived through `v1.0.0` (`7113d7d`)
- **Removed entirely** 2025-12-19 in `e0fac3e` ("refactor(cleanup): implement basic/none frame pacing")

Current `develop` contains **zero** occurrences of "mailbox" in any C++ source. FIFO is now hardcoded in two places: at creation (`swapchain.cpp:64`) and re-forced on *every* present (`swapchain.cpp:155`) to defend against `VkSwapchainPresentModeInfoEXT` overrides from gamescope or the game.

The replacement abstraction is the tell:

```cpp
enum class Pacing : uint8_t {
    /// do not perform any pacing (vsync+novrr)
    None
};
```

A single-valued enum (`lsfg-vk-common/include/lsfg-vk-common/configuration/config.hpp:23-26`), switched over with one case. The author reframed the area from "which present mode" to "which **pacing** strategy" — correctly identifying pacing, not tearing, as the real problem — then shipped exactly one option: FIFO.

Mailbox was not deprecated or hidden. It was concluded to be wrong and cut, by the author of the frame-generation layer, from the inside.

**Version note:** the plugin ships `xXJSONDeruloXx/lsfg-vk` at tag `fp16-test-2` (`package.json:52-53`), which predates the `e0fac3e` pacing refactor. That is why the plugin's UI still exposes a present-mode toggle at all — the option still exists in the build being shipped, it just should not be used.

## 1. Power consumption across the five configurations

### 1.0 Prerequisite: when does FG save power at all? The `I < R` condition

All five configurations below assume FG is worth running. That assumption is conditional, and the condition is worth stating before comparing tuning options — the table that follows compares configurations *against each other*, not FG against no-FG.

Let `R` = energy to render one frame, `I` = energy to generate one interpolated frame. At multiplier 2 targeting 60 output:

- **Native 60 FPS:** `60R`
- **FG 30→60:** `30R + 30I`

FG therefore saves power **iff `I < R`**. This is the whole basis of FG as a battery-saving technique, and it is not automatic.

**Why `I < R` usually holds:** `I` is **scene-independent** — it depends on output resolution and model choice, not on geometry count, light count, or shader complexity. Per base frame the layer pays a fixed set of work (from `swapchain.cpp`): blit game frame → backend source image (`:172`), optical flow + blend (`scheduleFrames`, `:140`), blit result → acquired swapchain image (`:226`). Two full-resolution image copies plus the flow model, every frame, regardless of content. On the Deck those blits also consume LPDDR5 bandwidth, which is shared with the CPU and is itself a real power draw.

`R`, by contrast, scales with how demanding the game is. So:

- **Demanding game** (large `R`) → `I ≪ R` → FG saves substantial power. The intended use case.
- **Light / old game** (small `R`) → `I ≈ R` → saving evaporates; may go either way.
- **Game already hitting the refresh cap** → worst case on every axis. See section 8.4.

**Two factors cut the other way and should not be forgotten:**

1. Halving base render rate also **halves CPU work** (simulation, culling, draw submission). On the Deck's shared TDP budget that is a genuine saving, independent of `R` vs `I`.
2. The plugin's defaults are not the cheapest available: `flow_scale` defaults to **0.8** (`shared_config.py:49`), and `performance_mode` defaults to **False** (`shared_config.py:57`) — i.e. the *heavier* model is on by default, despite its own description recommending the lighter one "for most games". Lowering `flow_scale` and enabling `performance_mode` both reduce `I`, shifting a marginal case toward favourable.

**Practical implication:** if a game is light enough that the Deck renders it comfortably at the refresh cap, do not assume FG helps power — measure. If it is heavy enough that base FPS is well below refresh, `I < R` holds comfortably and the tuning question below is the relevant one.

### 1.1 Configuration comparison

The table and reasons below compare the five configurations **against each other**, all with FG enabled at multiplier 2. They do not address the FG-vs-no-FG question of section 1.0.

| # | Base cap | Present | Relative power | Dominant reason |
|---|---|---|---|---|
| 1 | in-game 30 | FIFO | **baseline (lowest)** | deadline pacing + FIFO does frame spacing for free |
| 3 | DXVK 30 | FIFO | +0–2% | slight CPU run-ahead (reason d) |
| 5 | uncapped | FIFO | +2–8% | high V/f residency (reason a) + CPU run-ahead (reason c) |
| 2 | in-game 30 | Mailbox | +5–12% | discarded interpolated frames (reason b) |
| 4 | DXVK 30 | Mailbox | +5–12% | reason b + reason d |

### Reason a: DVFS voltage scaling (dominant term)

The APU picks GFX clock from activity residency over a short sampling window. Dynamic power is `P ≈ C·V²·f`, and within the DVFS range `V` rises roughly linearly with `f`, so **power scales close to `f³` for the same total work**.

This is why *how* you reach 30 FPS matters more than the fact that you reached it:

- **Explicit limiter (cases 1–4):** the limiter sleeps the frame loop until a wall-clock deadline. The GPU gets one long contiguous idle gap per frame. The governor sees moderate busy% → picks a mid clock → drops to a lower voltage rail. Same frames, much lower `V²f`.
- **Backpressure only (case 5):** the game renders flat-out until the swapchain fills, then blocks in present. Same average work, but *bunched*: a burst at boost clock followed by a stall. The governor's window sees high instantaneous activity → holds high clocks → identical work executed at a higher V/f point.

On this SoC, "race to idle then wait" is strictly worse than "pace to deadline," because idle gaps do not recover the voltage penalty already paid.

### Reason b: the Mailbox penalty (three distinct mechanisms)

Source-verified against the layer. This is the key asymmetry and it is specific to this configuration. An earlier revision of this document described it as "some frames get dropped" — that was right in conclusion but materially understated the cause. There are three separate mechanisms.

#### b1 — Mailbox discards *specifically the interpolated frame*

Mailbox is spec'd as an **internal single-entry queue**: if a new present arrives while an entry is pending, the new request **replaces** it and the old image becomes immediately available for reuse.

Combine that with the present order from section 0.5. Within one vblank interval the layer presents interpolated, then real. **The real frame replaces the interpolated one in the mailbox slot.** The panel displays the real frame. The interpolated frame is computed in full — optical flow, blend, blit — and then thrown away.

This is worse than symmetric frame dropping: the loss is *systematically biased*. The discarded frame is always the expensive generated one, never the cheap real one. In the limit you pay 100% of the frame-generation cost and display 30 FPS of purely real frames. You have bought nothing.

Under FIFO this cannot happen. FIFO's queue is ordered, so both presents are displayed one per vblank — and its blocking behavior spaces them exactly 16.67 ms apart for free.

#### b2 — Mailbox removes the layer's only throttle

This is the actual power mechanism, and it is sharper than a DVFS residency effect.

In the fp16 build the plugin ships, `LsContext::present` contains **no fence wait at all** — the only `UINT64_MAX` in the entire function is the `vkAcquireNextImageKHR` at `src/context.cpp:173`. Everything else is GPU-side semaphore serialization, which does not block the calling thread.

So `vkAcquireNextImageKHR` (plus `vkQueuePresentKHR` once images run out) is the *sole* regulator of how fast this loop runs.

- **Under FIFO:** acquired images stay owned by the presentation engine until their vblank. With ~4-5 swapchain images (`swapchain.cpp:60`, `createInfo.minImageCount += profile.multiplier`), acquire blocks once the queue fills. That block propagates back into the game's frame loop. **This is what makes case 5 hold 30 base FPS at all.**
- **Under Mailbox:** the replaced image becomes reusable *immediately*, per spec. Acquire returns without blocking. **The throttle is gone.**

Consequence: `scheduleFrames` (`swapchain.cpp:140`) is invoked as fast as the game can present, unbounded. Interpolation work queues onto the GPU faster than the display consumes it, and the GPU sits saturated with work that will be discarded per b1. That — not a subtle voltage-residency effect — is why Mailbox costs power.

Corroboration: upstream `develop` **added** an explicit `renderFence->wait(vk, 150ms)` (`swapchain.cpp:164`) that the shipped fp16 build lacks. Upstream evidently recognized that relying on acquire alone for throttling was fragile.

#### b3 — Unpaced presents cause beat-frequency judder

With the throttle gone, nothing ties the layer's present cadence to the 60 Hz vblank. Present pairs drift against the vblank clock. Most land in the same interval (b1), but some straddle a boundary and both get shown.

Effective output is therefore neither a clean 30 nor 60 — it wanders between them at the drift rate, reading as slow periodic hitching on a multi-second cycle. This is more objectionable than a steady 30, because the visual system adapts to constant cadence but not to a varying one.

It also degrades quality independently of dropping. Interpolation assumes the generated frame sits at a fixed temporal midpoint between N and N+1; unstable input frametimes place it at the wrong point in time, worsening ghosting and warping.

#### Why "no surplus to exploit" is the right intuition

Mailbox pays off when output rate > refresh rate, or when FIFO backpressure is collapsing base FPS. At multiplier 2 on a 60 Hz panel, output rate *equals* refresh — no surplus exists. But the sharper statement is: **Mailbox does not merely fail to help, it destroys the FIFO backpressure this layer's design depends on for both throttling and pacing.**

#### Verification

With Mailbox, watch presented FPS. If b1 dominates you will see ~30 presented alongside high GPU power — full cost, no benefit. That is the direct, unambiguous fingerprint. Paired spikes in the frametime graph indicate the same thing.

### Reason b-deck: gamescope makes Mailbox pointless on this device

In gamemode, gamescope is the compositor and always presents to the panel at the set refresh rate. This plugin's `enable_wsi` defaults to `False` (`shared_config.py:126-132`), exporting `ENABLE_GAMESCOPE_WSI=0`, so the game presents to a plain Wayland surface rather than through the gamescope WSI layer.

Either way, "Mailbox" is emulated by gamescope's compositing rather than by display hardware. Gamescope will not show the interpolated frame any sooner than the next composite, so Mailbox's one theoretical benefit — lower latency from skipping a queued frame — is largely absorbed by the compositor. Meanwhile b1/b2/b3 land in full, because they occur upstream of gamescope, inside the layer.

**Net: Mailbox has essentially no upside on a Steam Deck in gamemode.** See the latency revision in section 2.

### Reason c: CPU run-ahead and the shared TDP budget

The Deck shares one power budget between CPU and GPU. In case 5 the simulation loop is never throttled — it runs ahead doing culling, draw-call setup, and command building for frames that then sit in a queue. That CPU work is real power draw, and CPU boost residency steals headroom from the GPU when TDP-limited. An explicit limiter sleeps the whole loop, so CPU and GPU idle together.

### Reason d: where the limiter sleeps (the case 1 vs 3 gap)

Both are deadline limiters, but they hook different places:

- **In-game limiter (case 1):** sleeps in the engine's main loop. Simulation, submission, and present are all paced together.
- **DXVK limiter (case 3):** per the vkd3d-proton maintainer discussion in issue #1998, DXVK's limiter *"runs on the same thread that handles present waits and feeds back into app logic that way, rather than blocking DXGI present directly."* So the game can run slightly ahead on CPU before the throttle propagates back — marginally more CPU power and latency.

**One inversion worth knowing:** some in-game limiters **busy-wait** to hit their deadline instead of sleeping. When that happens, case 1 loses to case 3 on power, because a spinning CPU core keeps the whole package boosted. If case 1 measures *worse* than case 3, a spinning in-game limiter is the prime suspect.

## 2. Input latency

### Components of the chain

1. **Sample→render:** ~1 base frame ≈ 33 ms at 30 FPS. Identical in all five cases.
2. **Frame-generation tax: +16.7 ms, unavoidable in all five cases.** Interpolation is not extrapolation: the intermediate frame lies *between* N and N+1, so the layer cannot emit it until N+1 already exists. Real frame N+1 must therefore be held back by one full output interval (16.67 ms at 60 Hz) so the interpolated frame can occupy the slot ahead of it. This is inherent to FG; no setting removes it.
3. **Swapchain queue depth — the variable term.** FIFO queues presents in order, so a *full* queue adds 1–2 display intervals of staleness. Mailbox discards instead of queuing, so latency cannot accumulate. This is the one axis where Mailbox has any theoretical claim — but see the revision below.
4. **Sampling freshness.** A deadline limiter that sleeps starts the frame *right before* its deadline, so input is sampled as late as possible. Backpressure-limiting is the opposite: the frame was rendered long ago and has been aging in a queue.

### Revision: the Mailbox latency advantage is near zero on a Steam Deck

The −5 to −8 ms figure below assumes **direct-to-display presentation**. Under gamescope it does not hold (reason b-deck): the compositor will not show the interpolated frame sooner than its next composite, so most of the theoretical gain is absorbed. Treat the Mailbox rows as **≈0 to −3 ms in gamemode**, and note that b3's judder is a *latency-variance* cost that partially cancels even that.

This weakens the case 1 vs case 2 trade-off to the point where it is no longer a real trade-off. See the revised structural conclusion.

### Ranking, best → worst

| # | Config | Δ vs case 1 | Why |
|---|---|---|---|
| 2 | in-game 30 + Mailbox | **−5 to −8 ms direct; ≈0 to −3 ms under gamescope** | latest sampling, no queue accumulation (component 3) |
| 1 | in-game 30 + FIFO | **baseline (~55 ms)** | latest sampling, shallow queue |
| 4 | DXVK 30 + Mailbox | +0 to +5 ms | no queue, but CPU run-ahead (reason d) |
| 3 | DXVK 30 + FIFO | +5 to +10 ms | run-ahead + queued presents |
| 5 | uncapped + FIFO | **+20 to +30 ms** | queue held permanently full |

**On case 5 specifically:** the base *is* still held to 30 — FIFO admits 60 presents/s, the layer emits 2 per base frame, so backpressure propagates back through `vkAcquireNextImageKHR` (see b2) to the game. But the mechanism is "render ahead, then stall in acquire/present," which means every frame you see was rendered 2–3 output intervals ago. This is the classic uncapped-vsync latency penalty, and FG compounds it because the layer's own present queue sits in series with the swapchain queue.

### Structural conclusion (revised)

Compare the two orderings. **Case 5 is strictly dominated** — worse on both power and latency, no compensating benefit.

The earlier revision of this document treated case 1 vs case 2 as a genuine trade-off (Mailbox: worse power, better latency). **Source review invalidates that.** Mailbox's latency edge is near zero under gamescope (b-deck), while its costs are three independent mechanisms (b1 wasted work, b2 lost throttle, b3 judder) rather than one. Mailbox is therefore *also* close to strictly dominated on this hardware.

**Case 1 is the recommendation, and there is no configuration at 60 Hz / multiplier 2 where Mailbox wins.** Upstream reached the same conclusion and deleted the option (section 0.6).

### 2.1 The underlying rule, and why "limiter off" is not one setting

The case 1 vs case 5 gap is often misread as "in-game limiter on beats in-game limiter off." It is not. The general rule is:

> **Exactly one deadline limiter, placed as close to the input-sampling point as possible.** Zero deadline limiters (backpressure only) is the worst case. Two active limiters at the same target is frame-pacing jitter.

Case 5 is bad not because a checkbox is off, but because **no deadline limiter exists anywhere in the chain** — the only thing holding base FPS at 30 is FIFO backpressure blocking in `vkAcquireNextImageKHR` (section b2). "Render ahead, then stall" means every displayed frame was sampled 2–3 output intervals ago.

**This is why the Deck conclusion does not generalize to "always enable the in-game limiter."** On the Deck the in-game limiter is usually the *only* deadline limiter available, so case 1 and "limiter on" happen to coincide. Change the platform and they separate.

#### Worked counter-example: desktop, driver-side cap, no FG

Config: RTX 5070 Ti / 9700X, Nvidia App driver-level cap at 60, frame generation **off**.

Here the driver cap **is** an active deadline limiter — it sleeps the pipeline to a wall-clock target, and with Reflex it also performs render-queue reduction. So switching the in-game limiter off does **not** produce case 5; the chain still contains exactly one proper limiter. In this document's taxonomy the config is a **case 1/3 analogue, not case 5.**

Correct in-game limiter setting: **off**, or a few FPS *above* the driver cap (63–65) purely as a menu/loading-screen fallback. Setting it *at* 60 makes two limiters contend; setting it *below* 60 makes it the binding one, which silently discards the driver limiter's Reflex-based pacing.

Companion settings for a 60 cap with no FG:

- **Reflex: On** (not "On + Boost") — cooperates with the driver cap; this is the dominant latency term.
- Driver **Low Latency Mode: Off** when Reflex is available in-game; **Ultra** only for titles without Reflex.
- **With G-Sync/VRR:** G-Sync on, V-Sync on in the driver, cap comfortably under refresh — lowest-latency tear-free configuration.
- **Without VRR:** V-Sync off; consider 58–59 rather than exactly 60 to avoid brushing the refresh ceiling.

#### ⚠ What does not transfer

Every latency figure in section 2 assumes **FG is active**, so the unavoidable ~16.7 ms interpolation hold-back (component 2) is baked into every row, including the ~55 ms case 1 baseline. A no-FG desktop config does not pay that term at all. **Only the limiter-placement logic transfers across platforms; the absolute magnitudes do not.**

## 3. Recommended configuration

In-game limiter → 30; FrameCap slider → Off; Present Mode → FIFO.

If the game's own limiter is unreliable (oscillates 28–31, or is tied to its vsync setting) or absent, invert it: in-game uncapped, FrameCap → 30, still FIFO.

Two caveats:

- **Do not use the Deck's QAM framerate limiter for this.** Gamescope caps the *layer's output*, not the base render, which fights the multiplier.
- If the game cannot hold 30 base, FIFO drops to 20 base / 40 output — a hard cliff, not a gradual sag. Lower settings rather than switching to Mailbox.

**Do not use `immediate`.** The shipped build's `into_present` also accepts `"immediate"` (`src/config/config.cpp:39-41`). The plugin's UI toggle only exposes fifo/mailbox, but a hand-written `conf.toml` will accept it. It carries all three Mailbox penalties (b1/b2/b3) plus tearing.

## 4. Making the cap work on other APIs

### DX12 (vkd3d-proton)

Recommended set — two override-proof env vars plus one config fallback:

```bash
export DXVK_FRAME_RATE=30
export VKD3D_FRAME_RATE=30
export DXVK_CONFIG="dxvk.maxFrameRate=30"
```

Lines 1–2 are the env vars for D3D9/10/11 and DX12 respectively. Line 3 is a belt-and-braces config path that also covers builds where the env var is not implemented (see below).

#### Two independent paths reach the DX12 limiter

**⚠ Correction to an earlier revision of this document,** which stated that DXVK's config cannot reach DX12 and that DX12 requires `VKD3D_FRAME_RATE`. The env-var half of that is correct; the config half is **wrong**. Traced through source:

1. `src/dxgi/dxgi_swapchain.cpp:36` — `m_frameRateOption = m_factory->GetOptions()->maxFrameRate` reads `dxgi.maxFrameRate`
2. `src/dxgi/dxgi_swapchain.cpp:1087` — calls `m_presenter2->SetTargetFrameRate(frameRate)` on the `IDXGIVkSwapChain2` interface (`src/dxgi/dxgi_interfaces.h:140-144`)
3. **vkd3d-proton implements that exact interface** — `dxgi_vk_swap_chain_SetTargetFrameRate` at `libs/vkd3d/swapchain.c:1320`, with `IDXGIVkSwapChain2 IDXGIVkSwapChain_iface` embedded in its swapchain struct (`swapchain.c:160`)

DXVK owns the DXGI layer for vkd3d-proton, so **`dxgi.maxFrameRate` does reach DX12 titles** via this cross-project vtable call. This is exactly the design the maintainer described in issue #1998: rather than aliasing env vars, extend the swapchain interface so DXVK config options propagate.

#### The two paths are NOT interchangeable

| | `VKD3D_FRAME_RATE=30` | `dxgi.maxFrameRate=30` |
|---|---|---|
| Sets `has_user_override` | **yes** (`swapchain.c:3880`) | no |
| Overrides game's own limiter | yes | **no — returns early** |
| Reaches DX12 | yes | yes (via vtable) |
| Reaches D3D9/10/11 | no | yes |

The precedence guard is `swapchain.c:1326-1328`:

```c
/* Env var takes priority over the display mode and config option */
if (chain->frame_rate_limit.has_user_override)
    return;
```

So if a game calls `SetTargetFrameRate` itself, or DXVK computes a refresh-derived limit, a config-only cap can be **overwritten at runtime**. The env var cannot — it wins permanently. This is why both are worth setting.

#### Other verified details

- vkd3d-proton reads **only** `VKD3D_FRAME_RATE` as an env var. The sole getenv is `libs/vkd3d/swapchain.c:3871` in `dxgi_vk_swap_chain_init_frame_rate_limiter`. **`DXVK_FRAME_RATE` is not an env-var alias** (debated in issue #1998, rejected) — the sharing happens at the config layer, not the env layer.
- Parsed with `strtod`: `0` = uncapped, positive = capped.
- **Sign convention:** negative means "only engage if the target is actually exceeded"; positive means hard cap. `fabs()` sets the interval, `frame_rate > 0.0` sets `enable` (`swapchain.c:1338-1344`, `dxgi_swapchain.cpp:1077-1079`). Pass positive values for a hard cap. DXVK's own per-game defaults use negatives like `-60`.
- The env-var limiter is a relatively recent addition (PR #2014). On older Proton it is silently ignored — which is precisely when the `DXVK_CONFIG` fallback earns its place.
- **`dxvk.maxFrameRate` is the API-agnostic key and it takes precedence.** Documented in DXVK's shipped `dxvk.conf:94-96`, and read at `src/dxgi/dxgi_options.cpp:172-173` and `src/d3d9/d3d9_options.cpp:45-46`:
  ```cpp
  this->maxFrameRate = config.getOption<int32_t>("dxvk.maxFrameRate",
                       config.getOption<int32_t>("dxgi.maxFrameRate", 0));
  ```
  Per `getOption` (`src/util/config/config.h:73-79`) the inner call is evaluated first and passed as the `fallback`; `parseOptionValue` overwrites it only if the outer key has a value. So **`dxvk.maxFrameRate` wins and the per-API keys are its fallback** — not the other way round. Setting the one key covers both APIs; setting both to different values means the `dxvk.*` value takes effect, despite the per-API key looking more specific.
- **DXVK-side value semantics** (documented at `dxvk.conf:86-92`) — note `-1` is not "cap at 1 FPS":
  - `n` — hard cap at n FPS.
  - `-n` — engage only if the game runs significantly faster for a short period. The docs state the limiter **will not engage when running into Vsync or an external limiter** while above n FPS. Under FIFO you *are* running into vsync, so a negative value would likely never engage here. Use positive values.
  - `-1` — **always disables** the limiter. A special value for games that default to a low refresh rate with no way to change it.

#### Env var / config names that do NOT exist

Verified absent from upstream source. All fail **silently** — no warning, since unknown env vars are never read and unknown config keys are ignored:

- **`VKD3D_FRAME_LIMIT`** — does not exist. A typo for `VKD3D_FRAME_RATE`; `FRAME_LIMIT` appears nowhere in vkd3d-proton.
- **`VKD3D_CONFIG="fps_limit=30"`** — wrong syntax *and* nonexistent option. `VKD3D_CONFIG` is not a key=value store: `vkd3d_parse_debug_options` (`libs/vkd3d-common/debug.c:466-481`) parses it as a list of **boolean flag names**, walking a fixed table and setting a bit per match. There is no mechanism to carry a numeric value. Additionally `is_option_separator` (`debug.c:439-442`) accepts only `,` `;` `\0`, so an `=30` suffix would prevent a match even if `fps_limit` existed — which it does not.

Contrast with `DXVK_CONFIG`, which **is** a key=value store: it splits on `;` and feeds each piece to `parseUserConfigLine` (`src/util/config/config.cpp:1786-1787`).

#### How to confirm it engaged

Do not trust the FPS reading alone — confirm which path fired. vkd3d logs both:

```
INFO("Set frame rate limit to %.1lf FPS via environment.\n", ...)   // swapchain.c:3877 — env var path
INFO("Set target frame rate to %.1lf FPS.\n", ...)                  // swapchain.c:1341 — config/vtable path
```

`just watch` (journalctl on the Deck) shows these. Seeing only the second message means the env var was not picked up and only the overridable config path is active — a game could still overwrite your cap.

**Placement:** `DXVK_CONFIG` and `VKD3D_FRAME_RATE` are both unknown keys to the plugin's script parser and will be **silently dropped** on the next config write (see section 5). Put them in Steam launch options, not in `~/lsfg`.

### Native Vulkan

No translation layer is in the path, so no `*_FRAME_RATE` variable exists. Two options, in order of preference:

**Option 1: use the game's own limiter** (this is case 1 — the recommended configuration anyway). Nothing to edit.

**Option 2: MangoHud's limiter**, which is itself a Vulkan layer:

```bash
export MANGOHUD=1
export MANGOHUD_CONFIG=fps_limit=30,fps_limit_method=early
```

Use `early` (wait *before* present) rather than `late`. `late` optimizes for latency, but `early` gives smoother frametimes, which matters far more here — FG amplifies base-frame jitter into visible artifacts, since unstable input frametimes mean the interpolated frame is placed at the wrong temporal position.

**The trap you must check:** layer ordering between MangoHud and lsfg-vk is decided by the Vulkan loader and is not reliably user-controllable. If MangoHud ends up *outside* lsfg-vk it sees post-FG presents and caps the **output** to 30 — giving 15 base × 2 = 30, far worse than no cap at all. Verify empirically: after setting `fps_limit=30`, confirm base 30 / output 60, not base 15 / output 30. If the latter, MangoHud cannot serve as a base limiter for that title; fall back to the in-game limiter.

Convenient hook: the plugin's **MangoHud Workaround** toggle already exports `MANGOHUD=1` (`config_schema_generated.py:108-109`), so with it on you only need to add the `MANGOHUD_CONFIG` line.

### OpenGL

Ordering matters — a limiter is the *second* problem, not the first.

lsfg-vk is a Vulkan layer, so it cannot touch a native GL context at all. **FG does not apply to an OpenGL title unless GL is first routed through Vulkan via Zink.** Enable the plugin's **Enable Zink** toggle, which exports:

```bash
export __GLX_VENDOR_LIBRARY_NAME=mesa
export MESA_LOADER_DRIVER_OVERRIDE=zink
export GALLIUM_DRIVER=zink
```

Once Zink is active the app presents through Vulkan, so lsfg-vk engages and the native-Vulkan advice above applies verbatim (in-game limiter first, MangoHud second, same layer-ordering caveat). Note that Zink itself adds CPU translation overhead, which eats into the power savings — worth measuring whether FG is net-positive at all for a given GL title.

## 5. ⚠ Hand edits to `~/lsfg` get overwritten

`~/lsfg` is regenerated from scratch on **every** config write (`py_modules/lsfg_vk/configuration.py:94-96`). Worse, the round-trip parser only recognizes keys in its generated match list (`config_schema_generated.py:54-93`); unknown keys like `VKD3D_FRAME_RATE` are **silently dropped** rather than erroring. So a hand edit vanishes the next time any slider in the UI is touched, with no warning.

**Recommended instead — Steam launch options,** which the plugin never rewrites and which scope per-game:

```
VKD3D_FRAME_RATE=30 ~/lsfg %command%
```

This works because the generated script ends in `exec "$@"` (`configuration.py:176`), so the environment set on the outside is inherited by the game. Per-title overrides generally belong here.

## 6. Optional durable fix

To make the FrameCap slider work on DX12 directly, have `dxvk_frame_rate` emit `VKD3D_FRAME_RATE` alongside `DXVK_FRAME_RATE`: add a special case in `scripts/generate_python_boilerplate.py` following the existing `enable_wsi` → `DXVK_HDR` precedent (generation at lines 133-137, parsing at lines 78-83), then run `just generate-schema`. Roughly 6 lines, no schema change, no UI change.

Prefer the env var over emitting `DXVK_CONFIG="dxvk.maxFrameRate=<n>"`: per section 4 the env var sets `has_user_override` and cannot be overwritten by the game at runtime, whereas the config option can. A single `VKD3D_FRAME_RATE` line is also simpler to round-trip through the script parser than a quoted compound value.

Note: per `CODEBUDDY.md`, **never** edit `config_schema_generated.py` or `generatedConfigSchema.ts` directly — they are derived files.

## 7. How to verify empirically

Use the performance overlay at level 4 (or MangoHUD) and read **GPU power (W)** at a fixed scene. Expected relation:

```
case 1 ≈ case 3 < case 5 < case 2 ≈ case 4
```

Also confirm the base:output frame ratio really is 1:2. If not, the limiter is acting at the wrong level (see the QAM and MangoHud layer-ordering traps in section 4).

**The Mailbox fingerprint (most diagnostic single test).** Set Mailbox and read presented FPS together with GPU power. If b1 is dominant you will see **~30 presented FPS at high GPU power** — full frame-generation cost, zero displayed benefit. This is unambiguous and does not require differential measurement.

All power and latency figures above are magnitude estimates derived from architectural mechanisms; confirm by measurement on your own unit. The *mechanisms* in sections 0.5, 0.6, and b1/b2/b3 are source-verified; the *magnitudes* are not.

## 8. When frame generation does not engage at all

Distinct from everything above, which assumes FG *is* working and asks how to tune it. This section is for "nothing happens." Symptom is telling: **no crash, no artifacts, just no effect.**

Ordered by how often it is the actual cause on a Steam Deck.

### 8.1 The game is 32-bit — the most common hard blocker

**This is not a DX11/DX12 issue and is easy to misdiagnose as one.** DXVK translates DX11 → Vulkan, so a DX11 game *does* present through Vulkan and is a perfectly valid FG target. Bitness is the blocker, not the API. A 64-bit DX11 game works fine.

The plugin installs exactly **one, 64-bit** `liblsfg-vk.so` (`constants.py:14`, installed to `.local/lib` per `installation.py:30`), and the layer JSON's `library_path` points at it (`installation.py:165-168`). A 32-bit process cannot load a 64-bit `.so`, so the loader either fails to load the layer or skips it. No 32-bit build is shipped — there is nothing to load.

Upstream lists this second in its basic checklist, and the wording is unusually blunt (`docs/Troubleshooting.md:9`):

> Ensure you are running a 64-bit game (try `PROTON_USE_WOW64=1`, but if it doesn't work then you're out of luck).

**Fix:** the plugin's **Enable WOW64** toggle (`shared_config.py:86-92`) exports `PROTON_USE_WOW64=1`, making Proton run 32-bit Windows code inside a 64-bit Linux process so the 64-bit layer can load. Both upstream and the plugin's own field description say to pair it with **ProtonGE**. This is the only real fix, and upstream flags it as frequently insufficient.

**Confirmed affected (measured with `file`, not inferred):**

```
Future Soldier DX11.exe:              PE32 executable (GUI) Intel 80386, 5 sections
METAL GEAR RISING REVENGEANCE.exe:    PE32 executable (GUI) Intel 80386, 6 sections
```

Note `Future Soldier DX11.exe` — a **DX11** binary that is nonetheless 32-bit. This is the exact trap described above: the filename advertises the API, which invites blaming DX11, while the real blocker is `PE32`. Era heuristic: many 2012-and-earlier DX11 titles ship 32-bit-only executables.

**Check bitness directly:**

```bash
file ~/.steam/steam/steamapps/common/<game>/*.exe
```

`PE32` = 32-bit (this is your problem). `PE32+` = 64-bit (look further down this list).

### 8.2 Confirm whether the layer loads at all

Two commands separate "installed wrong" from "installed fine but rejected per-process":

```bash
# 1. Is the layer visible system-wide? (needs vulkan-tools)
vulkaninfo | grep -i VK_LAYER_LSFGVK_frame_generation
```

No output → installation problem, revisit install (`Troubleshooting.md:10-11`).
Output present but game unaffected → layer is fine, failure is per-process. Consistent with 8.1.

```bash
# 2. Per-game loader trace — put in Steam launch options
VK_LOADER_DEBUG=layer %command%
```

Look for `VK_LAYER_LSFGVK_frame_generation` between `<Loader>` and `<Device>` (`Troubleshooting.md:12-13`). For a 32-bit game you typically see the loader consider the layer then fail with an ELF-class / wrong-architecture error. **That message is the confirmation for 8.1.**

⚠ **Upstream's `LSFGVK_ENV=1` / `active_in` advice does NOT apply to this plugin.** `Troubleshooting.md:14-16` describes current `develop`. The shipped fp16 build has **no** `active_in` or `LSFGVK_ENV` support — verified absent from `cb234bd`. It activates by matching `LSFG_PROCESS` against the profile name (`src/config/config.cpp:153-156`, `name.first.ends_with(pair.first) || name.second == pair.first`), which the plugin writes as `export LSFG_PROCESS=<profile>` (`configuration.py:127`, `:156`). Ignore that part of the upstream doc.

### 8.3 Layer loads but FG does not run

From `Troubleshooting.md:20-27`. Note every `pacing_mode = none` caveat applies unconditionally here, since that is the **only** mode in the shipped build (see section 0.6):

- **Explicitly enable V-Sync in-game**, and **disable VRR**. Not optional. Per section 0.5/b2 the layer depends on FIFO backpressure for both throttling and pacing — a game that disables vsync internally removes the mechanism FG relies on.
- **`ENABLE_GAMESCOPE_WSI=0`** — specifically called out for Gamescope/Steam Deck. The plugin's `enable_wsi` already defaults to `False` (`shared_config.py:126-132`), so this should already be set; verify it is.
- **Disable in-game upscaling** (DLSS, FSR, etc).
- **Disable other Vulkan layers** (VkBasalt, MangoHud). The plugin has `disable_vkbasalt` / `force_enable_vkbasalt` toggles.
- **Black screen / game won't launch:** wrong GPU selected for the profile (`Troubleshooting.md:29-30`).

### 8.4 Check for a game-side framerate ceiling before blaming the layer

Some engines hard-cap internally, often because physics or animation is tied to the timestep. **Metal Gear Rising is a known 60 FPS-capped title.**

This matters arithmetically: on a 60 Hz panel with multiplier 2 you need **30 base** to reach 60 output. If the game already presents 60 and cannot exceed it, FG has no headroom — forcing 30 base gives you 30→60, i.e. the same 60 output you already had. Confirm your actual **base** FPS before concluding the layer is broken.

**On power, specifically — do not assume FG costs more here.** Per section 1.0 the comparison is `30R + 30I` vs `60R`, so it turns on whether `I < R`. An earlier revision of this document asserted "at extra power cost" for this case; that was unjustified. What is true is that a 2013 title the Deck runs easily has a *small* `R`, which pushes it toward the regime where `I` is comparable to `R` and the saving evaporates. Halving base render rate also halves CPU work, cutting the other way. **The honest verdict for MGR is power-neutral-at-best and genuinely uncertain without measurement** — not a definite cost.

**The recommendation does not depend on the power argument.** FG is still not worth enabling here:

- **No smoothness gain** — output is 60 either way on a 60 Hz panel. This alone settles it.
- **Latency clearly worse** — base frame time goes 16.7 → 33 ms, *plus* the unavoidable ~16.7 ms interpolation hold-back (section 2, component 2). Roughly 50 ms versus roughly 20–30 ms native. Mechanism-backed, unlike the power claim.
- **Artifacts introduced** where previously there were none.

Same output rate, worse latency, new artifacts, power neutral at best.

### 8.5 ⚠ Overlays frequently lie about FPS with Vulkan layers

`Troubleshooting.md:34-37` — an acknowledged limitation:

> If you are using performance overlays like Steam's built-in overlay, there is a good chance that they will not show the correct framerate. This is a known limitation of Vulkan layers […] there is little that can be done to fix this.

So "the framerate didn't go up" may be partly an overlay artifact rather than a real failure. Two more reliable signals:

- **Perceived motion smoothness.**
- **GPU power draw.** Interpolation costs `I` per frame and cannot be free. Note the baseline here is *same base FPS with the layer inactive* (`30R` vs `30R + 30I`) — not native 60 (`60R`), which is the different comparison in section 1.0. Against a same-base-FPS baseline, an engaged layer always draws more. So: flat power + flat FPS = layer genuinely not running (go to 8.1/8.2). Raised power + flat FPS = layer running, overlay misreporting (or the Mailbox b1 pathology from section 1).

---
---

# 中文版：LSFG-VK 呈现模式与帧率限制 —— 功耗、延迟与跨 API 适配

适用场景：Steam Deck（60Hz 面板），倍率 `multiplier = 2`，即 30 基础帧 → 60 输出帧。

以下结论中的机制部分均经直接阅读源码核实，非推断：

- **lsfg-vk 层** —— `/mnt/e/GitHub/lsfg-vk`，`develop` 分支，HEAD `8b0da26`，并追溯了已被删除的 present mode 标志的 git 历史
- **vkd3d-proton** —— 上游 master 的 `libs/vkd3d/swapchain.c`
- **本插件** —— `py_modules/`、`src/`、`shared_config.py`

## 0. 术语澄清（重要）

插件 UI 里的 **"Base FPS Cap"（FrameCap）滑块并不是 lsfg-vk 的功能**。它只做一件事：向 `~/lsfg` 写入一行

```bash
export DXVK_FRAME_RATE=<n>
```

见 `py_modules/lsfg_vk/config_schema_generated.py:101-103`，环境变量名的映射硬编码在 `scripts/generate_python_boilerplate.py:33`。

`DXVK_FRAME_RATE` 只被 DXVK 读取，而 DXVK 实现的是 D3D9/10/11。**对 DX12、原生 Vulkan、OpenGL 这一行完全无效**，于是"设了限帧"的配置会静默退化成"没限帧"。

注意环境变量与 config 选项的作用范围**不同**：`DXVK_FRAME_RATE`（env）不作用于 DX12，但 `dxvk.maxFrameRate`（config）经由一个跨项目接口**可以**。详见第 4 节——这一区别很容易搞错。

而 **Present Mode（FIFO / Mailbox）** 是 `conf.toml` 里的 `experimental_present_mode` 字段（`shared_config.py:70-76`），由 lsfg-vk 层作用在真实 swapchain 上。

两者层级完全不同：一个是渲染前的限帧器，一个是呈现阶段的队列策略。

## 0.5 层的 present 序列（下文一切结论的源头）

每个游戏帧，层会在**游戏自己的 swapchain 上发出 `multiplier` 次 present**。当前 `develop`（`lsfg-vk-layer/src/swapchain.cpp:205-301`）与插件实际发布的 fp16 构建（`src/context.cpp:169-232`，函数起始于 `:125`）在这一点上结构完全一致：

1. 把游戏帧 blit 进 backend 源图像（`swapchain.cpp:172-203`）
2. 循环 `multiplier - 1` 次 —— 每个插值帧：`vkAcquireNextImageKHR`（`swapchain.cpp:212`，`UINT64_MAX` 超时）→ 把插值结果 blit 进取得的图像 → `vkQueuePresentKHR`（`swapchain.cpp:281`）
3. **然后**才 present 原始的真实帧（`swapchain.cpp:289-301`）

由此得出两个关键事实，二者都至关重要：

- **插值帧先 present，真实帧后 present。**
- **两次 present 相隔微秒级，而非毫秒级。** 它们通过信号量在 GPU 侧串行化（`lastPCS.second`，`swapchain.cpp:294`），并没有任何定时器来拉开间隔。这段代码里没有任何东西在做帧间隔控制。

而 FIFO 的阻塞行为恰好补上了这个缺失的帧间隔控制。这是本文档中最重要的一个事实。

## 0.6 上游已删除 Mailbox 选项

`git log -S "e_present"` 给出了该标志的完整生命周期：

- 2025-07-18 由 `8b29b95`（"add experimental flags for present mode and fps limit"）引入，替换掉原本硬编码的 `VK_PRESENT_MODE_FIFO_KHR` —— 它删掉的那行注释写的是 `// enforce vsync`
- 一直保留到 `v1.0.0`（`7113d7d`）
- 2025-12-19 由 `e0fac3e`（"refactor(cleanup): implement basic/none frame pacing"）**完全移除**

当前 `develop` 的所有 C++ 源码中"mailbox"出现次数为**零**。FIFO 现在被硬编码在两处：创建时（`swapchain.cpp:64`），以及**每一次** present 时重新强制一遍（`swapchain.cpp:155`），用于防御来自 gamescope 或游戏的 `VkSwapchainPresentModeInfoEXT` 覆盖。

替代它的抽象很能说明问题：

```cpp
enum class Pacing : uint8_t {
    /// do not perform any pacing (vsync+novrr)
    None
};
```

一个只有单个取值的枚举（`lsfg-vk-common/include/lsfg-vk-common/configuration/config.hpp:23-26`），对应的 `switch` 只有一个 case。作者把这块从"选哪个 present mode"重构成了"选哪种**帧间隔控制（pacing）策略**"——准确地指出真正的问题是 pacing 而不是撕裂——然后只提供了一个选项：FIFO。

Mailbox 不是被弃用或隐藏，而是被判定为错误方案而砍掉，由帧生成层的作者从内部做出的判断。

**版本说明**：插件发布的是 `xXJSONDeruloXx/lsfg-vk` 的 `fp16-test-2` 标签（`package.json:52-53`），早于 `e0fac3e` 这次 pacing 重构。这正是插件 UI 里至今还有 present mode 开关的原因——所发布的构建中该选项确实还存在，只是不应该去用。

## 1. 五种配置的功耗对比

### 1.0 前提：FG 究竟在什么条件下省电？`I < R` 条件

下文五种配置都预设了"FG 值得开"。这个预设是有条件的，且值得在比较调优选项之前先讲清楚——后面那张表比较的是**各配置之间**的优劣，而不是"开 FG"对"不开 FG"。

设 `R` = 渲染一帧所需能量，`I` = 生成一张插值帧所需能量。倍率 2、目标 60 输出帧时：

- **原生 60FPS**：`60R`
- **FG 30→60**：`30R + 30I`

因此 FG 省电的充要条件是 **`I < R`**。这正是 FG 作为省电手段的全部依据，而它并非自动成立。

**为什么 `I < R` 通常成立**：`I` 是**与场景无关**的——它只取决于输出分辨率与模型选择，不取决于几何体数量、光源数量或着色器复杂度。每个基础帧，层都要付出一组固定的工作（见 `swapchain.cpp`）：把游戏帧 blit 进 backend 源图像（`:172`）、光流 + 混合（`scheduleFrames`，`:140`）、把结果 blit 进取得的 swapchain 图像（`:226`）。即两次全分辨率图像拷贝加上光流模型，每帧都要做，与画面内容无关。在 Deck 上这些 blit 还会消耗 LPDDR5 带宽，而带宽是与 CPU 共享的，其本身也是实打实的功耗。

而 `R` 则随游戏的负载而变。于是：

- **重负载游戏**（`R` 大）→ `I ≪ R` → FG 显著省电。这是它的目标场景。
- **轻量 / 老游戏**（`R` 小）→ `I ≈ R` → 收益消失，两个方向都有可能。
- **本来就已经顶到刷新率上限的游戏** → 各个维度上的最差情形。见 8.4 节。

**有两个因素朝反方向起作用，不应遗漏**：

1. 基础渲染帧率减半同时也让 **CPU 工作量减半**（模拟、剔除、draw 提交）。在 Deck 共享 TDP 预算的前提下这是实打实的节省，且与 `R` 对 `I` 的比较无关。
2. 插件的默认值并非最省的：`flow_scale` 默认 **0.8**（`shared_config.py:49`），`performance_mode` 默认 **False**（`shared_config.py:57`）——即默认开启的是**更重**的模型，尽管它自己的描述建议"大多数游戏"用更轻的那个。降低 `flow_scale` 与开启 `performance_mode` 都会减小 `I`，能把临界情形推向有利一侧。

**实践含义**：若某游戏轻到 Deck 能轻松顶满刷新率，不要假定 FG 省电——请实测。若它重到基础帧率远低于刷新率，则 `I < R` 稳稳成立，此时下文的调优问题才是真正相关的问题。

### 1.1 各配置对比

下表与后续各"原因"比较的是五种配置**彼此之间**的差异，且都以倍率 2 开启 FG 为前提。它们不涉及 1.0 节"开不开 FG"的问题。

| # | 基础限帧 | 呈现模式 | 相对功耗 | 主要原因 |
|---|---|---|---|---|
| 1 | 游戏内 30 | FIFO | **基准（最低）** | 截止时间调度 + FIFO 免费完成帧间隔控制 |
| 3 | DXVK 30 | FIFO | +0~2% | 轻微 CPU 提前跑（原因 d） |
| 5 | 无限制 | FIFO | +2~8% | 高 V/f 驻留（原因 a）+ CPU 提前跑（原因 c） |
| 2 | 游戏内 30 | Mailbox | +5~12% | 插值帧被丢弃（原因 b） |
| 4 | DXVK 30 | Mailbox | +5~12% | 原因 b + 原因 d |

### 原因 a：DVFS 电压调节（主导因素）

APU 依据采样窗口内的活动占比选择 GFX 频率。动态功耗 `P ≈ C·V²·f`，而在 DVFS 区间内 `V` 随 `f` 近似线性上升，因此**同样的工作量，功耗大致按 `f³` 变化**。

这解释了为什么"以什么方式达到 30FPS"比"是否达到 30FPS"更重要：

- **显式限帧器（case 1~4）**：限帧器让帧循环睡到一个墙钟截止时间。GPU 每帧获得一段连续空闲。调度器看到中等占用率 → 选中等频率 → 落到更低电压档。帧数相同，`V²f` 显著更低。
- **仅靠背压（case 5）**：游戏全速渲染直到 swapchain 填满，然后阻塞在 present。平均工作量相同，但**是成簇的**：Boost 频率下的一段爆发 + 一段停顿。调度器的采样窗口看到高瞬时活动 → 维持高频 → 同样的工作在更高 V/f 工作点上完成。

在这颗 SoC 上，"冲刺到空闲再等待"严格劣于"按截止时间匀速"，因为空闲间隙无法追回已经付出的电压代价。

### 原因 b：Mailbox 的代价（三个独立机制）

已对照层源码核实。这是本配置下最关键的不对称性。本文档早先的版本把它描述为"部分帧被丢弃"——结论方向正确，但对成因的刻画明显不足。实际上存在三个彼此独立的机制。

#### b1 —— Mailbox 丢弃的恰好是插值帧

Mailbox 在规范上是一个**内部单槽队列**：若已有条目待呈现时来了新的 present，新请求会**替换**掉它，被替换的图像立即变为可复用。

把这一点与 0.5 节的 present 顺序结合起来看：同一个 vblank 区间内，层先 present 插值帧，再 present 真实帧。**真实帧把插值帧从 mailbox 槽里替换掉了。** 面板显示的是真实帧。插值帧在光流、混合、blit 全部算完之后被直接丢弃。

这比对称的丢帧更糟，因为损失是**系统性偏向的**：被丢弃的永远是昂贵的生成帧，永远不是廉价的真实帧。极限情况下你付出了 100% 的帧生成成本，显示出来的却是纯真实帧的 30FPS。等于什么都没买到。

FIFO 下这不可能发生。FIFO 队列是有序的，两次 present 都会被显示、每个 vblank 一帧——并且它的阻塞行为免费地把两者精确拉开到 16.67ms。

#### b2 —— Mailbox 移除了层唯一的节流点

这才是真正的功耗机制，而且比"DVFS 驻留效应"要锐利得多。

在插件实际发布的 fp16 构建中，`LsContext::present` **完全没有 fence 等待**——整个函数里唯一的 `UINT64_MAX` 就是 `src/context.cpp:173` 处的 `vkAcquireNextImageKHR`。其余全是 GPU 侧的信号量串行化，不会阻塞调用线程。

因此 `vkAcquireNextImageKHR`（以及图像耗尽后的 `vkQueuePresentKHR`）是这个循环运行速度的**唯一**调节器。

- **FIFO 下**：取得的图像会被呈现引擎一直持有到它的 vblank。swapchain 有约 4~5 张图像（`swapchain.cpp:60`，`createInfo.minImageCount += profile.multiplier`），队列填满后 acquire 就会阻塞，该阻塞反向传播回游戏的帧循环。**这正是 case 5 仍能把基础帧率压在 30 的原因。**
- **Mailbox 下**：按规范，被替换的图像**立即**可复用。acquire 不再阻塞。**节流消失了。**

后果：`scheduleFrames`（`swapchain.cpp:140`）会以游戏能 present 的最快速度被无界地调用。插值工作入队 GPU 的速度快于显示消费的速度，GPU 被将要按 b1 丢弃的工作填满。这——而非某种细微的电压驻留效应——才是 Mailbox 耗电的原因。

旁证：上游 `develop` **新增**了一个显式的 `renderFence->wait(vk, 150ms)`（`swapchain.cpp:164`），而所发布的 fp16 构建没有。上游显然也意识到仅依赖 acquire 来节流是脆弱的。

#### b3 —— 无 pacing 的 present 造成差频抖动

节流消失后，没有任何东西把层的 present 节奏与 60Hz vblank 绑定。present 对会相对 vblank 时钟漂移。多数落在同一区间内（b1），少数跨越边界从而两帧都被显示。

于是有效输出既不是干净的 30 也不是 60，而是以漂移速率在两者之间游走，表现为周期数秒的缓慢卡顿。这比稳定的 30 更难接受——视觉系统能适应恒定节奏，但无法适应变化的节奏。

它还会独立于丢帧本身而损害画质。插值假定生成帧处于 N 与 N+1 之间的固定时间中点；输入帧时间不稳定会把它放在错误的时间位置上，加重重影与形变。

#### 为什么"没有余量可供利用"这个直觉是对的

Mailbox 的适用前提是"输出帧率 > 刷新率"，或"FIFO 背压把基础帧率压崩了"。倍率 2 + 60Hz 时输出帧率恰好等于刷新率，不存在余量。但更锐利的表述是：**Mailbox 不只是帮不上忙，它摧毁了这个层的设计所依赖的 FIFO 背压——而该背压同时承担着节流与 pacing 两项职责。**

#### 验证方法

开 Mailbox 后观察 presented FPS。若 b1 占主导，你会看到 **presented FPS 约 30，同时 GPU 功耗很高**——付出全额成本、显示端零收益。这是直接且无歧义的指纹。帧时间曲线上的成对尖峰指向同一现象。

### 原因 b-deck：gamescope 让 Mailbox 在本设备上失去意义

在 gamemode 下 gamescope 是合成器，它总是按设定刷新率向面板呈现。本插件的 `enable_wsi` 默认为 `False`（`shared_config.py:126-132`），会导出 `ENABLE_GAMESCOPE_WSI=0`，因此游戏是向普通 Wayland surface 呈现，而不是走 gamescope WSI 层。

无论走哪条路，"Mailbox"都是由 gamescope 的合成行为模拟的，而不是由显示硬件实现的。gamescope 不会比它的下一次合成更早显示插值帧，所以 Mailbox 唯一的理论收益——跳过排队帧从而降低延迟——大部分被合成器吸收了。而 b1/b2/b3 则全额生效，因为它们发生在 gamescope 之上游、在层内部。

**结论：在 Steam Deck 的 gamemode 下，Mailbox 基本没有任何好处。** 见第 2 节的延迟修订。

### 原因 c：CPU 提前跑与共享 TDP

Deck 的 CPU 和 GPU 共用一份功耗预算。case 5 中模拟循环从不被节流——它会提前完成剔除、draw call 组装、命令录制，而这些帧随后只是排在队列里等待。这部分 CPU 工作是实打实的功耗；在 TDP 受限时，CPU Boost 驻留会直接抢走 GPU 的余量。显式限帧器让整个循环睡眠，CPU 和 GPU 一起空闲。

### 原因 d：限帧器睡在哪里（case 1 vs 3 的差异）

两者都是截止时间型限帧器，但挂载点不同：

- **游戏内限帧（case 1）**：睡在引擎主循环里。模拟、提交、呈现被统一节流。
- **DXVK 限帧（case 3）**：根据 vkd3d-proton issue #1998 中维护者的说明，DXVK 的限帧器 *"运行在处理 present 等待的同一线程上，通过这种方式反馈到应用逻辑，而不是直接阻塞 DXGI present"*。因此游戏 CPU 侧会稍微提前跑一点，节流才反向传播回来——CPU 功耗和延迟都略高。

**一个会翻转的例外**：某些游戏内限帧器用**忙等（busy-wait）**而非睡眠来命中截止时间。此时 case 1 在功耗上会输给 case 3，因为空转的 CPU 核心会把整个封装维持在 Boost 状态。如果实测发现 case 1 比 case 3 更耗电，忙等式的游戏内限帧器是首要怀疑对象。

## 2. 输入延迟对比

### 延迟链的组成

1. **采样→渲染**：约 1 个基础帧 ≈ 33ms（30FPS）。五种配置相同。
2. **帧生成固有代价：+16.7ms，五种配置全都无法避免。** 插值不是外插：中间帧位于 N 和 N+1 *之间*，因此层必须等 N+1 已经存在才能输出它。真实帧 N+1 因此必须被推迟整整一个输出间隔（60Hz 下 16.67ms），好让插值帧占住它前面的时隙。这是 FG 的固有属性，任何设置都消不掉。
3. **swapchain 队列深度——可变项。** FIFO 按序排队，队列*填满*时会额外引入 1~2 个显示间隔的陈旧度。Mailbox 丢弃而非排队，延迟无法累积。这是 Mailbox 唯一有理论依据的维度——但见下方修订。
4. **采样新鲜度。** 睡眠式截止时间限帧器会在截止时间*前夕*才开始这一帧，因此输入采样尽可能晚。背压限帧则相反：你看到的帧早就渲染好了，一直在队列里变旧。

### 修订：在 Steam Deck 上 Mailbox 的延迟优势接近于零

下表中 −5 ~ −8ms 的数值假定的是**直接向显示器呈现**。在 gamescope 下该假定不成立（原因 b-deck）：合成器不会比它的下一次合成更早显示插值帧，理论收益大部分被吸收。gamemode 下 Mailbox 相关行应视为 **≈0 ~ −3ms**，并且 b3 的抖动本身是一项*延迟方差*代价，会进一步抵消这点收益。

这使得 case 1 vs case 2 的取舍弱化到不再构成真正的取舍。见修订后的结构性结论。

### 排序（优 → 劣）

| # | 配置 | 相对 case 1 | 原因 |
|---|---|---|---|
| 2 | 游戏内 30 + Mailbox | **直连 −5~−8ms；gamescope 下 ≈0~−3ms** | 采样最晚 + 无队列累积（第 3 项） |
| 1 | 游戏内 30 + FIFO | **基准（约 55ms）** | 采样晚，队列浅 |
| 4 | DXVK 30 + Mailbox | +0 ~ +5ms | 无队列，但有 CPU 提前跑（原因 d） |
| 3 | DXVK 30 + FIFO | +5 ~ +10ms | 提前跑 + present 排队 |
| 5 | 无限制 + FIFO | **+20 ~ +30ms** | 队列被永久填满 |

**关于 case 5 的说明**：基础帧率其实**仍被压在 30**——FIFO 每秒只接纳 60 次 present，层每个基础帧发 2 次，所以背压会经由 `vkAcquireNextImageKHR`（见 b2）反向传播到游戏。但其机制是"提前渲染，然后阻塞在 acquire/present"，意味着你看到的每一帧都是 2~3 个输出间隔之前渲染的。这就是经典的"不限帧 + 垂直同步"延迟惩罚，而 FG 会放大它，因为层自己的 present 队列与 swapchain 队列是串联的。

### 结构性结论（已修订）

对比两张排序表：**case 5 被严格支配**——功耗与延迟两方面都更差，没有任何补偿性收益。

本文档早先的版本把 case 1 vs case 2 当作一个真实取舍（Mailbox：功耗更差、延迟更好）。**源码复核否定了这一判断。** 在 gamescope 下 Mailbox 的延迟优势接近于零（b-deck），而它的代价是三个独立机制（b1 浪费算力、b2 失去节流、b3 抖动）而非一个。因此 Mailbox 在本硬件上也接近于被严格支配。

**推荐 case 1；在 60Hz + 倍率 2 的条件下，不存在任何让 Mailbox 取胜的配置。** 上游得出了同样的结论并删除了该选项（见 0.6 节）。

### 2.1 底层规则，以及为什么"关闭限帧"不是一个设置

case 1 与 case 5 的差距常被误读为"开游戏内限帧优于关游戏内限帧"。并非如此。通用规则是：

> **整条链路上恰好有一个截止时间型限帧器，且尽可能靠近输入采样点。** 零个截止时间型限帧器（仅靠背压）是最差情形。两个同时生效、目标相同的限帧器会造成帧间隔抖动。

case 5 之所以糟糕，不是因为某个勾选框被关掉，而是因为**整条链路上不存在任何截止时间型限帧器**——把基础帧率压在 30 的唯一机制是 FIFO 背压阻塞在 `vkAcquireNextImageKHR`（见 b2 节）。"提前渲染，然后停顿"意味着你看到的每一帧都是 2~3 个输出间隔之前采样的。

**这就是为什么 Deck 上的结论不能推广成"永远开启游戏内限帧"。** 在 Deck 上，游戏内限帧器通常是*唯一*可用的截止时间型限帧器，于是 case 1 与"开限帧"恰好重合。换个平台，两者就分离了。

#### 反例推演：桌面平台、驱动层限帧、不开 FG

配置：RTX 5070 Ti / 9700X，Nvidia App 驱动层限帧 60，帧生成**关闭**。

此处驱动层限帧**本身就是**一个生效中的截止时间型限帧器——它把管线睡到墙钟目标时间，配合 Reflex 还会做渲染队列缩减。因此关闭游戏内限帧**不会**产生 case 5，链路中依然恰好有一个正规限帧器。按本文档的分类，该配置是 **case 1/3 的类比，而非 case 5。**

游戏内限帧器的正确设置：**关闭**，或设为略*高于*驱动上限的值（63~65），纯粹作为菜单/加载画面的兜底。设成*正好* 60 会让两个限帧器互相争抢；设成*低于* 60 则它成为实际生效的那个，从而静默丢弃驱动限帧器基于 Reflex 的调度。

60 上限、不开 FG 时的配套设置：

- **Reflex：开**（不是"开 + Boost"）——与驱动上限协同工作；这是延迟的主导项。
- 游戏内有 Reflex 时，驱动的 **Low Latency Mode 设为 Off**；仅对没有 Reflex 的游戏才用 **Ultra**。
- **有 G-Sync/VRR**：G-Sync 开，驱动内 V-Sync 开，上限留出足够余量低于刷新率——这是延迟最低的无撕裂配置。
- **无 VRR**：V-Sync 关；可考虑设 58~59 而非正好 60，避免贴着刷新率上限。

#### ⚠ 哪些结论无法迁移

第 2 节所有延迟数值都以 **FG 已生效**为前提，因此不可避免的约 16.7ms 插值滞留（第 2 项）被计入了每一行，包括 case 1 那个约 55ms 的基准。不开 FG 的桌面配置完全不付这一项。**跨平台可迁移的只有限帧器放置逻辑，绝对数值不可迁移。**

## 3. 推荐配置

游戏内限帧器 → 30；FrameCap 滑块 → Off；Present Mode → FIFO。

若游戏自带限帧器不可靠（在 28~31 之间震荡、或与其 vsync 选项绑死）或根本没有，则反过来：游戏内不限帧，FrameCap → 30，仍用 FIFO。

两个注意事项：

- **不要用 Deck 快捷菜单（QAM）的帧率限制**来做这件事。gamescope 限制的是*层的输出*而不是基础渲染，会与倍率打架。
- 若游戏撑不住 30 基础帧，FIFO 会掉到 20 基础 / 40 输出（是硬台阶式跌落，不是渐进下滑）。遇到这种情况应当降画质，而不是切到 Mailbox。

**不要用 `immediate`。** 所发布构建的 `into_present` 还接受 `"immediate"`（`src/config/config.cpp:39-41`）。插件 UI 只暴露 fifo/mailbox，但手写 `conf.toml` 是可以设进去的。它同时具备 Mailbox 的三项代价（b1/b2/b3），外加画面撕裂。

## 4. 跨 API 让限帧生效

### DX12（vkd3d-proton）

推荐配置——两个不可被覆盖的环境变量，加一个 config 兜底：

```bash
export DXVK_FRAME_RATE=30
export VKD3D_FRAME_RATE=30
export DXVK_CONFIG="dxvk.maxFrameRate=30"
```

第 1、2 行分别是 D3D9/10/11 与 DX12 的环境变量；第 3 行是兜底的 config 路径，可覆盖环境变量尚未实现的构建（见下）。

#### 有两条独立路径能到达 DX12 限帧器

**⚠ 对本文档早先版本的更正**：早先版本称 DXVK 的 config 无法作用于 DX12、DX12 必须用 `VKD3D_FRAME_RATE`。其中环境变量部分正确，config 部分**是错的**。源码追踪如下：

1. `src/dxgi/dxgi_swapchain.cpp:36` —— `m_frameRateOption = m_factory->GetOptions()->maxFrameRate` 读取 `dxgi.maxFrameRate`
2. `src/dxgi/dxgi_swapchain.cpp:1087` —— 在 `IDXGIVkSwapChain2` 接口上调用 `m_presenter2->SetTargetFrameRate(frameRate)`（接口定义见 `src/dxgi/dxgi_interfaces.h:140-144`）
3. **vkd3d-proton 实现了这个接口** —— `dxgi_vk_swap_chain_SetTargetFrameRate` 位于 `libs/vkd3d/swapchain.c:1320`，其 swapchain 结构体内嵌 `IDXGIVkSwapChain2 IDXGIVkSwapChain_iface`（`swapchain.c:160`）

DXVK 为 vkd3d-proton 提供 DXGI 层，因此 **`dxgi.maxFrameRate` 确实能通过这个跨项目虚表调用作用到 DX12 游戏**。这正是维护者在 issue #1998 中描述的设计：不做环境变量别名，而是扩展 swapchain 接口让 DXVK 的 config 选项传导过去。

#### 两条路径并不等价

| | `VKD3D_FRAME_RATE=30` | `dxgi.maxFrameRate=30` |
|---|---|---|
| 是否置 `has_user_override` | **是**（`swapchain.c:3880`） | 否 |
| 能否覆盖游戏自带限帧器 | 能 | **不能——会提前 return** |
| 是否作用于 DX12 | 是 | 是（经虚表） |
| 是否作用于 D3D9/10/11 | 否 | 是 |

优先级判断在 `swapchain.c:1326-1328`：

```c
/* Env var takes priority over the display mode and config option */
if (chain->frame_rate_limit.has_user_override)
    return;
```

也就是说，若游戏自己调用 `SetTargetFrameRate`，或 DXVK 依刷新率算出一个限制值，仅靠 config 设定的上限可能在**运行时被覆盖**。环境变量则不会——它永久胜出。这就是两者都值得设置的原因。

#### 其它已核实细节

- vkd3d-proton 作为环境变量**只**读取 `VKD3D_FRAME_RATE`，唯一的 getenv 在 `libs/vkd3d/swapchain.c:3871` 的 `dxgi_vk_swap_chain_init_frame_rate_limiter` 中。**`DXVK_FRAME_RATE` 不是环境变量层面的别名**（issue #1998 讨论后否决）——共享发生在 config 层，而不是 env 层。
- 用 `strtod` 解析：`0` = 不限制，正值 = 限制。
- **符号约定**：负值表示"仅当实际帧率超过目标时才启用限帧"，正值表示硬性上限。`fabs()` 决定间隔，`frame_rate > 0.0` 决定 `enable`（`swapchain.c:1338-1344`、`dxgi_swapchain.cpp:1077-1079`）。要硬性上限就传正值。DXVK 自带的逐游戏默认值用的是 `-60` 这类负数。
- 环境变量版限帧器是较新加入的（PR #2014）。旧 Proton 上会被静默忽略——这恰恰是 `DXVK_CONFIG` 兜底路径的价值所在。
- **`dxvk.maxFrameRate` 是与 API 无关的键，且优先级更高。** 它记载于 DXVK 自带的 `dxvk.conf:94-96`，读取处为 `src/dxgi/dxgi_options.cpp:172-173` 与 `src/d3d9/d3d9_options.cpp:45-46`：
  ```cpp
  this->maxFrameRate = config.getOption<int32_t>("dxvk.maxFrameRate",
                       config.getOption<int32_t>("dxgi.maxFrameRate", 0));
  ```
  按 `getOption`（`src/util/config/config.h:73-79`）的语义，内层调用先求值并作为 `fallback` 传入，只有当外层键确实有值时 `parseOptionValue` 才会覆盖它。因此**是 `dxvk.maxFrameRate` 胜出、逐 API 的键作为它的回退**——而非反过来。只设这一个键即可覆盖两类 API；若两者设成不同值，生效的是 `dxvk.*` 的值，尽管逐 API 的键看起来更"具体"。
- **DXVK 侧的取值语义**（记载于 `dxvk.conf:86-92`）——注意 `-1` 并不是"限制到 1FPS"：
  - `n` —— 硬性限制到 n FPS。
  - `-n` —— 仅当游戏在短时间内显著快于目标时才启用。文档明确指出：当撞上 Vsync 或外部限帧器时，即使帧率高于 n，限帧器**也不会启用**。而 FIFO 下你*正是*在撞 vsync，因此负值在这里很可能永远不生效。请用正值。
  - `-1` —— **永久禁用**限帧器。这是为那些默认低刷新率且无法更改的游戏预留的特殊值。

#### 不存在的环境变量 / config 名

以下均已核实在上游源码中不存在，且全部**静默失效**——不会有任何警告，因为未知环境变量根本不会被读取，未知 config 键会被忽略：

- **`VKD3D_FRAME_LIMIT`** —— 不存在。是 `VKD3D_FRAME_RATE` 的拼写错误；`FRAME_LIMIT` 在 vkd3d-proton 中任何地方都没有出现。
- **`VKD3D_CONFIG="fps_limit=30"`** —— 语法错误*且*选项不存在。`VKD3D_CONFIG` 不是 key=value 存储：`vkd3d_parse_debug_options`（`libs/vkd3d-common/debug.c:466-481`）把它解析为**布尔标志名列表**，遍历一张固定表、命中就置一个位，根本没有承载数值的机制。此外 `is_option_separator`（`debug.c:439-442`）只接受 `,` `;` `\0`，所以即便 `fps_limit` 存在，`=30` 后缀也会导致匹配失败——而它并不存在。

对比 `DXVK_CONFIG`，后者**确实**是 key=value 存储：它按 `;` 切分，再把每段交给 `parseUserConfigLine`（`src/util/config/config.cpp:1786-1787`）。

#### 如何确认已生效

不要只看 FPS 数字，要确认是哪条路径生效了。vkd3d 两条都有日志：

```
INFO("Set frame rate limit to %.1lf FPS via environment.\n", ...)   // swapchain.c:3877 —— 环境变量路径
INFO("Set target frame rate to %.1lf FPS.\n", ...)                  // swapchain.c:1341 —— config/虚表路径
```

用 `just watch`（Deck 上的 journalctl）即可看到。若只看到第二条而没有第一条，说明环境变量没被读取、只有可被覆盖的 config 路径在起作用——游戏仍可能覆盖掉你的上限。

**放置位置**：`DXVK_CONFIG` 与 `VKD3D_FRAME_RATE` 对插件的脚本解析器都是未知键，会在下一次配置写入时被**静默丢弃**（见第 5 节）。请放进 Steam 启动选项，不要手改 `~/lsfg`。

### 原生 Vulkan

链路中没有翻译层，因此不存在任何 `*_FRAME_RATE` 变量。两个方案，按优先级：

**方案 1：用游戏自带限帧器**（这就是 case 1，本来就是推荐配置）。无需改动。

**方案 2：MangoHud 限帧器**（它本身也是一个 Vulkan 层）：

```bash
export MANGOHUD=1
export MANGOHUD_CONFIG=fps_limit=30,fps_limit_method=early
```

用 `early`（present *之前*等待）而非 `late`。`late` 优化延迟，但 `early` 给出更平滑的帧时间——在这里后者重要得多，因为 FG 会把基础帧的抖动放大成可见瑕疵：输入帧时间不稳定意味着插值帧被放在错误的时间位置上。

**必须验证的陷阱**：MangoHud 与 lsfg-vk 的层顺序由 Vulkan loader 决定，用户无法可靠控制。若 MangoHud 排在 lsfg-vk *外侧*，它看到的是 FG 之后的 present，于是会把**输出**限制到 30 —— 变成 15 基础 × 2 = 30，比不限帧糟糕得多。实测确认：设 `fps_limit=30` 后应看到基础 30 / 输出 60，而不是基础 15 / 输出 30。若是后者，该游戏无法用 MangoHud 做基础限帧，必须回退到游戏内限帧器。

便利点：插件的 **MangoHud Workaround** 开关已经会导出 `MANGOHUD=1`（`config_schema_generated.py:108-109`），所以开着它的话只需再补 `MANGOHUD_CONFIG` 一行。

### OpenGL

顺序很关键——限帧是*第二*个问题，不是第一个。

lsfg-vk 是 Vulkan 层，完全碰不到原生 GL 上下文。**除非先用 Zink 把 GL 路由到 Vulkan，否则 OpenGL 游戏根本不会有 FG 效果。** 打开插件的 **Enable Zink** 开关，它会导出：

```bash
export __GLX_VENDOR_LIBRARY_NAME=mesa
export MESA_LOADER_DRIVER_OVERRIDE=zink
export GALLIUM_DRIVER=zink
```

Zink 生效后应用经由 Vulkan 呈现，lsfg-vk 才会介入，此时上面"原生 Vulkan"的建议完全适用（优先游戏内限帧，其次 MangoHud，同样注意层顺序陷阱）。注意 Zink 自身有 CPU 翻译开销，会吃掉一部分功耗收益——对某个 GL 游戏而言 FG 是否净收益，值得实测。

## 5. ⚠ 手改 `~/lsfg` 会被覆盖

`~/lsfg` 在**每一次**配置写入时都被从头重新生成（`py_modules/lsfg_vk/configuration.py:94-96`）。更糟的是，往返解析器只识别其生成的匹配列表中的键（`config_schema_generated.py:54-93`），像 `VKD3D_FRAME_RATE` 这样的未知键会被**静默丢弃**而不是报错。所以你的手改会在下一次动 UI 里任何一个滑块时消失，且没有任何提示。

**推荐做法——写进 Steam 启动选项**，插件永不重写它，而且天然按游戏隔离：

```
VKD3D_FRAME_RATE=30 ~/lsfg %command%
```

之所以可行：生成的脚本以 `exec "$@"` 结尾（`configuration.py:176`），外部设置的环境会被游戏继承。一般而言，按游戏的覆盖配置都应该放在这里。

## 6. 可选的长期修复

若希望 FrameCap 滑块在 DX12 上直接生效，让 `dxvk_frame_rate` 在输出 `DXVK_FRAME_RATE` 的同时也输出 `VKD3D_FRAME_RATE`：在 `scripts/generate_python_boilerplate.py` 中加一个特例，参照已有的 `enable_wsi` → `DXVK_HDR` 先例（生成逻辑见 133-137 行，解析逻辑见 78-83 行），然后运行 `just generate-schema`。约 6 行代码，无需改 schema，无需改 UI。

优先选环境变量而不是输出 `DXVK_CONFIG="dxvk.maxFrameRate=<n>"`：按第 4 节，环境变量会置 `has_user_override`、运行时不会被游戏覆盖，而 config 选项会。并且单行 `VKD3D_FRAME_RATE` 比带引号的复合值更容易在脚本解析器里往返读写。

注意：按 `CODEBUDDY.md` 的约定，**绝不能**直接编辑 `config_schema_generated.py` 或 `generatedConfigSchema.ts`——它们是派生文件。

## 7. 实测验证方法

用性能覆盖层 level 4（或 MangoHUD），在固定场景下读 **GPU 功耗（W）**。预期关系：

```
case 1 ≈ case 3 < case 5 < case 2 ≈ case 4
```

同时确认基础帧率与输出帧率的比值确实是 1:2。若不是，说明限帧器作用在了错误的层级（见第 4 节的 QAM 与 MangoHud 层顺序陷阱）。

**Mailbox 的指纹（单项诊断力最强的测试）**：开 Mailbox，同时读 presented FPS 与 GPU 功耗。若 b1 占主导，你会看到 **presented FPS 约 30 而 GPU 功耗很高**——付出全额帧生成成本，显示端零收益。该判据无歧义，且不需要做差分测量。

以上功耗与延迟数值均为基于架构机制的推断量级，需在自己的机器上实测确认。而 0.5、0.6 以及 b1/b2/b3 三节所述的**机制**是经源码核实的，**数值量级**则不是。

## 8. 帧生成完全不起作用时的排查

本节与上文所有内容不同：上文假定 FG **已经在工作**、讨论如何调优；本节针对的是"什么都没发生"。症状本身很有指示性：**不崩溃、无画面异常，就是没效果。**

按在 Steam Deck 上的实际命中率排序。

### 8.1 游戏是 32 位的 —— 最常见的硬性阻断

**这不是 DX11/DX12 的问题，而且很容易被误判成 API 问题。** DXVK 把 DX11 翻译成 Vulkan，因此 DX11 游戏**确实**是经 Vulkan 呈现的，是完全合法的 FG 目标。阻断点是位数，不是 API。64 位的 DX11 游戏工作正常。

插件只安装**一个 64 位**的 `liblsfg-vk.so`（`constants.py:14`，安装到 `.local/lib`，见 `installation.py:30`），层 JSON 的 `library_path` 指向它（`installation.py:165-168`）。32 位进程无法加载 64 位 `.so`，于是 loader 要么加载失败、要么直接跳过。项目未提供 32 位构建——根本没有可加载的东西。

上游把这一条列在基础排查清单的第二位，措辞异常直白（`docs/Troubleshooting.md:9`）：

> Ensure you are running a 64-bit game (try `PROTON_USE_WOW64=1`, but if it doesn't work then you're out of luck).

**解决办法**：插件的 **Enable WOW64** 开关（`shared_config.py:86-92`）会导出 `PROTON_USE_WOW64=1`，让 Proton 在 64 位 Linux 进程内运行 32 位 Windows 代码，从而使 64 位层能够加载。上游与插件自身的字段描述都建议**搭配 ProtonGE** 使用。这是唯一真正的解法，而上游也明确指出它经常不奏效。

**已确认受影响（用 `file` 实测，非推断）**：

```
Future Soldier DX11.exe:              PE32 executable (GUI) Intel 80386, 5 sections
METAL GEAR RISING REVENGEANCE.exe:    PE32 executable (GUI) Intel 80386, 6 sections
```

注意 `Future Soldier DX11.exe`——一个**DX11** 可执行文件，但仍然是 32 位的。这正是上文所说的陷阱：文件名把 API 摆在明面上，诱导人去怪 DX11，而真正的阻断点是 `PE32`。年代经验法则：2012 年及更早的 DX11 游戏中，相当多只有 32 位可执行文件。

**直接检查位数**：

```bash
file ~/.steam/steam/steamapps/common/<游戏>/*.exe
```

`PE32` = 32 位（问题就在这）。`PE32+` = 64 位（继续往下看）。

### 8.2 先确认层到底有没有被加载

两条命令即可区分"装错了"与"装对了但被进程拒绝"：

```bash
# 1. 层在系统层面可见吗？（需要 vulkan-tools）
vulkaninfo | grep -i VK_LAYER_LSFGVK_frame_generation
```

无输出 → 安装问题，重做安装步骤（`Troubleshooting.md:10-11`）。
有输出但游戏无效果 → 层本身没问题，失败发生在单个进程上，与 8.1 吻合。

```bash
# 2. 逐游戏的 loader 跟踪 —— 填进 Steam 启动选项
VK_LOADER_DEBUG=layer %command%
```

在 `<Loader>` 与 `<Device>` 之间查找 `VK_LAYER_LSFGVK_frame_generation`（`Troubleshooting.md:12-13`）。对 32 位游戏，通常会看到 loader 考虑了该层、随后以 ELF class / 架构不匹配错误失败。**这条报错就是 8.1 的确证。**

⚠ **上游关于 `LSFGVK_ENV=1` / `active_in` 的建议不适用于本插件。** `Troubleshooting.md:14-16` 描述的是当前 `develop`。所发布的 fp16 构建**没有** `active_in` 或 `LSFGVK_ENV` 支持——已核实在 `cb234bd` 中不存在。它靠把 `LSFG_PROCESS` 与 profile 名做匹配来激活（`src/config/config.cpp:153-156`，`name.first.ends_with(pair.first) || name.second == pair.first`），而插件写入的是 `export LSFG_PROCESS=<profile>`（`configuration.py:127`、`:156`）。请忽略上游文档的这一部分。

### 8.3 层已加载但 FG 不运行

出自 `Troubleshooting.md:20-27`。注意其中所有标注 `pacing_mode = none` 的条件在这里都**无条件适用**，因为所发布构建里这是**唯一**的模式（见 0.6 节）：

- **在游戏内显式开启垂直同步**，并**关闭 VRR**。这不是可选项。按 0.5 / b2 节，层依赖 FIFO 背压来同时完成节流与 pacing——游戏内部关掉 vsync 就等于抽掉了 FG 依赖的机制。
- **`ENABLE_GAMESCOPE_WSI=0`** —— 上游专门为 Gamescope/Steam Deck 列出。插件的 `enable_wsi` 已默认为 `False`（`shared_config.py:126-132`），因此本应已设置；请确认。
- **关闭游戏内的升采样选项**（DLSS、FSR 等）。
- **关闭其它 Vulkan 层**（VkBasalt、MangoHud）。插件有 `disable_vkbasalt` / `force_enable_vkbasalt` 开关。
- **黑屏 / 游戏起不来**：profile 选错了 GPU（`Troubleshooting.md:29-30`）。

### 8.4 在归咎于层之前，先确认游戏侧是否有帧率上限

某些引擎存在内部硬性上限，通常是因为物理或动画与时间步长绑定。**Metal Gear Rising 是已知的 60FPS 锁帧游戏。**

这在算术上很关键：60Hz 面板 + 倍率 2，需要 **30 基础帧**才能得到 60 输出帧。若游戏本来就输出 60 且无法更高，FG 没有任何余量可用——强行压到 30 基础帧只会得到 30→60，即你本来就已经拥有的 60 输出。在断定层坏掉之前，请先确认你实际的**基础**帧率。

**关于功耗，特别说明——不要假定此处 FG 更耗电。** 按 1.0 节，真正要比较的是 `30R + 30I` 对 `60R`，成败取决于是否 `I < R`。本文档早先的版本对本情形断言"还额外付了功耗"，那是缺乏依据的。可以确定的是：一款 Deck 能轻松驱动的 2013 年游戏 `R` 很**小**，这会把它推向 `I` 与 `R` 相当、收益消失的区间。而基础渲染帧率减半也让 CPU 工作量减半，作用方向相反。**对 MGR 的诚实结论是"最好也就是功耗持平"，且不实测无法确定**——而不是确定更耗电。

**该建议并不依赖功耗论证。** 此处 FG 仍然不值得开：

- **流畅度没有任何提升**——60Hz 面板上输出都是 60。仅此一条就足以定论。
- **延迟明确更差**——基础帧时间从 16.7ms 变成 33ms，**再加上**不可避免的约 16.7ms 插值滞留（见第 2 节第 2 项）。约 50ms 对原生的约 20~30ms。这一条有机制支撑，与功耗那条不同。
- **引入了原本不存在的画面瑕疵。**

输出帧率相同、延迟更差、多出瑕疵、功耗最好也只是持平。

### 8.5 ⚠ 叠加层在有 Vulkan 层时经常报错帧率

`Troubleshooting.md:34-37` —— 这是被承认的固有限制：

> If you are using performance overlays like Steam's built-in overlay, there is a good chance that they will not show the correct framerate. This is a known limitation of Vulkan layers […] there is little that can be done to fix this.

因此"帧率没上去"可能部分只是叠加层的显示假象，而非真的失败。两个更可靠的信号：

- **主观的运动流畅度。**
- **GPU 功耗。** 插值每帧要花 `I`，不可能免费。注意这里的基线是**相同基础帧率下层未生效**（`30R` 对 `30R + 30I`），而不是原生 60（`60R`）——后者是 1.0 节中另一个比较。相对"相同基础帧率"这个基线，层一旦生效功耗必然上升。因此：功耗平、帧率平 = 层确实没跑（转 8.1/8.2）。功耗升高、帧率平 = 层在跑但叠加层显示错误（或是第 1 节所述的 Mailbox b1 病态情形）。
