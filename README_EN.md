# Valtio State Management Skill

A comprehensive skill for Claude Code that provides guidelines and best practices for creating valtio-based state management hooks/stores in React applications.

## Features

- **Complete Valtio Guidelines**: Covers proxy, persist, derive, and utility APIs
- **Best Practices**: Clear naming conventions, file structure patterns, and code organization
- **Common Patterns**: Ready-to-use templates for basic, persisted, and derived stores
- **Anti-patterns**: Detailed error examples and explanations
- **Tool Functions**: Includes serialization strategy, AES encryption/decryption utilities

## Installation

### Option 1: Copy to your project

```bash
# Copy the skill directory to your project's .claude/skills/ folder
cp -r .claude/skills/valtio-state /path/to/your/project/.claude/skills/
```

### Option 2: Global installation

```bash
# Copy to your global Claude skills directory
cp -r .claude/skills/valtio-state ~/.claude/skills/
```

## Usage

Once installed, invoke the skill in Claude Code when:

- Creating new stores
- Writing React hooks for state management
- Modifying existing state management code

```
User: Create a user store with persistence
Claude: [Automatically follows valtio-state guidelines]
```

## Quick Reference

### Basic Store Pattern

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

### Persisted Store Pattern

```typescript
const { store: state } = await persist<{
  token?: string
}>({
  token: undefined,
}, 'AUTH_STORE', {
  serializationStrategy: serialization(['token']),
})
```

### Derived Values Pattern

```typescript
const derived = derive({
  count: (get) => get(state).items.length,
  isEmpty: (get) => get(state).items.length === 0,
})
```

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Hook File | `use-xxx.ts` | `use-user.ts` |
| Hook Function | `useXxx` | `useUser` |
| State Variable | `state` | - |
| Derived Values | `derived` | - |
| Snapshot | `snap` | - |
| Get State | `getState` | - |
| Get Derived | `getDerive` | - |

## Important Warnings

### effect API

Valtio's native `effect` API has known bugs:
- Circular dependencies
- Complex object monitoring failures
- Array monitoring failures

**Recommendation**: Avoid using `effect`. Use `useEffect` in an `AppEffect` component instead.

### derive Limitations

`derive` may fail to update for arrays and objects.

**Workaround**: Calculate directly in React components or use `useMemo`.

## Checklist

When creating a new Store, ensure:

- [ ] File named `use-xxx.ts`
- [ ] Hook named `useXxx` (1-2 words)
- [ ] State variable named `state`
- [ ] Derived values named `derived`
- [ ] `getState()` and `getDerive()` functions written
- [ ] `getXxx()` export function (functions only)
- [ ] `useXxx()` React Hook written
- [ ] Cross-store references use `getXxx()` pattern
- [ ] Use `persist` for persistence
- [ ] Use `proxyMap`/`proxySet` for Map/Set
- [ ] Use `ref` for shallow monitoring

## Dependencies

```json
{
  "valtio": "^1.x",
  "valtio-persist": "^0.x",
  "derive-valtio": "^0.x"
}
```

## License

MIT
