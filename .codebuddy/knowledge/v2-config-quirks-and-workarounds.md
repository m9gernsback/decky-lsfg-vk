# lsfg-vk v2 配置语义与插件 Workaround 行为（2026-10 核查）

本会话核查的结论汇总。源码引用基于本仓库 HEAD `eee1dbf`；上游语义引用 lsfg-vk 官方文档与 DXVK/vkd3d-proton 源码结论（详见同目录 `frame-generation-power-and-latency.md` 第 4 节）。

## 1. v2 profile 的两个 Quirks 开关

来源：https://lsfg-vk.dev/docs/configuration/configuration-options

### `override_present_mode`（默认 `true`）

- 作用：lsfg-vk 强制把游戏的 present mode 改写为 **Vsync/FIFO**——v2 唯一的 pacing mode 是 `vsync`，它依赖 FIFO 队列阻塞来做帧 pacing，这是补帧不错位的前提
- 上游原文警告："Do not turn this option off, unless you know what you are doing"
- 改为 `false`：帧 pacing 失效，表现为卡顿、帧重复，甚至补帧完全不生效
- **结论：保持默认 `true`**

### `preserve_swapchain_image_count`（默认 `false`）

- 作用：默认 lsfg-vk 会调整 swapchain 图像数量以容纳生成帧；开启后保持游戏自己申请的数量
- 改为 `true` 的好处：修复对 swapchain 图像数有硬假设的 Vulkan 游戏的**崩溃**
- 代价：其他游戏会出现**卡顿和 slowdown**（交换链缓冲不足阻塞补帧管线）
- **结论：默认 `false`；仅当某游戏开补帧即崩溃时，在该游戏的 profile 里单独打开**（插件支持 `active_in` 按 AppID/可执行名匹配多 profile，正适合按游戏设置）

插件侧落点：`py_modules/lsfg_vk/config_schema.py:27-28`（默认值）、`src/components/ConfigurationSection.tsx:19-22`（UI，目前是硬编码英文，未走 i18n）。

## 2. Base FPS Cap 的真实覆盖面

插件实现：`wrapper_service.py:177-190` 注入 `DXVK_CONFIG="dxvk.maxFrameRate = N"`（N ∈ 0–60，0=关）。

| API | `dxvk.maxFrameRate` 是否生效 |
|---|---|
| DX9 / DX11（DXVK） | ✅ |
| DX12（vkd3d-proton） | ❌（`dxvk.*` 不到 DX12；但 **`dxgi.maxFrameRate`** 能经 DXGI 虚表 `IDXGIVkSwapChain2::SetTargetFrameRate` 传导到 DX12——见旧文档第 4 节源码追踪；插件目前不输出 `dxgi.*`） |
| 原生 Vulkan / OpenGL | ❌ |

环境变量处置：

- **`DXVK_FRAME_RATE`**：插件视其为遗留变量，wrapper 启动时 `unset`（`wrapper_service.py:39,153`），前端清理启动选项时也会剔除（`steamLaunchOptions.ts:18`）。合理性有双重依据：① 避免与 `dxvk.maxFrameRate` 重复限帧；② 该变量在 **DXVK v3.0（随 Proton 10.0-3）已被上游删除**，写了也没人读。用户手写 `DXVK_FRAME_RATE=30` 会被静默清除，应改用插件滑块
- **`VKD3D_FRAME_RATE`**：插件完全不碰（不在 MANAGED_ENV_KEYS，清理逻辑也不涉及），可放心手写，是目前 DX12 限帧的补充手段；且它置 vkd3d `has_user_override`，比 `dxgi.maxFrameRate` 路径更抗覆盖

## 3. Multiplier "Off" 与 FrameCap 相互独立

- UI 上的 "Off" = `multiplier = 1`（`FpsMultiplierControl.tsx:34-49`），即生成 0 个补充帧
- FrameCap 注入条件只有一个：该 AppID 的 `dxvkFrameRate > 0`（`wrapper_service.py:177`），**不检查 multiplier**
- 因此 "Off + FrameCap=30" 对 DX9/DX11 游戏是纯限帧器：真实帧压到 30，无补帧，输出 30 fps。这是合法用法
- 若想完全不加载补帧层，应把游戏从 profile 的 `active_in` 移除（仅靠 multiplier=1 时 layer 仍可能强制 FIFO）

## 4. Disable Steam Deck Mode（`SteamDeck=0`）的用意

- `SteamDeck=1` 由 Gamescope 会话注入，游戏/Proton 据此启用 Deck 专属模式：锁定图形选项、强制手柄 UI/字体缩放、部分游戏强制 vsync 或帧率上限
- 补帧需要解锁图形选项（关游戏内 vsync、解锁帧率、调分辨率），Deck 模式构成障碍；插件开关（默认 `false`，`wrapper_service.py:54`；修复提交 `e744826` 保证默认保留）打开时注入 `SteamDeck=0`（`wrapper_service.py:164-165`）
- 坑：`SteamDeck` 在前端清理清单里（`steamLaunchOptions.ts:16`），用户手写进启动选项的 `SteamDeck=0` 会在插件移除注入时被一并清掉——应使用 UI 开关而非手写

## 5. OpenGL 游戏：必须经 Zink

- lsfg-vk 是 Vulkan layer，原生 OpenGL 管线里无 Vulkan 可 hook，**不可能**补帧
- 插件开关 "Force Zink for OpenGL Games"（`enableZink`，`wrapper_service.py:168-176`）注入 `MESA_LOADER_DRIVER_OVERRIDE=zink` 等，走 OpenGL→Zink→Vulkan，使 layer 能挂载
- 官方 i18n 文案自带警告："May cause crashes or freezes with some games"；Zink 多一层翻译会略降基础帧；需重启游戏

## 6. 用户游戏清单 API 调查结果（PC 版）

来源：PCGamingWiki + Steam 商店页，2026-10 核查。

| 游戏 | API | FrameCap 直用？ |
|---|---|---|
| ACE COMBAT 7 | DX11（UE4） | ✅ |
| Call of Duty: Advanced Warfare | DX11 | ✅ |
| Devil May Cry 5 | **DX12 默认**；改 `dmc5config.ini` 的 `TargetPlatform=DirectX11` 可切 DX11 | ⚠️ 默认需 `VKD3D_FRAME_RATE` |
| Fate/EXTELLA LINK | DX11 | ✅ |
| Hades | DX11 + Vulkan（2020-10 加入）；Proton 默认走 DX11 | ✅（勿加 `-vulkan`） |
| MGS RISING: REVENGEANCE | DX9（9.0c，32 位） | ✅ |
| MGSV: GROUND ZEROES | DX11 | ✅ |
| MGSV: THE PHANTOM PAIN | DX11 | ✅ |
| NieR: Automata | DX11 | ✅ |
| SD GUNDAM BATTLE ALLIANCE | DX11（商店页推荐配置提及 DX12 API 路径，可能可选） | ✅ |
| Ghost Recon Future Soldier | DX9 + DX11 双模式 | ✅ |
| 嗜血印 Bloody Spell | DX11（UE4；商店页最低 DX10 系虚标） | ✅ |

12 款中 11 款默认 DX9/DX11，唯 DMC5 默认 DX12。

## 7. 待办/缺口（核查中发现，未修）

- i18n 残留死键：`CONFIG_PRESENT_MODE*`（mailbox 时代）仍在 `defaults/i18n/*.json` 与 `src/i18n/languages.json`，代码已无引用
- `ConfigurationSection.tsx` 四个控件硬编码英文，未走 i18n
- 插件 FrameCap 只输出 `dxvk.maxFrameRate`，未利用 `dxgi.maxFrameRate` 这条可覆盖 DX12 的路径
