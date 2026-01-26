#CSS #响应式布局 #兼容性

> <https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_containment/Container_queries>
> <https://developer.mozilla.org/en-US/docs/Web/CSS/container-type>

# Container Queries（容器查询）

CSS Container Queries 允许根据**容器元素**的尺寸来应用样式，而不是基于视口（viewport）尺寸。这是对传统 Media Queries 的重要补充。

## TLDR

- **`container-type`**: 定义一个元素为查询容器
- **`container-name`**: 为容器命名（可选）
- **`@container`**: 根据容器尺寸应用样式
- **兼容性**: Safari 16+, Chrome 105+, Firefox 110+
- <mark style="background: #FF5582A6;">Safari 16 存在已知 bug，`container-type` 可能导致 flex 子元素滚动失效</mark>

## 基本概念

### Media Queries vs Container Queries

| 特性 | Media Queries | Container Queries |
|------|---------------|-------------------|
| 查询依据 | 视口（viewport）尺寸 | 容器元素尺寸 |
| 响应范围 | 整个页面 | 单个组件 |
| 侧边栏影响 | ❌ 无法感知 | ✅ 自动响应 |
| 组件复用 | 需要知道使用场景 | 完全独立 |

### 核心价值

Container Queries 使组件**真正可复用**。无论组件被放置在哪里（全宽布局、侧边栏、卡片内），它都能根据可用空间自适应。

## 语法详解

### 1. `container-type` - 定义查询容器

```css
.container {
  container-type: inline-size;
}
```

**可选值**:

| 值 | 含义 |
|---|---|
| `normal` | 默认值，不作为查询容器 |
| `size` | 基于容器的宽度和高度查询 |
| `inline-size` | 仅基于容器的宽度（inline 方向）查询，**最常用** |

<mark style="background: #FFB86CA6;">注意: `container-type` 会创建新的布局上下文（containment context），可能影响子元素的布局行为</mark>

### 2. `container-name` - 容器命名（可选）

```css
.sidebar {
  container-type: inline-size;
  container-name: sidebar;
}

.main-content {
  container-type: inline-size;
  container-name: main;
}
```

命名容器后，可以在 `@container` 查询中指定具体容器。

### 3. `container` 简写属性

```css
.container {
  /* container: <name> / <type> */
  container: sidebar / inline-size;
}
```

### 4. `@container` - 容器查询规则

```css
/* 查询最近的容器 */
@container (max-width: 600px) {
  .card {
    flex-direction: column;
  }
}

/* 查询指定名称的容器 */
@container sidebar (max-width: 300px) {
  .nav-item {
    font-size: 12px;
  }
}

/* 使用容器查询单位 */
@container (min-width: 400px) {
  .title {
    font-size: 5cqw; /* 容器宽度的 5% */
  }
}
```

## 容器查询单位

| 单位 | 含义 |
|------|------|
| `cqw` | 容器宽度的 1% |
| `cqh` | 容器高度的 1% |
| `cqi` | 容器 inline 尺寸的 1% |
| `cqb` | 容器 block 尺寸的 1% |
| `cqmin` | `cqi` 和 `cqb` 中较小的值 |
| `cqmax` | `cqi` 和 `cqb` 中较大的值 |

## 完整示例

### HTML 结构

```html
<div class="content-list">
  <div class="card-container">
    <div class="card">Card 1</div>
    <div class="card">Card 2</div>
    <div class="card">Card 3</div>
  </div>
</div>
```

### CSS 样式

```css
/* 定义容器 */
.content-list {
  container-type: inline-size;
  container-name: content-list;
}

/* 默认水平排列 */
.card-container {
  display: flex;
  gap: 12px;
}

.card {
  flex: 1;
}

/* 容器宽度 ≤ 768px 时，改为垂直排列 */
@container content-list (max-width: 768px) {
  .card-container {
    flex-direction: column;
  }
  
  .card {
    flex: none;
    width: 100%;
  }
}
```

## 兼容性与注意事项

### 浏览器支持

| 浏览器 | 最低版本 |
|--------|----------|
| Chrome | 105 |
| Firefox | 110 |
| Safari | 16 |
| Edge | 105 |

### 已知问题

#### Safari 16 Flexbox 滚动 Bug

<mark style="background: #FF5582A6;">**重要**: Safari 16 中 `container-type: inline-size` 可能导致 flex 子元素无法滚动</mark>

**问题表现**:

- 父容器设置 `container-type: inline-size`
- 子元素设置 `overflow-y: auto` 无法触发滚动
- 内容溢出但滚动条不生效

**解决方案**:

1. **改用 Media Queries**（推荐）

```css
/* 将 @container 改为 @media */
@media (max-width: 768px) {
  .card-container {
    flex-direction: column;
  }
}
```

1. **条件性启用**

```css
/* 使用 @supports 检测，但无法区分有 bug 的 Safari 16 */
@supports (container-type: inline-size) {
  .container {
    container-type: inline-size;
  }
}
```

1. **确保 flex 链上的 `min-height: 0`**

```css
.flex-child {
  min-height: 0; /* 允许 flex 子项收缩 */
}
```

### 其他注意事项

1. **性能影响**: `container-type` 会创建新的包含块（containing block），浏览器需要额外计算
2. **嵌套限制**: 容器查询不能查询自身，只能查询祖先容器
3. **层叠上下文**: `container-type: size` 会创建新的层叠上下文

## 最佳实践

### 1. 优先使用 `inline-size`

```css
/* 推荐 - 大多数情况只需要宽度查询 */
container-type: inline-size;

/* 仅在需要高度查询时使用 */
container-type: size;
```

### 2. 为复杂布局命名容器

```css
.page-layout {
  container: page / inline-size;
}

.sidebar {
  container: sidebar / inline-size;
}

/* 明确指定查询哪个容器 */
@container sidebar (max-width: 250px) { ... }
@container page (max-width: 1200px) { ... }
```

### 3. 与 Media Queries 结合使用

```css
/* 视口级别的响应式 */
@media (max-width: 768px) {
  .sidebar {
    display: none;
  }
}

/* 组件级别的响应式 */
@container (max-width: 400px) {
  .card {
    padding: 8px;
  }
}
```

### 4. 渐进增强

```css
/* 基础样式 */
.card-container {
  display: flex;
  flex-wrap: wrap;
}

/* 支持 Container Queries 的浏览器增强体验 */
@supports (container-type: inline-size) {
  .content-list {
    container-type: inline-size;
  }
  
  @container (max-width: 600px) {
    .card-container {
      flex-direction: column;
    }
  }
}
```

## 相关概念

- [[CSS/Media Queries]] - 基于视口的响应式查询
- [[CSS/Flexbox]] - 弹性盒子布局
- [[CSS/CSS Containment]] - CSS 包含规范

## 总结

1. **Container Queries** 使组件可以根据自身容器尺寸响应，而非视口
2. 使用 **`container-type: inline-size`** 定义容器
3. 使用 **`@container`** 规则编写响应式样式
4. <mark style="background: #FF5582A6;">注意 Safari 16 的兼容性问题，可能需要回退到 Media Queries</mark>
5. 结合 Media Queries 实现**视口级 + 组件级**的双重响应式
