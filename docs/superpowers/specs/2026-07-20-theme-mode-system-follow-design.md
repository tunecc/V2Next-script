# 深浅色主题：跟随系统与切换体验修复

- 日期：2026-07-20
- 状态：已批准（brainstorming）
- 范围：V2Next userscript（PC + Mobile），核心在 `packages/core`，两端 UI 同步

## 1. 背景与问题

当前主题实现（`abc37d0` 引入）体验差：

1. **首次打开不跟随系统**：无缓存时 bootstrap 写死 `light`（`getStoredThemeMode() ?? 'light'`）。
2. **系统主题变化不跟随**：无 `matchMedia('(prefers-color-scheme: dark)')` 的 `change` 监听。
3. **`system` 无法真正持久化**：
   - `Config.themeMode` 类型只有 `'light' | 'dark'`
   - `normalizeThemeMode` / 设置页校验会把非 light/dark 值落回 light
   - resolve 后的结果被写回 `config.themeMode` 与 localStorage，等于把「跟随系统」固化成某一色

结果：浏览器/OS 为深色时，脚本既不会首次跟随，也不会在系统切换后更新。

## 2. 目标与非目标

### 2.1 目标

- 无有效主题缓存时，**默认跟随系统**（`system`）。
- 用户可在设置中选择：`跟随系统` / `浅色` / `深色`；跟随系统时展示当前解析结果。
- 偏好为 `system` 时，OS 主题变化 → 页面自动切换。
- 偏好为 `light`/`dark` 时，忽略 OS 变化。
- 顶栏/原站切换按钮：点击即**锁定对立色并退出跟随系统**。
- 已有 `light`/`dark` 缓存的老用户**不被强制迁移**。
- 启动尽早 apply，降低错误主题闪烁。
- 主题逻辑收敛到 `packages/core`，PC/Mobile 删除重复存储实现。

### 2.2 非目标

- 不重做全站配色（沿用现有 `html.dark` 样式体系）。
- 不引入 VueUse / next-themes 等新依赖。
- 不修改 V2EX 原站 cookie 夜间逻辑（继续覆盖/隐藏原站入口）。
- 不做「临时覆盖后自动回 system」等复杂交互。

## 3. 已确认产品决策

| 项 | 决策 |
|---|---|
| 默认策略 | A：无有效缓存 → `system` |
| 设置 UI | B：三选一 + 跟随系统时显示「当前：深色/浅色」 |
| 快捷切换按钮 | A：点击锁定对立色，退出 system |
| 老用户缓存 | A：尊重已有 `light`/`dark`，不强制改 system |
| 实现路径 | 方案 1：三态持久化 + 系统监听（对齐 next-themes / VueUse 模型） |
| 跨标签同步 | 做：监听 `storage` 同步 preference |

## 4. 数据模型

### 4.1 Preference vs Resolved

| 概念 | 含义 | 持久化 | 取值 |
|---|---|---|---|
| **preference** | 用户选择 | 是（config + localStorage） | `'light' \| 'dark' \| 'system'` |
| **resolved** | 实际渲染主题 | 否（DOM / 内存） | `'light' \| 'dark'` |

**铁律：持久化只写 preference；禁止把 resolve 结果写回 `themeMode`。**

### 4.2 类型变更

```ts
// packages/core/types.ts
themeMode: 'light' | 'dark' | 'system'  // 原为 'light' | 'dark'
```

core 建议导出：

```ts
type ThemePreference = 'light' | 'dark' | 'system'
type ThemeResolved = 'light' | 'dark'
```

### 4.3 默认值

- `getDefaultConfig().themeMode = 'system'`
- bootstrap：`getStoredThemePreference() ?? 'system'`
- 非法 preference → `normalizeThemePreference` 回退为 `'system'`（不再回退为 `'light'`）

### 4.4 存储

| Key | 内容 |
|---|---|
| `v2ex-config[userKey].themeMode` | preference |
| `v2ex-config.default.themeMode` | 同步 preference（保持现有双写习惯） |
| `v2next-theme-mode` | preference 快读缓存，**允许 `system`** |
| `v2next-theme-user-key` | 当前用户 key（保持） |

兼容：

- 旧缓存 `light`/`dark` → 原样作为 preference
- 旧逻辑若曾写入非法值 → 归一为 `system`
- 历史上 `LegacyThemeMode` 含 `system` 的路径保留并成为一等公民

## 5. 核心 API（packages/core）

职责收敛到 core；PC/Mobile `main.ts` 删除本地重复的 get/set 主题存储。

| API | 职责 |
|---|---|
| `getSystemThemeMode()` | `matchMedia('(prefers-color-scheme: dark)')` → resolved（已有） |
| `normalizeThemePreference(v)` | → `'light'\|'dark'\|'system'`；非法 → `'system'` |
| `resolveThemeMode(preference)` | preference → resolved；`system` 时读系统 |
| `getStoredThemePreference(userKey?)` | 读缓存；无有效值 → `undefined` |
| `setStoredThemePreference(pref, userKey?)` | 只写 preference |
| `applyThemeMode(preference)` | resolve → DOM（`html.dark` / color-scheme / 清原站 Night）+ `window.isNight`；**不改写 preference / storage** |
| `subscribeSystemThemeChange(cb)` | 注册 mql `change`；返回 unsubscribe |
| （可选封装）`bootstrapTheme()` | 读缓存 ?? system → apply；不把 system 固化 |

命名落地时可在现有 `normalizeThemeMode` / `getStoredThemeMode` 上演进，但语义必须符合上表；若保留旧名，实现与调用点必须按 preference 语义修正，避免 silent 回归。

### 5.1 apply 行为（保持并修正）

- `document.documentElement.classList.toggle('dark', resolved === 'dark')`
- `dataset.v2nextTheme` / `style.colorScheme` 使用 **resolved**
- 移除原站 `Night` class（`resetOriginThemeMode` 现有职责）
- 更新 `meta[name=theme-color]`
- `window.isNight = (resolved === 'dark')`

### 5.2 系统监听

```ts
const mql = window.matchMedia('(prefers-color-scheme: dark)')
// 优先 addEventListener('change')；必要时兼容 addListener
mql.addEventListener('change', () => {
  if (currentPreference !== 'system') return
  applyThemeMode('system')
  // UI：刷新按钮图标、「当前：x」文案、isNight
})
```

- 订阅挂在 App `mounted`，`beforeUnmount` 取消
- 仅 preference === `'system'` 时响应
- 不写 storage

### 5.3 跨标签

监听 `window` `storage`：

- key 为 `v2next-theme-mode` 或 `v2ex-config` 变化时
- 重新读取 preference → 若与当前不同则更新 config 并 apply
- 避免回环（本页自己写入触发的 storage 事件在同源同页通常不触发，但仍以「值变化才 apply」为准）

## 6. 启动与运行时流程

### 6.1 Bootstrap（修首次不跟随）

```
脚本注入
  → preference = getStoredThemePreference() ?? 'system'
  → applyThemeMode(preference)          // 只改 DOM / isNight
  → body 未就绪则 DOMContentLoaded 再 apply 一次
  → 禁止：setStored(resolved) / 把 system 写成 light|dark
```

### 6.2 配置变更

```
config.themeMode 变更（设置页 / 按钮）
  → preference = normalizeThemePreference(...)
  → 持久化 preference（v2ex-config + THEME_CACHE_KEY）
  → applyThemeMode(preference)
  → 更新切换按钮（按 resolved）
```

### 6.3 快捷切换按钮

```
resolved = resolveThemeMode(currentPreference)
preference = resolved === 'dark' ? 'light' : 'dark'
// 从 system 点击 = 退出跟随并锁定对立色
```

图标与 title 一律按 **resolved** 显示。

建议 title：

- system：`当前：跟随系统（深色/浅色），点击切换为浅色/深色`
- light/dark：`当前：浅色/深色模式，点击切换为深色/浅色`

### 6.4 与原站夜间入口

- 继续劫持/隐藏 `.light-toggle`、拦截 `/settings/night/toggle` 链接
- 上述入口统一走「按钮点击逻辑」（写 light/dark，非 toggle system）

## 7. UI

### 7.1 设置页（PC SettingModal + Mobile Setting）

Radio 三选一：

1. **跟随系统** — 写入 `system`；**选中时**旁显 `当前：深色` / `当前：浅色`（随系统/resolved 更新）
2. **浅色** — `light`
3. **深色** — `dark`

- active 绑定 `config.themeMode === preference`（不是 resolved）
- 删除「非法则强制 light」的校验，改为 `normalizeThemePreference`

### 7.2 PC 顶栏 `.v2next-theme-toggle`

- 逻辑与 title 按第 6.3 节
- Mobile 无独立顶栏按钮时，依赖原站 night 链接拦截即可

## 8. 关键调用点修正清单

必须修掉的错误模式：

1. `normalizeThemeMode` 后写回 `config.themeMode` 导致 system 消失  
2. `getStored` 的 normalize 只接受 light/dark，丢掉 system  
3. `getDefaultConfig().themeMode = 'light'`  
4. `bootstrapThemeMode = getStored() ?? 'light'`  
5. 设置页 `if (!['light','dark'].includes) themeMode = 'light'`  
6. `applyThemeByConfig` 内 `this.config.themeMode = mode`（mode 为 resolved）  
7. watch config 时 `localStorage.setItem(THEME_CACHE_KEY, resolved)`

## 9. 建议改动文件

| 文件 | 改动 |
|---|---|
| `packages/core/types.ts` | `themeMode` 含 `system` |
| `packages/core/core.ts` | preference/resolve/存储/监听/默认值/API 语义 |
| `packages/pc/src/main.ts` | bootstrap 用 core；删重复 get/set |
| `packages/mobile/src/main.ts` | 同上 |
| `packages/pc/src/App.vue` | watch / apply / 按钮 / 系统监听 / storage |
| `packages/mobile/src/pages/App.vue` | 同上（含 night 链接拦截） |
| `packages/pc/src/components/Modal/SettingModal.vue` | 三选一 + 当前解析文案 |
| `packages/mobile/src/pages/Setting.vue` | 同上 |
| `packages/*/dist/*` | 按仓库习惯是否同步提交构建产物 |

## 10. 验收标准

| # | 场景 | 期望 |
|---|---|---|
| 1 | 清缓存 / 新用户，OS 深色 | 首次打开深色；preference 为 `system`（不被固化为 dark） |
| 2 | 同上，OS 浅色 | 首次浅色；preference 为 `system` |
| 3 | preference=`system`，系统深→浅 | 页面自动变浅；preference 仍为 `system` |
| 4 | preference=`dark`，系统变浅 | 页面保持深色 |
| 5 | preference=`system` 时点切换按钮 | 变为对立色并写入 light/dark；之后不跟系统 |
| 6 | 设置选「跟随系统」 | 立即按系统 apply；旁显「当前：x」 |
| 7 | 老用户已有 `themeMode=light` | 升级后仍浅色 |
| 8 | 刷新 | bootstrap 尽早 apply，无明显错误主题闪烁 |
| 9 | 另一标签修改主题 | 本标签同步 preference 并 apply |

## 11. 参考（外部实践）

调研结论（实现前已检索）：

- **next-themes**：默认 `defaultTheme='system'`；storage 存主题名含 system；`matchMedia` change 仅在 theme===system 时 re-apply；启动脚本在 paint 前写 DOM 防 FOUC。
- **VueUse `useColorMode`**：`store` 为 `auto|light|dark`，`system` 为 OS 偏好；`auto` 时跟随并监听。
- **Tailwind dark mode 文档**：`localStorage` 显式值 vs 无 key 时跟 `prefers-color-scheme`；早期脚本防 FOUC。
- **MDN**：`matchMedia` + `addEventListener('change')`；配合 `color-scheme`。

本项目采用同一模型的 userscript 内聚实现，不引入依赖。

## 12. 风险与回滚

| 风险 | 缓解 |
|---|---|
| 老用户突然变色 | 不迁移已有 light/dark 缓存 |
| system 再次被固化 | 验收 #1/#3；code review 盯 storage 写入点 |
| 监听泄漏 | unmount 时 unsubscribe |
| 原站 Night 与 html.dark 冲突 | 保持现有 reset/拦截策略 |
| dist 与源码不一致 | 按仓库习惯构建后提交或文档标明 |

回滚：还原本 spec 涉及文件至上一 commit；localStorage 中 `system` 值在旧代码下会被 normalize 为 light，可接受为降级。

## 13. 实现顺序建议（供 writing-plans）

1. core 类型 + preference API + 默认值 + 存储接受 system  
2. core `subscribeSystemThemeChange`  
3. PC/Mobile bootstrap 改用 core 默认 system  
4. 两端 App watch / apply 去固化  
5. 设置页 UI  
6. 切换按钮 / 原站拦截 title 与逻辑  
7. 系统监听 + storage 跨标签  
8. 手工按第 10 节验收（必要时再构建 dist）
