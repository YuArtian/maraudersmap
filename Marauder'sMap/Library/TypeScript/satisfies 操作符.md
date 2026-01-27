#typescript

> TypeScript 4.9 引入的 `satisfies` 操作符：在保留类型推断的同时进行类型检查

## TLDR

`satisfies` 让你**既能检查类型是否符合约束，又能保留字面量类型推断**，而不是像 `as` 那样强制断言或像类型注解那样丢失精确类型。

## 问题场景

### 使用类型注解的问题

```typescript
type Color = 'red' | 'green' | 'blue'
type ColorMap = Record<string, Color>

// 使用类型注解
const colors: ColorMap = {
  primary: 'red',
  secondary: 'blue',
}

// ❌ 问题：丢失了 key 的精确类型
colors.primary    // 类型是 Color，而不是 'red'
colors.unknown    // ✅ 不报错，但其实不存在这个 key
```

### 不使用类型注解的问题

```typescript
// 不使用类型注解
const colors = {
  primary: 'red',
  secondary: 'blue',
}

// ✅ 保留了精确类型
colors.primary    // 类型是 'red'
colors.unknown    // ❌ 报错

// ❌ 但是没有类型检查，可能写错值
const colors2 = {
  primary: 'red',
  secondary: 'yellow',  // 不会报错，但 'yellow' 不是 Color
}
```

## satisfies 解决方案

```typescript
type Color = 'red' | 'green' | 'blue'
type ColorMap = Record<string, Color>

// 使用 satisfies
const colors = {
  primary: 'red',
  secondary: 'blue',
} satisfies ColorMap

// ✅ 保留了精确类型
colors.primary    // 类型是 'red'
colors.unknown    // ❌ 报错，不存在这个 key

// ✅ 同时有类型检查
const colors2 = {
  primary: 'red',
  secondary: 'yellow',  // ❌ 报错：'yellow' 不能赋值给 Color
} satisfies ColorMap
```

## 对比总结

| 方式 | 类型检查 | 保留精确类型 |
|------|---------|------------|
| 无类型注解 | ❌ | ✅ |
| 类型注解 (`:`) | ✅ | ❌ |
| 类型断言 (`as`) | ❌ (强制通过) | ❌ |
| **`satisfies`** | ✅ | ✅ |

## 实际应用

### 配合 `as const` 使用

`satisfies` 常与 `as const` 配合使用，实现**类型安全的常量定义**：

```typescript
// 定义 tag 必须以 'tag_' 开头
type ResourceTag = `tag_${string}`

// as const: 保留字面量类型
// satisfies: 确保符合约束
export const BUILT_IN_TAGS = ['tag_comic', 'tag_favorite'] as const satisfies ResourceTag[]

// 自动推导出联合类型
type BuiltInTag = (typeof BUILT_IN_TAGS)[number]  // 'tag_comic' | 'tag_favorite'

// ❌ 如果写错会立即报错
const WRONG_TAGS = ['comic'] as const satisfies ResourceTag[]
// Error: Type '"comic"' does not satisfy '`tag_${string}`'
```

### 配置对象验证

```typescript
interface Config {
  env: 'development' | 'production'
  port: number
  debug?: boolean
}

const config = {
  env: 'development',
  port: 3000,
  debug: true,
} satisfies Config

// config.env 的类型是 'development'，而不是 'development' | 'production'
```

### 路由配置

```typescript
interface Route {
  path: string
  component: string
}

const routes = {
  home: { path: '/', component: 'Home' },
  about: { path: '/about', component: 'About' },
} satisfies Record<string, Route>

// routes.home.path 类型是 '/'，而不是 string
```

## 语法说明

```typescript
expression satisfies Type
```

- `expression`：要检查的表达式
- `Type`：约束类型
- **返回**：原表达式的推断类型（不是 `Type`）

关键区别：

- `const x: Type = value` → `x` 的类型是 `Type`
- `const x = value satisfies Type` → `x` 的类型是 `value` 的推断类型

## 最佳实践

1. **常量数组**：`as const satisfies Type[]` 组合使用
2. **配置对象**：需要精确 key 类型时使用
3. **类型安全的映射**：`Record<K, V>` 配合 `satisfies`
4. **保留字面量**：需要窄类型推断时使用

## 参考

- [TypeScript 4.9 Release Notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html)
- [satisfies Operator - TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator)
