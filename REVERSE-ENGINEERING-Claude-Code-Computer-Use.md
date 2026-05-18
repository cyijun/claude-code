# Claude Code Computer Use 逆向分析

> 基于 `claude-code` 2.1.88 版本的源码和编译产物分析
> 分析时间：2026-05-18

---

## 概述

Claude Code 的 Computer Use 功能由三个私有 npm 包协作实现：

| 包名 | 语言/运行时 | 职责 |
|---|---|---|
| `@ant/computer-use-mcp` | TypeScript | MCP 协议层、状态机、权限管理、工具编排 |
| `@ant/computer-use-input` | Rust (NAPI) | 鼠标、键盘、前台应用检测 |
| `@ant/computer-use-swift` | Swift (NAPI) | 截图(SCContentFilter)、应用管理、TCC 权限 |

这三个包均未发布到公共 npm，仅通过 Anthropic 内部仓库分发。本文通过逆向工程还原了它们的完整接口。

---

## 1. `@ant/computer-use-mcp` — MCP 协议与编排层

### 1.1 导出 API

```typescript
// 主入口
export { buildComputerUseTools, createComputerUseMcpServer, bindSessionContext }
export { API_RESIZE_PARAMS, targetImageSize }

// 类型定义
export type {
  ComputerUseSessionContext,
  CuCallToolResult,
  CuPermissionRequest,
  CuPermissionResponse,
  ScreenshotDims,
  ComputerExecutor,
  CoordinateMode,          // 'pixels' | 'normalized'
}

// 常量
export { DEFAULT_GRANT_FLAGS }

// 子模块
import { getSentinelCategory } from '@ant/computer-use-mcp/sentinelApps'
import type {
  CuSubGates,
  FrontmostApp,
  InstalledApp,
  RunningApp,
  DisplayGeometry,
  ResolvePrepareCaptureResult,
  ScreenshotResult,
} from '@ant/computer-use-mcp/types'
```

### 1.2 `buildComputerUseTools(capabilities, coordinateMode, installedAppNames?)`

动态生成 MCP tool 定义数组。生成的 tools：

| Tool 名 | 功能描述 |
|---|---|
| `request_access` | 请求用户授权控制一组应用，返回 granted/denied apps 和 grant flags |
| `request_teach_access` | 请求"教模式"权限（teach mode），用于分步引导用户操作 |
| `screenshot` | 全屏截图（排除未授权应用），参数 `save_to_disk?: boolean` |
| `zoom` | 对上次截图的指定区域高分辨率放大，`region: [x, y, w, h]` |
| `left_click` | 左键点击，`coordinate: [x, y]` |
| `double_click` | 双击（通常选中单词） |
| `triple_click` | 三击（通常选中整行） |
| `right_click` | 右键（打开上下文菜单） |
| `middle_click` | 中键（滚轮点击） |
| `mouse_move` | 仅移动鼠标触发 hover 状态 |
| `left_mouse_down` | 按住左键不释放 |
| `left_mouse_up` | 释放左键 |
| `left_click_drag` | 拖拽操作，`coordinate` 为终点坐标 |
| `type` | 键盘输入文本，支持换行符 |
| `key` | 按单个键或组合键，`repeat?: number` |
| `hold_key` | 按住键持续一段时间，`duration: number`（秒） |
| `scroll` | 滚动，`scroll_amount: number` |
| `wait` | 等待指定秒数 |
| `cursor_position` | 获取当前鼠标坐标（相对于最近一次 screenshot） |
| `open_application` | 打开/激活应用，`app` 参数支持 bundleId 或显示名 |
| `switch_display` | 切换显示器，`display` 为显示器名称或 `"auto"` |
| `list_granted_applications` | 列出当前 session 已授权的应用 |
| `read_clipboard` | 读取剪贴板文本，需 `clipboardRead` grant |
| `write_clipboard` | 写入剪贴板文本，需 `clipboardWrite` grant |
| `computer_batch` | 批量执行多个 action，减少 model↔API round-trip |
| `teach_step` | 教模式单步：显示 tooltip + 执行 actions + 截图 |
| `teach_batch` | 教模式批量：N 步合成一次 tool call |

### 1.3 `bindSessionContext(adapter, coordinateMode, sessionContext)`

核心状态机绑定函数。返回 `dispatch(toolName, args)`，内部处理：

- **并发锁检查**：`checkCuLock` → `acquireCuLock`，防止多 session 同时控制电脑
- **应用白名单过滤**：只允许操作 `allowedApps` 中的应用
- **前台应用安全门**（frontmost gate）：点击/输入前检查目标应用是否前台
- **坐标转换**：`pixels` 或 `normalized` 模式下的坐标缩放
- **Display 自动解析**：根据窗口位置自动选择显示器，支持 pin/unpin
- **隐藏恢复**：操作前隐藏无关应用，turn 结束后自动 unhide
- **教模式**：`teach_step`/`teach_batch` 的 tooltip 渲染与截图回传

### 1.4 `createComputerUseMcpServer(adapter, coordinateMode)`

创建 MCP Server 实例：
- 注册 `ListTools` handler，返回动态生成的 tools 数组
- `CallTool` 是 **stub** —— 真正的 dispatch 被 Claude Code 的 `wrapper.tsx` 覆盖为进程内调用

---

## 2. `@ant/computer-use-input` — Rust/enigo 输入层

这是一个 **Rust NAPI 原生模块**（`.node`），封装了 [enigo](https://github.com/enigo-rs/enigo) 库，负责低级别 HID 输入和前台应用检测。

### 2.1 导出接口

```typescript
export interface ComputerUseInput {
  isSupported: boolean
  // 非 macOS 平台时 isSupported = false
}

export interface ComputerUseInputAPI {
  // ── 键盘 ──
  keys(parts: string[]): Promise<void>
  // 发送组合键，如 ['command', 'v']

  key(keyName: string, action: 'press' | 'release'): Promise<void>
  // 单个按键的按下/释放

  typeText(text: string): Promise<void>
  // 逐字符输入（直接发 HID，不经过剪贴板）

  // ── 鼠标 ──
  moveMouse(x: number, y: number, animate: boolean): Promise<void>
  // 移动鼠标。animate=true 时带动画轨迹

  mouseButton(
    button: 'left' | 'right' | 'middle',
    action: 'press' | 'release' | 'click',
    clickCount?: 1 | 2 | 3
  ): Promise<void>
  // 鼠标按键操作，支持单击/连击

  mouseLocation(): Promise<{ x: number; y: number }>
  // 获取当前鼠标物理坐标

  mouseScroll(amount: number, axis: 'vertical' | 'horizontal'): Promise<void>
  // 滚轮滚动

  // ── 应用检测 ──
  getFrontmostAppInfo(): { bundleId: string; appName: string } | null
  // 获取当前前台应用信息
}
```

### 2.2 技术实现细节

- `keys()` / `key()` 内部通过 `dispatch2::run_on_main` 把 enigo 工作发到 **DispatchQueue.main**，然后阻塞 tokio worker 等待 channel 返回
- 在 Node/bun 的 libuv 环境下，主队列不会自动 drain，因此 executor.ts 中所有 key 调用都必须包裹在 `drainRunLoop()` 中泵送 `CFRunLoop`
- `typeText()` **不**走主队列，直接同步调用（或内部已处理队列）
- `getFrontmostAppInfo()` 被用于 frontmost gate：如果目标应用不在前台，则拒绝点击/输入操作（防止误操作）

---

## 3. `@ant/computer-use-swift` — Swift 原生层

这是一个 **Swift NAPI 原生模块**，负责 macOS 特有的高级功能：SCContentFilter 截图、应用生命周期管理、TCC 权限检查。

### 3.1 导出接口

```typescript
export interface ComputerUseAPI {
  // ── 内部 RunLoop 泵送 ──
  _drainMainRunLoop(): void
  // 手动泵送 RunLoop.main，让 @MainActor 方法 resolve

  // ── TCC 权限 ──
  tcc: {
    checkAccessibility(): boolean      // 辅助功能权限
    checkScreenRecording(): boolean    // 屏幕录制权限
  }

  // ── 显示器信息 ──
  display: {
    getSize(displayId?: number): DisplayGeometry
    // logical width/height + scaleFactor

    listAll(): DisplayGeometry[]
    // 枚举所有连接的显示器
  }

  // ── 应用管理 ──
  apps: {
    prepareDisplay(
      allowlistBundleIds: string[],
      surrogateHost: string,
      displayId?: number
    ): Promise<{ activated: string | null; hidden: string[] }>
    // 隐藏非白名单应用，激活白名单应用
    // surrogateHost（终端 bundleId）被豁免隐藏

    previewHideSet(
      allowlistBundleIds: string[],
      displayId?: number
    ): Promise<Array<{ bundleId: string; displayName: string }>>
    // 预览本次操作会隐藏哪些应用

    findWindowDisplays(
      bundleIds: string[]
    ): Promise<Array<{ bundleId: string; displayIds: number[] }>>
    // 查找应用窗口分布在哪些显示器

    listInstalled(): Promise<InstalledApp[]>
    // 枚举所有已安装应用（@MainActor，需 drainRunLoop）

    listRunning(): Promise<RunningApp[]>
    // 枚举运行中应用

    open(bundleId: string): Promise<void>
    // 通过 NSWorkspace 打开应用

    appUnderPoint(
      x: number, y: number
    ): Promise<{ bundleId: string; displayName: string } | null>
    // 获取指定坐标下的应用

    iconDataUrl(path: string): string | null
    // 获取应用图标 DataURL（用于权限对话框展示）

    unhide(bundleIds: string[]): Promise<void>
    // 恢复被隐藏的应用窗口
  }

  // ── 截图 ──
  screenshot: {
    captureExcluding(
      allowedBundleIds: string[],
      jpegQuality: number,
      targetW: number,
      targetH: number,
      displayId?: number
    ): Promise<ScreenshotResult>
    // 全屏截图，只保留白名单应用窗口（@MainActor）

    captureRegion(
      allowedBundleIds: string[],
      x: number, y: number,
      w: number, h: number,
      outW: number, outH: number,
      jpegQuality: number,
      displayId?: number
    ): Promise<{ base64: string; width: number; height: number }>
    // 区域截图（@MainActor）
  }

  // ── 复合操作 ──
  resolvePrepareCapture(
    allowedBundleIds: string[],
    surrogateHost: string,
    jpegQuality: number,
    targetW: number,
    targetH: number,
    displayId?: number,
    autoResolve?: boolean,
    doHide?: boolean
  ): Promise<ResolvePrepareCaptureResult>
  // 解析显示器 + 准备隐藏 + 预计算截图尺寸（@MainActor）
}
```

### 3.2 相关类型定义

```typescript
interface DisplayGeometry {
  displayId: number
  width: number       // logical width（如 1920）
  height: number      // logical height（如 1080）
  scaleFactor: number // 物理像素倍率（如 2.0 for Retina）
}

interface ScreenshotResult {
  base64: string
  width: number
  height: number
}

interface InstalledApp {
  bundleId: string
  displayName: string
  path: string
  iconDataUrl?: string
}

interface RunningApp {
  bundleId: string
  displayName: string
}

interface ResolvePrepareCaptureResult {
  // 包含：目标显示器、隐藏的应用列表、截图尺寸等
}

interface FrontmostApp {
  bundleId: string
  displayName: string
}
```

### 3.3 关键技术细节

#### SCContentFilter 截图过滤

`captureExcluding` 使用 macOS `SCContentFilter` API 实现**原生应用级截图过滤**。被排除的 bundleId 对应的应用窗口不会出现在截图中，这比普通截图后裁剪更安全、性能更好。

> ⚠️ **已知 bug**: Swift 0.2.1 的 `captureExcluding` 实际上接收的是 **allow-list**（允许出现在截图中的应用），但方法名叫 excluding。Claude Code 在 `executor.ts` 中通过 `withoutTerminal()` 处理了这个问题（apps#30355）。

#### @MainActor 异步方法

以下四个方法是 `@MainActor` 的，异步 dispatch 到 `DispatchQueue.main`：
- `captureExcluding`
- `captureRegion`
- `apps.listInstalled`
- `resolvePrepareCapture`

在 Node/bun 环境下必须配合 `drainRunLoop()` 使用，否则 promise 会永久挂起。

#### prepareDisplay 的隐藏机制

```
1. 传入 allowlistBundleIds（用户授权的应用）
2. surrogateHost（终端 emulator 的 bundleId）被加入豁免列表
3. 所有不在 allowlist 也不在豁免列表的应用窗口被隐藏
4. 尝试激活 allowlist 中的第一个应用
5. 返回 { activated: 被激活的应用, hidden: 被隐藏的应用 bundleIds }
```

---

## 4. 完整调用链路

```
用户输入 → 模型生成 tool_use
    ↓
Claude Code: wrapper.tsx 的 .call() override
    ↓
@ant/computer-use-mcp: bindSessionContext() 返回的 dispatch(toolName, args)
    ↓
    ├─ 权限检查（cu_lock, frontmost_gate, allowlist）
    ├─ 坐标转换（pixels / normalized）
    ├─ Display 解析（auto-resolve / pin）
    └─ prepareDisplay（隐藏无关应用）
    ↓
hostAdapter.ts: ComputerUseHostAdapter
    ↓
executor.ts: createCliExecutor()
    ↓
    ├─ 鼠标/键盘/前台应用 → @ant/computer-use-input (Rust/enigo)
    │   └─ enigo → CGEvent / HID
    │
    └─ 截图/应用管理/TCC → @ant/computer-use-swift (Swift)
        ├─ SCContentFilter → ScreenCaptureKit
        ├─ NSWorkspace → 应用枚举/打开
        └─ TCC API → 权限检查
```

---

## 5. 与开源方案的对比

| 维度 | Claude Code（私有） | 开源替代（如 AB498/computer-control-mcp） |
|---|---|---|
| 截图过滤 | `SCContentFilter` 原生过滤 | 通常全屏截图或窗口截图 |
| 输入模拟 | Rust enigo 直接发 HID 事件 | PyAutoGUI / Playwright |
| 坐标系统 | 支持物理/逻辑像素自动转换 | 多为屏幕像素坐标 |
| 权限管理 | TCC 集成 + 应用级 allowlist | 需手动给终端授权 |
| 隐藏窗口 | 操作前自动隐藏无关应用 | 一般不支持 |
| 跨平台 | 仅 macOS | Windows/macOS/Linux 都有 |
| 教模式 | 内置 teach_step/teach_batch | 无 |
| 多显示器 | 自动解析 + pin 切换 | 大多不支持 |

---

## 6. 附录：关键源码位置

| 文件 | 说明 |
|---|---|
| `src/utils/computerUse/mcpServer.ts` | 创建 MCP server，注册 ListTools |
| `src/utils/computerUse/setup.ts` | 动态生成 MCP config 和 allowedTools |
| `src/utils/computerUse/wrapper.tsx` | `.call()` override，绑定 session context |
| `src/utils/computerUse/executor.ts` | `ComputerExecutor` 实现，桥接两个原生模块 |
| `src/utils/computerUse/hostAdapter.ts` | `ComputerUseHostAdapter` 单例工厂 |
| `src/utils/computerUse/inputLoader.ts` | `@ant/computer-use-input` 懒加载 |
| `src/utils/computerUse/swiftLoader.ts` | `@ant/computer-use-swift` 懒加载 |
| `src/utils/computerUse/drainRunLoop.ts` | CFRunLoop 泵送机制 |
| `src/utils/computerUse/gates.ts` | GrowthBook feature flag 控制 |
| `src/utils/computerUse/common.ts` | 常量定义（capabilities、host bundleId） |
| `src/services/mcp/client.ts:925-943` | 进程内 Computer Use MCP 服务器连接 |

---

*本文仅供学习研究使用。*
