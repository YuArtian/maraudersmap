
> [[https://juejin.cn/post/7089058201814958116]]
>
# TL;DR

- **npm ci**
  可以根据 lock 文件生成准确版本的 node_modules 文件夹，主要用于 ci/cd 流程中
- **npm i**
  只是根据 package json 的版本信息，安装符合 [[语义版本控制 SemVer]]的相关依赖，但是可以更新 lock 文件

## **执行 npm i 命令主要做了两件事：**

- 首先，它会根据 `package.json` 文件，创建`node_modules` 文件夹并安装对应的依赖版本；

- 然后， 生成/更新 `package-lock.json` 文件；

## **执行 npm ci 命令也做了两件事：**

- 首先，它会删除 `node_modules` 文件夹；

- 然后，依照 `package-lock.json`（或 `npm-shrinkwrap.json`）文件 创建新得`node_modules` 文件夹并**精准安装对应的依赖版本**

>`npm-shrinkwrap.json`文件，是在 NPM v5 版本之前，通过运行`npm shrinkwrap`命令产生用来精准控制安装依赖版本的文件；

package.json 中的版本并不是一个固定的版本，而是符合 semver 版本控制

## npm 安装过程

```text
npm install
    │
    ▼
┌─────────────────────────────┐
│  1. 读取配置                │  ← .npmrc / npm config
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│  2. 构建依赖树              │  ← 解析 package.json 中的依赖
│     - 递归解析所有依赖       │
│     - 扁平化处理 (npm v3+)  │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│  3. 检查本地缓存            │  ← ~/.npm/_cacache
│     - 有缓存 → 直接使用      │
│     - 无缓存 → 从 registry 下载
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│  4. 下载 & 校验             │
│     - 下载 tarball          │
│     - 校验 integrity hash   │
│     - 写入缓存              │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│  5. 解压到 node_modules     │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│  6. 生成/更新 lock 文件     │  ← package-lock.json
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│  7. 执行生命周期脚本        │  ← preinstall / install / postinstall
└─────────────────────────────┘
```

**关键步骤说明：**

1. **读取配置**：从 `.npmrc` 文件和环境变量中读取 registry、proxy、auth 等配置

2. **构建依赖树**：
   - 从 `package.json` 开始，递归解析所有 `dependencies` 和 `devDependencies`
   - npm v3+ 采用**扁平化策略**，尽量将依赖提升到顶层 `node_modules`，减少嵌套

3. **检查缓存**：npm 会在本地维护一份缓存（默认 `~/.npm/_cacache`），避免重复下载

4. **下载 & 校验**：从 registry 下载 tarball，并通过 `integrity` 字段（SHA-512 hash）校验完整性

5. **解压安装**：将包解压到 `node_modules` 目录

6. **更新 lock 文件**：记录实际安装的精确版本和依赖结构

7. **执行脚本**：按顺序执行 `preinstall` → `install` → `postinstall` 钩子
