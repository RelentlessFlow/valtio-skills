# Valtio 工具函数

本文档包含 Valtio 状态管理所需的工具函数和序列化策略。

## 基础 API 导出

```typescript
// utils/valtio.ts
export { derive, underive } from 'derive-valtio'
export { proxy, ref, snapshot, useSnapshot } from 'valtio'
export { persist } from 'valtio-persist'
export { proxyMap, proxySet } from 'valtio/utils'
```

## 序列化策略

用于 `valtio-persist` 的持久化序列化配置：

```typescript
// utils/valtio.ts
import type { Snapshot } from 'valtio'
import type { SerializationStrategy } from 'valtio-persist'
import { aesDecrypt, aesEncrypt, objectPick } from 'utils'
import { AES_KEY } from '@/constants'

/**
 * 序列化策略
 */
export function serialization<T extends object>(
  keys: (keyof Snapshot<T>)[] = [],
  options: { encrypt?: boolean } = {},
): SerializationStrategy<T, false> {
  const { encrypt } = options

  return {
    isAsync: false,

    serialize: (state) => {
      const picked = keys.length
        ? objectPick(state, ...keys)
        : state

      let result = JSON.stringify(picked)
      if (encrypt)
        result = aesEncrypt(result, AES_KEY)

      return result
    },

    deserialize: (data) => {
      let result = data

      if (encrypt) {
        try {
          result = aesDecrypt(data, AES_KEY)
        }
        catch (e) {
          console.warn('[Serialization] AES Decrypt failed:', e)
          return null
        }
      }

      try {
        return JSON.parse(result || '""')
      }
      catch (e) {
        console.warn('[Serialization] JSON Parse failed:', e)
        return null
      }
    },
  }
}
```

## 依赖函数

### AES 加密

```typescript
// utils/common/aes/encrypt.ts
import { random, rc2, util } from 'node-forge'

/**
 * AES 对称加密
 * @param text 待加密的文本
 * @param key 加密密钥
 * @returns 加密后的密文
 */
export function aesEncrypt(text: string, key: string) {
  try {
    const iv = random.getBytesSync(8)
    const cipher = rc2.createEncryptionCipher(key)
    cipher.start(iv)
    cipher.update(util.createBuffer(text, 'utf8'))
    cipher.finish()
    return `${util.bytesToHex(iv)}${cipher.output.toHex()}`
  }
  catch {
    return text
  }
}
```

### AES 解密

```typescript
// utils/common/aes/decrypt.ts
import { rc2, util } from 'node-forge'

/**
 * AES 对称解密
 * @param hash 待解密的密文
 * @param key 解密密钥
 * @returns 解密后的文本
 */
export function aesDecrypt(hash: string, key: string) {
  try {
    const text = util.hexToBytes(hash)
    const iv = text.slice(0, 8)
    const cipher = rc2.createDecryptionCipher(key)
    cipher.start(iv)
    cipher.update(util.createBuffer(text.slice(8)))
    cipher.finish()
    return cipher.output.toString()
  }
  catch {
    return hash
  }
}
```

### 对象属性提取

```typescript
// utils/common/object/pick.ts

/**
 * 从对象中取出指定的属性组成新的对象
 * @param obj 原始对象
 * @param keys 指定属性
 * @returns 新对象
 */
export function objectPick<T, K extends keyof T>(obj: T, ...keys: K[]): Pick<T, K> {
  return keys.reduce((acc, key) => {
    acc[key] = obj[key]
    return acc
  }, {} as Pick<T, K>)
}
```

## 使用示例

### 基础持久化

```typescript
const { store: state } = await persist<{
  token?: string
}>({
  token: undefined,
}, 'AUTH_STORE', {
  serializationStrategy: serialization(['token']),
})
```

### 加密持久化

```typescript
const { store: state } = await persist<{
  password?: string
}>({
  password: undefined,
}, 'SECRET_STORE', {
  serializationStrategy: serialization(['password'], { encrypt: true }),
})
```

### 完整 Store 示例

```typescript
import { persist } from 'valtio-persist'
import { proxy, useSnapshot } from '@/utils'
import { serialization } from '@/utils/valtio'

const { store: state } = await persist<{
  /** 用户令牌 */
  authToken: string
  /** 用户信息 */
  userInfo?: UserInfo
}>({
  authToken: '',
  userInfo: undefined,
}, 'USER_STORE', {
  serializationStrategy: serialization([
    'authToken',
    'authTokenExpire',
  ]),
})

export function getUser() {
  return {
    getState: () => state,
  }
}

export function useUser() {
  const store = getUser()
  const snap = useSnapshot(state)
  return { state, snap, ...store }
}
```
