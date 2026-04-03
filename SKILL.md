---
name: "valtio-state"
description: "Guidelines for creating valtio-based state management hooks/stores. Invoke when creating new stores, hooks, or modifying existing state management code."
---

# Valtio 状态管理规范

本规范定义了基于 Valtio 的状态管理最佳实践，适用于 React 应用的状态管理。

## 参考文档

- [reference/utils.md](./reference/utils.md)：Valtio 辅助工具函数，包含序列化策略、AES 加解密、对象操作等工具函数的实现

## 核心概念

### 基础 API

```typescript
import { proxy, ref, useSnapshot } from '@/utils'
import { derive } from 'derive-valtio'
import { persist } from 'valtio-persist'
import { proxyMap, proxySet } from 'valtio/utils'
```

### 状态定义

#### 1. 基础状态（proxy）

```typescript
const state = proxy<{
  /** 属性注释 */
  property: Type
}>({
  property: defaultValue,
})
```

**规范：**
- `proxy` 必须传递泛型
- 类型定义直接写在泛型中，非必要不拆分类型文件
- 复杂类型无法推断时，才单独定义类型
- 状态变量统一命名为 `state`

#### 2. 持久化状态（persist）

```typescript
const { store: state } = await persist<Type>(
  {
    property: defaultValue,
  },
  'STORAGE_KEY',
  {
    serializationStrategy: serialization(['key1', 'key2']),
  },
)
```

**规范：**
- 使用 `serialization` 函数进行序列化配置
- 一般不需要加密，不传递第二个参数
- `proxyMap` 和 `proxySet` 无法持久化

#### 3. 衍生值（derive）

```typescript
const derived = derive({
  /** 衍生值注释 */
  computedValue: (get) => {
    return get(state).property
  },
})
```

**规范：**
- 衍生值变量统一命名为 `derived`
- 必须使用 `derive` 函数，严禁使用 object get 或其他方式
- 必须编写 `getDerive()` 函数导出

### 数据类型处理

| 类型 | API | 说明 |
|------|-----|------|
| 基础类型 | 直接赋值 | string, number, boolean 等 |
| 对象 | 直接赋值 | 自动深度代理 |
| 浅层监听对象 | `ref(obj)` | 只监听引用变化 |
| Map | `proxyMap()` | 无法持久化 |
| Set | `proxySet()` | 无法持久化 |

## ⚠️ 重要注意事项

### effect API 慎用

Valtio 原生的 `effect` 监听 API **bug 较多**，存在以下问题：
- 循环依赖
- 复杂对象监听失效
- 数组监听失效

**非必要不建议使用！**

如果必须使用，限制条件：
- 只监听 **1-2 个属性**
- 只能是 **基本数据类型**（string, number, boolean）

```typescript
// 仅限这种简单场景
effect(
  () => startWatcher(state.root),
  cleanupWatcher,
)
```

**替代方案**：将逻辑放到 `useEffect` 中，在 `AppEffect` 组件里处理。

### derive 的局限性

`derive` 对 **数组、对象** 的监听可能失效，导致衍生值不更新。

**遇到这种情况，在 React 组件中直接计算：**

```typescript
// ❌ 可能失效
const derived = derive({
  count: (get) => get(state).items.length,
})

// ✅ 推荐在组件中直接计算
export const Component: FC = () => {
  const { snap: { items } } = useStore()
  const count = items.length
  // 或使用 useMemo
  const filteredItems = useMemo(() => items.filter(...), [items])
}
```

### useEffect 与异步函数

`useEffect` **无法接受异步函数**。

**替代方案**：使用 `useRequest` + `refreshDeps`

```typescript
// ❌ 错误
useEffect(async () => {
  await fetchData()
}, [dep])

// ✅ 正确
useRequest(fetchData, { refreshDeps: [dep] })
```

## Store 编写规范

### 文件结构

```typescript
// 1. 类型导入
import type { SomeType } from 'types'

// 2. 外部依赖（其他 store）
import { getOtherStore } from './other-store'
const { getState: otherState } = getOtherStore()

// 3. 状态定义
const state = proxy<Type>({ ... })
const derived = derive({ ... })

// 4. 工具函数
function helper() { ... }

// 5. 状态操作函数
function doSomething() {
  // 操作 state
}

// 6. 导出函数
export function getStoreName() {
  return {
    getState,
    getDerive,
    doSomething,
    // 严禁放置 state、derived
  }
}

// 7. React Hook
export function useStoreName() {
  const store = getStoreName()
  const snap = useSnapshot(state)
  const derives = useSnapshot(derived)
  return {
    state,
    derived,
    snap,
    derives,
    ...store,
  }
}
```

### 命名规范

| 元素 | 规范 | 示例 |
|------|------|------|
| Hook 文件 | `use-xxx.ts` | `use-user.ts` |
| Hook 函数 | `useXxx` | `useUser` |
| Store 名称 | 1-2 个单词 | `user`, `appTheme` |
| 状态变量 | `state` | - |
| 衍生值 | `derived` | - |
| 快照 | `snap` | - |
| 衍生快照 | `derives` | - |
| 获取状态 | `getState` | - |
| 获取衍生 | `getDerive` | - |
| 导出函数 | `getXxx` | `getUser`, `getApp` |
| useEffect 内函数 | `onXxx` | `onSyncTheme` |

### 循环引用处理

**严禁直接引入其他 store 的 state 或 derived**

正确方式：
```typescript
import { getOtherStore } from './other-store'
const { getState: otherState } = getOtherStore()

// 使用时
function doSomething() {
  const value = otherState().property
}
```

**如遇循环引用，不要动态导入，需询问用户处理方式**

## React 组件使用规范

### 基础用法

```typescript
export const Component: FC = () => {
  const { state: user, snap: { userInfo } } = useUser()
  
  return <div>{userInfo.name}</div>
}
```

**规范：**
- `state` 必须重命名：`const { state: user } = useUser()`
- `snap` 必须解构后使用：`const { snap: { userInfo } } = useUser()`
- 重命名后避免与 store 属性重名

### Snapshot 类型传递

`useSnapshot` 解构出的对象是**不可变的**，无法直接修改。

**类型定义**：需要传递 snap 对象（尤其是嵌套对象）时，使用 `Snapshot<T>` 包裹类型：

```typescript
import type { Snapshot } from 'valtio'

interface TreeNode {
  path: string
  name: string
  children?: TreeNode[]
}

// Props 类型定义
interface FileNodeProps {
  /** 文件树节点（不可变快照） */
  node: Snapshot<TreeNode>
}

// 组件使用
export const FileNode: FC<FileNodeProps> = ({ node }) => {
  return <div>{node.name}</div>
}

// 父组件传递
const { snap: { treeRoot } } = useResource()
return <FileNode node={treeRoot} />
```

**重要：**
- `Snapshot<T>` 表示不可变的快照类型
- 嵌套对象（对象套对象）必须包裹 `Snapshot`
- **无法修改 snap**，如需修改必须操作 `state`

```typescript
// ❌ 错误：无法修改 snap
const { snap: { userInfo } } = useUser()
userInfo.name = 'new'  // 报错！

// ✅ 正确：通过 state 修改
const { state: user } = useUser()
user.userInfo.name = 'new'
```

**注意**：不建议直接在组件中修改 state，建议在 store 中编写函数，除非逻辑简单。

```typescript
// ❌ 不推荐：直接在组件中修改
const { state: user } = useUser()
<Button onClick={() => user.userInfo.name = 'new'}>修改</Button>

// ✅ 推荐：在 store 中编写函数
function updateUserName(name: string) {
  state.userInfo.name = name
}

// 组件中调用函数
const { updateUserName } = useUser()
<Button onClick={() => updateUserName('new')}>修改</Button>
```

### 响应式更新

| 场景 | 使用方式 | 原因 |
|------|----------|------|
| UI 渲染 | `snap.xxx` | 响应式更新 |
| 事件处理（最新值） | `state.xxx` | 避免闭包 |
| useEffect 依赖 | `snap.xxx` | 触发更新 |
| useEffect 内部 | `state.xxx` | 保证最新值 |

### useEffect 规范

```typescript
const { snap: { theme } } = useTheme()

const onSyncTheme = useEffectEvent(() => {
  const { theme } = appTheme  // 使用 state 而非 snap
  document.body.setAttribute('data-theme', theme)
})

useEffect(() => onSyncTheme(), [theme])
```

**规范：**
- useEffect 内函数使用 `useEffectEvent` 包裹
- 函数命名：`onXxx`
- 内部使用 `state.xxx` 保证最新值
- `useEffectEvent` 在 hox 中无效，使用 `useMemoizedFn`

### 简单状态更新

```typescript
const { state: user } = useUser()

// 简单 boolean 更新可直接修改
<Button onClick={() => user.dialog = true}>打开</Button>

// 复杂逻辑建议创建函数
function openDialog() {
  state.dialog = true
  state.loading = true
}
```

## 常见模式

### 模式 1：基础 Store

```typescript
const state = proxy<{
  count: number
}>({
  count: 0,
})

function increment() {
  state.count++
}

export function getCounter() {
  return {
    getState: () => state,
    increment,
  }
}

export function useCounter() {
  const store = getCounter()
  const snap = useSnapshot(state)
  return { state, snap, ...store }
}
```

### 模式 2：持久化 Store

```typescript
const { store: state } = await persist<{
  token?: string
  user?: UserInfo
}>({
  token: undefined,
  user: undefined,
}, 'AUTH_STORE', {
  serializationStrategy: serialization(['token']),
})
```

### 模式 3：带衍生值的 Store

```typescript
const state = proxy<{ items: Item[] }>({ items: [] })

const derived = derive({
  count: (get) => get(state).items.length,
  isEmpty: (get) => get(state).items.length === 0,
})

export function getItems() {
  return {
    getState: () => state,
    getDerive: () => derived,
  }
}
```

### 模式 4：跨 Store 依赖

```typescript
import { getUser } from './use-user'

const { getState: userState } = getUser()

const state = proxy<{ data: Data[] }>({ data: [] })

function fetchData() {
  const userId = userState().userInfo?.id
  // 使用 userId 获取数据
}
```

## 错误示例

### ❌ 错误：直接引入 state

```typescript
// 错误！会导致循环引用
import { state } from './other-store'
```

### ❌ 错误：未解构 snap

```typescript
// 错误！snap 不能直接使用
const { snap } = useUser()
return <div>{snap.userInfo.name}</div>
```

### ❌ 错误：未重命名 state

```typescript
// 错误！state 未重命名
const { state } = useUser()
return <div>{state.user.name}</div>  // 变量冲突
```

### ❌ 错误：使用 object get 做衍生

```typescript
// 错误！会失去 proxy，无法响应式更新
const derived = {
  count: Object.keys(state.items).length,
}

// ✅ 正确：使用 derive 函数
const derived = derive({
  count: (get) => get(state).items.length,
})
```

**注意**：`derive` 对数组、对象的监听可能失效，建议在 React 组件中直接计算或使用 `useMemo`。

### ❌ 错误：getXXX 导出 state

```typescript
// 错误！getXXX 只能导出函数
export function getStore() {
  return {
    state,  // ❌
    doSomething,
  }
}
```

## 检查清单

创建新 Store 时，确保：

- [ ] 文件命名为 `use-xxx.ts`
- [ ] Hook 命名为 `useXxx`（1-2 个单词）
- [ ] 状态变量命名为 `state`
- [ ] 衍生值变量命名为 `derived`
- [ ] 编写了 `getState()` 和 `getDerive()` 函数
- [ ] 编写了 `getXxx()` 导出函数（只包含函数）
- [ ] 编写了 `useXxx()` React Hook
- [ ] 跨 Store 引用使用 `getXxx()` 模式
- [ ] 需要持久化时使用 `persist`
- [ ] 复杂类型才拆分类型定义
- [ ] Map/Set 使用 `proxyMap`/`proxySet`
- [ ] 浅层监听对象使用 `ref`
