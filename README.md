# Valtio 状态管理技能

[English](./README_EN.md)

一个为 Claude Code 提供的综合性技能，用于在 React 应用中创建基于 valtio 的状态管理 hooks/stores，提供指南和最佳实践。

## 特性

- **完整的 Valtio 指南**：覆盖 proxy、persist、derive 和工具 API
- **最佳实践**：清晰的命名规范、文件结构模式和代码组织方式
- **常用模式**：可直接使用的基础 store、持久化 store 和派生 store 模板
- **反模式**：详细的错误示例和解释
- **工具函数**：包含序列化策略、AES 加密/解密工具

## 安装

### 方式一：复制到项目

```bash
# 将技能目录复制到项目的 .claude/skills/ 文件夹
cp -r .claude/skills/valtio-state /path/to/your/project/.claude/skills/
```

### 方式二：全局安装

```bash
# 复制到全局 Claude 技能目录
cp -r .claude/skills/valtio-state ~/.claude/skills/
```

## 使用方法

安装后，在以下场景中调用此技能：

- 创建新的 store
- 编写状态管理的 React hooks
- 修改现有的状态管理代码

```
用户: 创建一个带持久化的用户 store
Claude: [自动遵循 valtio-state 指南]
```

## 快速参考

### 基础 Store 模式

```typescript
const state = proxy<{ count: number }>({ count: 0 })

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

### 持久化 Store 模式

```typescript
const { store: state } = await persist<{
  token?: string
}>({
  token: undefined,
}, 'AUTH_STORE', {
  serializationStrategy: serialization(['token']),
})
```

### 派生值模式

```typescript
const derived = derive({
  count: (get) => get(state).items.length,
  isEmpty: (get) => get(state).items.length === 0,
})
```

## 命名规范

| 元素 | 规范 | 示例 |
|------|------|------|
| Hook 文件 | `use-xxx.ts` | `use-user.ts` |
| Hook 函数 | `useXxx` | `useUser` |
| 状态变量 | `state` | - |
| 派生值 | `derived` | - |
| 快照 | `snap` | - |
| 获取状态 | `getState` | - |
| 获取派生 | `getDerive` | - |

## 重要警告

### effect API

Valtio 原生的 `effect` API 存在已知问题：
- 循环依赖
- 复杂对象监听失败
- 数组监听失败

**建议**：避免使用 `effect`，改用 `AppEffect` 组件中的 `useEffect`。

### derive 限制

`derive` 在处理数组和对象时可能无法正常更新。

**解决方案**：在 React 组件中直接计算，或使用 `useMemo`。

## 检查清单

创建新 Store 时，确保：

- [ ] 文件命名为 `use-xxx.ts`
- [ ] Hook 命名为 `useXxx`（1-2 个单词）
- [ ] 状态变量命名为 `state`
- [ ] 派生值命名为 `derived`
- [ ] 编写 `getState()` 和 `getDerive()` 函数
- [ ] 编写 `getXxx()` 导出函数（仅函数）
- [ ] 编写 `useXxx()` React Hook
- [ ] 跨 store 引用使用 `getXxx()` 模式
- [ ] 需要持久化时使用 `persist`
- [ ] Map/Set 使用 `proxyMap`/`proxySet`
- [ ] 浅层监听使用 `ref`

## 依赖

```json
{
  "valtio": "^1.x",
  "valtio-persist": "^0.x",
  "derive-valtio": "^0.x"
}
```

## 许可证

MIT
