# 深浅色主题跟随系统 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修复 V2Next userscript 的深浅色主题：默认跟随系统、`system` 可持久化、系统主题变化时自动切换，且快捷按钮一点即锁定对立色。

**Architecture:** 在 `packages/core` 明确区分 preference（`light|dark|system`，持久化）与 resolved（`light|dark`，仅 DOM/内存）。启动 bootstrap 读缓存否则 `system` 并尽早 apply；App 生命周期订阅 `matchMedia` change 与 `storage` 跨标签同步；设置页三选一；切换按钮/原站夜间入口写入 light/dark。PC 与 Mobile 删除本地重复存储实现，统一调 core。

**Tech Stack:** Vue 3（Options API）、TypeScript 5.0.4、Vite + vite-plugin-monkey userscript、pnpm workspace、`@v2next/core` 源码直出（无独立测试框架 → 核心逻辑用浏览器控制台/手工清单验收，构建用 `pnpm --filter pc run build-pc` / mobile build）。

**Spec:** `docs/superpowers/specs/2026-07-20-theme-mode-system-follow-design.md`（commit `fecc5b4`）

## Global Constraints

- 持久化**只写 preference**，禁止把 resolve 结果写回 `config.themeMode` / localStorage
- 无有效缓存默认 `system`；非法值归一为 `system`（不是 `light`）
- 老用户已有 `light`/`dark` 缓存不迁移
- 不引入 VueUse / next-themes 等新依赖
- 不重做配色，沿用 `html.dark`
- 继续覆盖/隐藏原站 Night 入口，不改原站 cookie 逻辑
- 系统 `matchMedia` change **仅** preference === `'system'` 时 re-apply
- 跨标签监听 `storage`（`v2next-theme-mode` 与 `v2ex-config`）
- 提交前按仓库习惯评估是否构建并提交 `packages/*/dist/*`（上次主题提交带了 dist；若构建失败可只交源码并在 commit message 标明）

## File map

| File | Responsibility |
|---|---|
| `packages/core/types.ts` | `Config.themeMode` → 含 `system` |
| `packages/core/core.ts` | preference/resolve/存储/监听/默认值；`initConfig` 不再 resolve 固化 |
| `packages/pc/src/main.ts` | bootstrap 用 core；删本地 get/set |
| `packages/mobile/src/main.ts` | 同上 |
| `packages/pc/src/App.vue` | watch/apply 去固化；按钮；系统监听；storage |
| `packages/mobile/src/pages/App.vue` | 同上 + night 链接 |
| `packages/pc/src/components/Modal/SettingModal.vue` | 三选一 + 当前解析 |
| `packages/mobile/src/pages/Setting.vue` | 同上 |

---

### Task 1: Core 类型 + preference/resolve/存储 API

**Files:**
- Modify: `packages/core/types.ts:78`
- Modify: `packages/core/core.ts`（约 619–793：主题 API + `getDefaultConfig`；约 467–482：`initConfig`）

**Interfaces:**
- Produces:
  - `export type ThemeResolved = 'light' | 'dark'`
  - `export type ThemePreference = ThemeResolved | 'system'`
  - `export type ThemeMode = ThemeResolved`（保留别名，表示 resolved）
  - `normalizeThemePreference(input, fallback?: ThemePreference): ThemePreference`
  - `resolveThemeMode(preference, originNight?: boolean): ThemeResolved`
  - `getStoredThemePreference(userKey?: string): ThemePreference | undefined`（可保留 `getStoredThemeMode` 作为同实现别名）
  - `setStoredThemePreference(pref: ThemePreference, userKey?: string): void`（可保留 `setStoredThemeMode` 别名）
  - `applyThemeMode(preference, originNight?: boolean): boolean`（返回 isNight；**不写 storage**）
  - `getDefaultConfig().themeMode` 默认 `'system'`

- [ ] **Step 1: 改 `Config.themeMode` 类型**

在 `packages/core/types.ts` 将：

```ts
themeMode: 'light' | 'dark'//主题模式
```

改为：

```ts
themeMode: 'light' | 'dark' | 'system'//主题偏好（持久化；非 resolved）
```

- [ ] **Step 2: 重写 core 主题类型与 normalize/resolve**

在 `packages/core/core.ts` 将现有主题相关块（约 619–745）替换为语义清晰实现。关键点（完整写入文件时保持周围 `DefaultVal` / `DEFAULT_MAX_REPLY_COUNT_LIMIT` 不变）：

```ts
export type ThemeResolved = 'light' | 'dark'
export type ThemePreference = ThemeResolved | 'system'
/** @deprecated 语义上表示 resolved；新代码优先用 ThemeResolved */
export type ThemeMode = ThemeResolved
export type LegacyThemeMode = ThemePreference
export const THEME_CACHE_KEY = 'v2next-theme-mode'
export const THEME_USER_KEY = 'v2next-theme-user-key'
export const THEME_MEDIA_QUERY = '(prefers-color-scheme: dark)'

export function getSystemThemeMode(originNight = false): ThemeResolved {
  if (typeof window !== 'undefined' && typeof window.matchMedia === 'function') {
    return window.matchMedia(THEME_MEDIA_QUERY).matches ? 'dark' : 'light'
  }
  return originNight ? 'dark' : 'light'
}

/** 归一化为 preference；非法 → fallback（默认 system） */
export function normalizeThemePreference(
  themeMode: ThemePreference | string | undefined | null,
  fallback: ThemePreference = 'system'
): ThemePreference {
  if (themeMode === 'light' || themeMode === 'dark' || themeMode === 'system') {
    return themeMode
  }
  if (fallback === 'light' || fallback === 'dark' || fallback === 'system') {
    return fallback
  }
  return 'system'
}

/**
 * 历史名：旧实现会把 system resolve 成 light/dark（错误）。
 * 现改为 preference 归一，保留 system。调用方若需要 resolved 请用 resolveThemeMode。
 */
export function normalizeThemeMode(
  themeMode: ThemePreference | string | undefined | null,
  fallbackMode?: ThemePreference | string
): ThemePreference {
  const fallback = normalizeThemePreference(fallbackMode, 'system')
  return normalizeThemePreference(themeMode, fallback)
}

export function resolveThemeMode(
  themeMode: ThemePreference | string | undefined | null = 'system',
  originNight = false
): ThemeResolved {
  const preference = normalizeThemePreference(themeMode, 'system')
  if (preference === 'system') return getSystemThemeMode(originNight)
  return preference
}

export function resolveThemeNight(
  themeMode: ThemePreference | string | undefined | null = 'system',
  originNight = false
) {
  return resolveThemeMode(themeMode, originNight) === 'dark'
}
```

- [ ] **Step 3: 存储 API 接受 `system`**

替换 `getStoredThemeMode` / `setStoredThemeMode`：

```ts
function parseStoredPreference(mode: any): ThemePreference | undefined {
  return mode === 'light' || mode === 'dark' || mode === 'system' ? mode : undefined
}

export function getStoredThemePreference(userKey = 'default'): ThemePreference | undefined {
  try {
    const raw = localStorage.getItem('v2ex-config')
    const configMap = raw ? JSON.parse(raw) : {}
    const userMode = parseStoredPreference(configMap?.[userKey]?.themeMode)
    if (userMode) return userMode
    if (userKey !== 'default') {
      const defaultMode = parseStoredPreference(configMap?.default?.themeMode)
      if (defaultMode) return defaultMode
    }
    return parseStoredPreference(localStorage.getItem(THEME_CACHE_KEY))
  } catch (e) {
    return parseStoredPreference(localStorage.getItem(THEME_CACHE_KEY))
  }
}

/** 兼容旧名 */
export function getStoredThemeMode(userKey = 'default'): ThemePreference | undefined {
  return getStoredThemePreference(userKey)
}

export function setStoredThemePreference(mode: ThemePreference, userKey = 'default') {
  const preference = normalizeThemePreference(mode, 'system')
  try {
    const raw = localStorage.getItem('v2ex-config')
    const configMap = raw ? JSON.parse(raw) : {}
    const userConfig = configMap?.[userKey] ?? {}
    userConfig.themeMode = preference
    configMap[userKey] = userConfig
    const defaultConfig = configMap?.default ?? {}
    defaultConfig.themeMode = preference
    configMap.default = defaultConfig
    localStorage.setItem('v2ex-config', JSON.stringify(configMap))
  } catch (e) {
    // localStorage 解析失败时至少保留轻量缓存。
  }
  localStorage.setItem(THEME_CACHE_KEY, preference)
  localStorage.setItem(THEME_USER_KEY, userKey)
}

export function setStoredThemeMode(mode: ThemePreference, userKey = 'default') {
  setStoredThemePreference(mode, userKey)
}
```

- [ ] **Step 4: `resetOriginThemeMode` / `applyThemeMode` 使用 resolved，不碰 storage**

```ts
export function resetOriginThemeMode(
  themeMode: ThemePreference | string | undefined | null = 'system'
): ThemeResolved {
  const mode = resolveThemeMode(themeMode, false)
  if (typeof document === 'undefined') return mode

  const root = document.documentElement
  root.classList.remove('Night')
  root.dataset.v2nextTheme = mode
  root.style.colorScheme = mode

  if (document.body) {
    document.body.classList.remove('Night')
    document.body.dataset.v2nextOriginTheme = 'light'
  }

  const themeColor = document.querySelector('meta[name="theme-color"]') as HTMLMetaElement | null
  if (themeColor) {
    themeColor.setAttribute('content', mode === 'dark' ? '#18222d' : '#ffffff')
  }

  return mode
}

export function applyThemeMode(
  themeMode: ThemePreference | string | undefined | null = 'system',
  originNight = false
): boolean {
  const mode = resetOriginThemeMode(
    // resolve 一次，避免 reset 内再读 system 时与 originNight 不一致
    resolveThemeMode(themeMode, originNight)
  )
  // 注意：resetOriginThemeMode 入参若已是 light|dark，resolve 仍返回自身
  const isNight = mode === 'dark'
  if (typeof document !== 'undefined') {
    document.documentElement.classList.toggle('dark', isNight)
  }
  if (typeof window !== 'undefined') {
    window.isNight = isNight
  }
  return isNight
}
```

说明：`applyThemeMode` 入参是 **preference 或已 resolved 的 light/dark**；`resolveThemeMode` 对 light/dark 原样返回。不要在 `applyThemeMode` 内调用 `setStored*`。

- [ ] **Step 5: 默认配置与 `initConfig` 使用 preference**

`getDefaultConfig`：

```ts
themeMode: 'system',
// ...
config.themeMode = normalizeThemePreference(config.themeMode, 'system')
```

`functions.initConfig` 内将：

```ts
window.config.themeMode = normalizeThemeMode(window.config.themeMode)
```

确保此时 `normalizeThemeMode` 已是 preference 语义（保留 system）。若本地配置已有 light/dark，原样保留。

- [ ] **Step 6: 控制台快速自检（无自动化测试时）**

在任意已加载页面 DevTools，或临时在 core 旁用 node 无法测 matchMedia，故用构建后脚本页，或在修改后 `pnpm --filter pc run build-pc` 前先用 TS 逻辑手读：

手工断言（实现者在浏览器控制台，先确保能 import 到这些函数；若 userscript 未暴露，则在临时 dev 页验证，或读代码 walkthrough）：

| 输入 | `normalizeThemePreference` | `resolveThemeMode`（假设系统 dark） |
|---|---|---|
| `undefined` | `system` | `dark` |
| `'system'` | `system` | `dark` |
| `'light'` | `light` | `light` |
| `'dark'` | `dark` | `dark` |
| `'foo'` | `system` | `dark` |

`getStoredThemePreference`：当 `localStorage['v2next-theme-mode']='system'` 时应返回 `'system'`（旧实现会返回 `undefined`）。

- [ ] **Step 7: Commit**

```bash
git add packages/core/types.ts packages/core/core.ts
git commit -m "$(cat <<'EOF'
feat(core): theme preference 支持 system 并修正 resolve 边界

持久化 light|dark|system；resolve 仅用于 DOM。默认 system。
EOF
)"
```

---

### Task 2: Core `subscribeSystemThemeChange`

**Files:**
- Modify: `packages/core/core.ts`（紧接 applyThemeMode 之后）

**Interfaces:**
- Consumes: `THEME_MEDIA_QUERY`, `getSystemThemeMode`
- Produces: `subscribeSystemThemeChange(listener: (resolved: ThemeResolved) => void): () => void`

- [ ] **Step 1: 实现订阅**

```ts
export function subscribeSystemThemeChange(
  listener: (resolved: ThemeResolved) => void
): () => void {
  if (typeof window === 'undefined' || typeof window.matchMedia !== 'function') {
    return () => {}
  }
  const mql = window.matchMedia(THEME_MEDIA_QUERY)
  const handler = (event?: MediaQueryListEvent) => {
    const resolved: ThemeResolved = event
      ? (event.matches ? 'dark' : 'light')
      : getSystemThemeMode()
    listener(resolved)
  }
  if (typeof mql.addEventListener === 'function') {
    mql.addEventListener('change', handler)
    return () => mql.removeEventListener('change', handler)
  }
  // Safari 旧 API
  mql.addListener(handler as any)
  return () => mql.removeListener(handler as any)
}
```

- [ ] **Step 2: 确认不在 core 内读取 config**

listener 由 App 传入；core **不**判断 preference === system（由调用方判断），保持 API 纯净。App 内：

```ts
this._unsubSystemTheme = subscribeSystemThemeChange(() => {
  if (this.config.themeMode !== 'system') return
  this.applyThemeByConfig()
  this.updateThemeToggleIcon?.()
})
```

- [ ] **Step 3: Commit**

```bash
git add packages/core/core.ts
git commit -m "feat(core): 增加 subscribeSystemThemeChange"
```

---

### Task 3: PC / Mobile bootstrap 改用 core 默认 system

**Files:**
- Modify: `packages/pc/src/main.ts:1-90`（主题 bootstrap + `applyConfiguredThemeMode`）
- Modify: `packages/mobile/src/main.ts:1-120`（同上）
- Modify: `packages/mobile/src/main.ts` 中其它 `normalizeThemeMode` 写回点（约 875）

**Interfaces:**
- Consumes: `getStoredThemePreference`, `setStoredThemePreference`, `applyThemeMode`, `normalizeThemePreference` from `@v2next/core`
- Produces: 启动时 preference 正确 apply；无缓存 → system

- [ ] **Step 1: PC `main.ts` 删除本地 get/set，改 bootstrap**

删除文件顶部本地 `THEME_CACHE_KEY` / `getStoredThemeMode` / `setStoredThemeMode` 重复实现（约 17–53 行）。

import 改为：

```ts
import {
  applyThemeMode,
  DefaultUser,
  DefaultVal,
  functions,
  getDefaultConfig,
  getDefaultPost,
  getStoredThemePreference,
  normalizeThemePreference,
  setStoredThemePreference,
} from "@v2next/core";
```

（若工程从 `@v2next/core/core.ts` 导入，保持与现有一致路径。）

bootstrap：

```ts
const bootstrapThemePreference = getStoredThemePreference() ?? 'system'
const bootstrapIsNight = applyThemeMode(bootstrapThemePreference, false)
if (!document.body) {
  document.addEventListener('DOMContentLoaded', () => {
    applyThemeMode(bootstrapThemePreference, false)
  }, {once: true})
}
```

`applyConfiguredThemeMode`：

```ts
const applyConfiguredThemeMode = () => {
  const storedMode = getStoredThemePreference(getUserKey())
  if (storedMode) {
    window.config.themeMode = storedMode
  } else {
    window.config.themeMode = normalizeThemePreference(window.config.themeMode, 'system')
  }
  const preference = normalizeThemePreference(window.config.themeMode, 'system')
  window.config.themeMode = preference
  window.isNight = applyThemeMode(preference, false)
  // 仅在「已有明确 preference」时写回；若本就是默认 system 且无缓存，可写 system 以便后续读取一致
  setStoredThemePreference(preference, getUserKey())
  return preference
}
```

注意：首次无缓存时写入 `system` 是正确的（preference 不是 resolved）。**禁止** `setStoredThemePreference(resolveThemeMode(...))`。

- [ ] **Step 2: Mobile `main.ts` 做同样修改**

与 PC 对称：删本地存储函数；bootstrap `?? 'system'`；`applyConfiguredThemeMode` 同上。

检查 `packages/mobile/src/main.ts` 约 875 行：

```ts
window.config.themeMode = normalizeThemeMode(window.config.themeMode)
```

确认此时 normalize 保留 system（Task 1 后已成立）。若该处意图是 apply DOM，应改为：

```ts
window.config.themeMode = normalizeThemePreference(window.config.themeMode, 'system')
window.isNight = applyThemeMode(window.config.themeMode, false)
```

- [ ] **Step 3: 手工验收 bootstrap**

1. DevTools → Application → Local Storage：删除 `v2next-theme-mode`，并编辑 `v2ex-config` 去掉各 user 的 `themeMode`（或整 key 清掉后仅测主题）。
2. 将 OS/浏览器设为深色，刷新 v2ex 页（脚本生效后）：
   - `document.documentElement.classList.contains('dark') === true`
   - `localStorage.getItem('v2next-theme-mode') === 'system'`（若 applyConfigured 已跑）
   - **不得**为 `'dark'` 却声称跟随系统却无法再跟系统——preference 必须是 `system`
3. OS 浅色重复：应为无 `dark` class，preference `system`。
4. 预设 `localStorage.setItem('v2next-theme-mode','light')` 且 config 内 light：刷新后保持浅色（老用户）。

- [ ] **Step 4: Commit**

```bash
git add packages/pc/src/main.ts packages/mobile/src/main.ts
git commit -m "$(cat <<'EOF'
fix: bootstrap 默认跟随系统并复用 core 主题存储

删除 PC/Mobile 重复 get/set，无缓存时 preference=system。
EOF
)"
```

---

### Task 4: PC / Mobile App — watch / apply 去固化

**Files:**
- Modify: `packages/pc/src/App.vue`（import、config watch、`applyThemeByConfig`、常量）
- Modify: `packages/mobile/src/pages/App.vue`（同上）

**Interfaces:**
- Consumes: `normalizeThemePreference`, `applyThemeMode`, `setStoredThemePreference` / `THEME_*` from core
- Produces: config 变更时 preference 原样持久化；DOM 用 resolve

- [ ] **Step 1: 修正 import 与常量**

两端 App 改为从 core 引入（删除本地 `THEME_CACHE_KEY` 硬编码也可改为 import）：

```ts
import {
  applyThemeMode,
  // DefaultVal, functions, getDefaultPost 等保持原样
  normalizeThemePreference,
  resolveThemeMode,
  setStoredThemePreference,
  THEME_CACHE_KEY,
  THEME_USER_KEY,
  subscribeSystemThemeChange,
} from "@v2next/core/core.ts";
```

（路径与现有 `@v2next/core/core.ts` 或 `@v2next/core` 保持一致。）

data 中增加（若尚无）：

```ts
_unsubSystemTheme: null as null | (() => void),
_onStorageTheme: null as null | ((e: StorageEvent) => void),
```

- [ ] **Step 2: 修正 deep `config` watch 中的主题写入**

将类似：

```ts
const mode = normalizeThemeMode(newVal?.themeMode)
if (newVal?.themeMode !== mode) {
  newVal.themeMode = mode
}
// ...
defaultConfig.themeMode = mode
localStorage.setItem(THEME_CACHE_KEY, mode)
```

改为：

```ts
const preference = normalizeThemePreference(newVal?.themeMode, 'system')
if (newVal?.themeMode !== preference) {
  newVal.themeMode = preference
}
// ...
defaultConfig.themeMode = preference
localStorage.setItem(THEME_CACHE_KEY, preference)
localStorage.setItem(THEME_USER_KEY, userKey)
```

或直接：

```ts
setStoredThemePreference(preference, userKey)
// 若 setStored 已写 v2ex-config 的 themeMode，注意与下方整份 config 写入的顺序：
// 推荐：先规范化 newVal.themeMode，再 JSON 整存 configMap[userKey]=newVal，再 set THEME_CACHE_KEY=preference
```

完整推荐片段（PC，Mobile 同构）：

```ts
handler(newVal, oldVal) {
  const preference = normalizeThemePreference(newVal?.themeMode, 'system')
  if (newVal.themeMode !== preference) {
    newVal.themeMode = preference
  }
  const configStr = localStorage.getItem('v2ex-config')
  const configObj = configStr ? JSON.parse(configStr) : {}
  const userKey = window.user.username || 'default'
  configObj[userKey] = newVal
  const defaultConfig = configObj.default ?? {}
  defaultConfig.themeMode = preference
  configObj.default = defaultConfig
  localStorage.setItem('v2ex-config', JSON.stringify(configObj))
  localStorage.setItem(THEME_CACHE_KEY, preference)
  localStorage.setItem(THEME_USER_KEY, userKey)
  window.config = newVal
  // 原有 note 同步等逻辑保持
  window.parse.editNoteItem?.(window.user.configPrefix + JSON.stringify(window.config), window.user.configNoteId)
}
```

- [ ] **Step 3: 修正 `applyThemeByConfig`**

```ts
applyThemeByConfig() {
  const preference = normalizeThemePreference(this.config?.themeMode, 'system')
  if (this.config.themeMode !== preference) {
    // 只纠正非法值；system 必须保留
    this.config.themeMode = preference
  }
  this.isNight = applyThemeMode(preference, false)
  // 不要 setItem resolved；cache 已在 config watch 写 preference
  // 若仅 themeMode watch 触发而 deep watch 未跑，可：
  localStorage.setItem(THEME_CACHE_KEY, preference)
}
```

**删除**任何 `this.config.themeMode = resolveThemeMode(...)` 或把 resolved 赋回 config 的代码。

- [ ] **Step 4: `config.themeMode` watch 保持 apply + 图标**

```ts
'config.themeMode': {
  handler() {
    this.applyThemeByConfig()
    this.updateThemeToggleIcon?.()
  },
  immediate: true
}
```

- [ ] **Step 5: Commit**

```bash
git add packages/pc/src/App.vue packages/mobile/src/pages/App.vue
git commit -m "fix: 主题 config watch/apply 不再把 system 固化为 light/dark"
```

---

### Task 5: 设置页三选一 + 当前解析文案

**Files:**
- Modify: `packages/pc/src/components/Modal/SettingModal.vue`（主题 radio 区约 115–130；watch 约 483–487）
- Modify: `packages/mobile/src/pages/Setting.vue`（约 59–76；watch 约 244–248）

**Interfaces:**
- Consumes: `config.themeMode` preference；`resolveThemeMode` 或父级传入的 resolved / `window.isNight`
- Produces: UI 可写 `system|light|dark`

- [ ] **Step 1: PC SettingModal 模板**

替换主题模式 radio 为：

```vue
<div class="row border">
  <label class="item-title">主题模式</label>
  <div class="wrapper">
    <div class="radio-group2">
      <div class="radio"
           @click="config.themeMode = 'system'"
           :class="config.themeMode === 'system' ? 'active' : ''">
        跟随系统
        <span v-if="config.themeMode === 'system'" class="theme-resolved-hint">
          （当前：{{ resolvedThemeLabel }}）
        </span>
      </div>
      <div class="radio"
           @click="config.themeMode = 'light'"
           :class="config.themeMode === 'light' ? 'active' : ''">浅色
      </div>
      <div class="radio"
           @click="config.themeMode = 'dark'"
           :class="config.themeMode === 'dark' ? 'active' : ''">深色
      </div>
    </div>
  </div>
</div>
```

- [ ] **Step 2: PC computed + watch 校验**

在 script 中 import `resolveThemeMode`（或 `normalizeThemePreference`）：

```ts
import { resolveThemeMode, normalizeThemePreference, /* 已有 */ } from "@v2next/core/core.ts"
```

computed：

```ts
resolvedThemeLabel() {
  const resolved = resolveThemeMode(this.config?.themeMode, false)
  return resolved === 'dark' ? '深色' : '浅色'
}
```

watch `config` handler 内替换：

```ts
if (!['light', 'dark'].includes(n.themeMode)) {
  n.themeMode = 'light'
}
```

为：

```ts
n.themeMode = normalizeThemePreference(n.themeMode, 'system')
```

- [ ] **Step 3: Mobile Setting.vue 同样修改**

模板与 watch 与 PC 对称；Mobile 的 `resolvedThemeLabel` 同逻辑。

可选样式（若换行难看，加 scoped）：

```less
.theme-resolved-hint {
  font-size: 12px;
  opacity: 0.75;
  margin-left: 2px;
}
```

- [ ] **Step 4: 手工验收 UI**

1. 打开设置 → 可见三项；默认新用户为「跟随系统」active。
2. 系统深色时跟随系统旁显示「当前：深色」。
3. 点「深色」→ 立即深色；点「跟随系统」→ 回到系统色且 cache 为 `system`。

- [ ] **Step 5: Commit**

```bash
git add packages/pc/src/components/Modal/SettingModal.vue packages/mobile/src/pages/Setting.vue
git commit -m "feat: 设置页支持跟随系统/浅色/深色并显示当前解析"
```

---

### Task 6: 切换按钮与原站夜间入口

**Files:**
- Modify: `packages/pc/src/App.vue`（`interceptThemeToggle`、`updateThemeToggleIcon`、clickA night 分支约 615–619）
- Modify: `packages/mobile/src/pages/App.vue`（night toggle 约 261–265）

**Interfaces:**
- Consumes: `resolveThemeMode(this.config.themeMode)`
- Produces: 点击后 preference = 对立 resolved 色

- [ ] **Step 1: 抽出统一切换方法（PC）**

```ts
toggleThemePreference() {
  const resolved = resolveThemeMode(this.config.themeMode, false)
  this.config.themeMode = resolved === 'dark' ? 'light' : 'dark'
},
```

所有原：

```ts
this.config.themeMode = this.isNight ? 'light' : 'dark'
```

改为 `this.toggleThemePreference()`（`interceptThemeToggle` 内 origin toggle、v2next 按钮、clickA night 链接）。

- [ ] **Step 2: 更新 `updateThemeToggleIcon` title**

```ts
updateThemeToggleIcon() {
  const toggle = document.querySelector('.v2next-theme-toggle')
  if (!toggle) return
  const preference = normalizeThemePreference(this.config?.themeMode, 'system')
  const resolved = resolveThemeMode(preference, false)
  const size = 20
  const icons = { /* 保持现有 light/dark svg */ }
  const resolvedLabel = resolved === 'dark' ? '深色' : '浅色'
  const nextLabel = resolved === 'dark' ? '浅色' : '深色'
  toggle.innerHTML = icons[resolved]
  if (preference === 'system') {
    toggle.title = `当前：跟随系统（${resolvedLabel}），点击切换为${nextLabel}`
  } else {
    toggle.title = `当前：${resolvedLabel}模式，点击切换为${nextLabel}`
  }
}
```

- [ ] **Step 3: Mobile night 链接**

```ts
if (href.includes('/settings/night/toggle')) {
  const resolved = resolveThemeMode(this.config.themeMode, false)
  this.config.themeMode = resolved === 'dark' ? 'light' : 'dark'
  functions.stopEvent(e)
  return
}
```

- [ ] **Step 4: 验收**

1. preference=system 且系统深色 → 点按钮 → `themeMode==='light'`，页面浅色，再改系统主题页面不变。
2. 再点 → `dark`。
3. 设置改回跟随系统 → 再跟系统。

- [ ] **Step 5: Commit**

```bash
git add packages/pc/src/App.vue packages/mobile/src/pages/App.vue
git commit -m "fix: 主题切换按钮锁定对立色并退出跟随系统"
```

---

### Task 7: 系统监听 + 跨标签 storage

**Files:**
- Modify: `packages/pc/src/App.vue`（`mounted` / `beforeUnmount` / methods）
- Modify: `packages/mobile/src/pages/App.vue`（同上）

**Interfaces:**
- Consumes: `subscribeSystemThemeChange`, `getStoredThemePreference`, `normalizeThemePreference`, `applyThemeMode`

- [ ] **Step 1: mounted 订阅**

在现有 `mounted` 末尾追加：

```ts
this._unsubSystemTheme = subscribeSystemThemeChange(() => {
  if (normalizeThemePreference(this.config?.themeMode, 'system') !== 'system') return
  this.isNight = applyThemeMode('system', false)
  this.updateThemeToggleIcon?.()
})

this._onStorageTheme = (e: StorageEvent) => {
  if (!e.key || (e.key !== THEME_CACHE_KEY && e.key !== 'v2ex-config')) return
  const userKey = window.user?.username || 'default'
  const next = getStoredThemePreference(userKey)
  if (!next) return
  const preference = normalizeThemePreference(next, 'system')
  if (this.config.themeMode === preference) {
    // 仍可能需要 re-apply（例如仅 DOM 被改）
    return
  }
  this.config.themeMode = preference
  // themeMode watch 会 apply；若 deep watch 触发 note 同步可接受
}
window.addEventListener('storage', this._onStorageTheme)
```

import 增加 `getStoredThemePreference`。

- [ ] **Step 2: beforeUnmount 清理**

```ts
if (this._unsubSystemTheme) {
  this._unsubSystemTheme()
  this._unsubSystemTheme = null
}
if (this._onStorageTheme) {
  window.removeEventListener('storage', this._onStorageTheme)
  this._onStorageTheme = null
}
```

Mobile 的 `beforeUnmount` 同样清理（现有只清 eventBus / click）。

- [ ] **Step 3: 验收**

| # | 操作 | 期望 |
|---|---|---|
| 3 | system + 系统深→浅 | 页面变浅，`themeMode` 仍 `system`，`v2next-theme-mode==='system'` |
| 4 | dark + 系统变浅 | 保持深色 |
| 9 | 两标签同域，A 改主题 | B 同步（storage 事件；需真实两标签，非同页） |

- [ ] **Step 4: Commit**

```bash
git add packages/pc/src/App.vue packages/mobile/src/pages/App.vue
git commit -m "feat: 系统主题 change 监听与跨标签 theme 同步"
```

---

### Task 8: 全量验收 + 构建产物

**Files:**
- 可能更新: `packages/pc/dist/V2Next.user.js`, `packages/mobile/dist/V2Next-Mobile.user.js`

- [ ] **Step 1: 按 spec §10 完整清单走查**

| # | 场景 | 通过标准 |
|---|---|---|
| 1 | 清主题相关缓存，OS 深色，首次打开 | `html.dark`；preference/`v2next-theme-mode` 为 `system` |
| 2 | OS 浅色首次 | 无 dark；preference `system` |
| 3 | system 下切换 OS | 页面跟随；preference 仍 system |
| 4 | 锁定 dark 后切换 OS | 不跟随 |
| 5 | system 下点切换按钮 | 写入对立 light/dark，退出跟随 |
| 6 | 设置三选一 | 跟随系统显示「当前：x」 |
| 7 | 预置 light 缓存 | 升级后仍浅色 |
| 8 | 硬刷新 | 无长时间错误主题闪烁 |
| 9 | 跨标签 | B 标签同步 |

额外 grep 回归（实现后在仓库根执行）：

```bash
rg -n "themeMode = (normalizeThemeMode|resolveThemeMode)" packages --glob '!**/dist/**'
rg -n "\?\? 'light'" packages/pc/src/main.ts packages/mobile/src/main.ts
rg -n "\['light', 'dark'\]" packages --glob '!**/dist/**'
rg -n "themeMode: 'light'" packages/core/core.ts
```

期望：无「bootstrap ?? light」；无设置页仅 light/dark 强制；`getDefaultConfig` 为 system；无 resolve 写回 themeMode。

- [ ] **Step 2: 构建**

```bash
pnpm --filter pc run build-pc
pnpm --filter vite-project run build-mobile
```

（mobile 包名以 `packages/mobile/package.json` 的 `"name"` 为准，当前为 `vite-project`；也可用 `pnpm --dir packages/mobile run build-mobile`。）

若 `vue-tsc` 失败：先修类型（`themeMode` 含 system 后 Setting 校验、global.d.ts 若写死 light|dark 需同步）。

检查 `packages/pc/src/global.d.ts` / `packages/mobile/src/global.d.ts` 是否声明 `themeMode`；若有，改为含 `system`。

- [ ] **Step 3: Commit 源码（及可选 dist）**

```bash
git add packages/core packages/pc/src packages/mobile/src
# 若构建成功且仓库习惯提交 dist：
# git add packages/pc/dist packages/mobile/dist
git status
git commit -m "$(cat <<'EOF'
feat: 完成深浅色跟随系统主题切换

默认 system、三态持久化、系统监听与设置/按钮联动。
EOF
)"
```

若 dist 体积大且本次仅源码验证通过，可只提交源码，commit message 注明「dist 未更新」。

---

## Spec coverage checklist

| Spec 要求 | Task |
|---|---|
| preference vs resolved 铁律 | 1, 4 |
| 类型含 system、默认 system | 1 |
| 存储接受 system | 1, 3 |
| bootstrap 无缓存 → system | 3 |
| apply 不写 storage / 不固化 | 1, 4 |
| subscribeSystemThemeChange | 2, 7 |
| 设置三选一 + 当前解析 | 5 |
| 按钮锁定对立色 | 6 |
| 原站 night 劫持 | 6 |
| 跨标签 storage | 7 |
| 老缓存不迁移 | 3 验收、1 存储语义 |
| 验收 §10 | 8 |
| 不引入新依赖 / 不重做配色 | 全局约束 |

## Placeholder / 一致性自检

- 无 TBD；API 名在 Task 1 定义，后续任务统一用 `normalizeThemePreference` / `resolveThemeMode` / `getStoredThemePreference` / `setStoredThemePreference` / `subscribeSystemThemeChange`
- `normalizeThemeMode` 语义已改为 preference（与旧 resolve 行为不同）——所有调用点在 Task 3–6 按新语义使用
- 默认 fallback 统一 `'system'`，与 spec 一致
