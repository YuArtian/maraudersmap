#iOS #Xcode #依赖管理 #SPM #CocoaPods

# iOS 依赖管理：SPM 与 CocoaPods

## TL;DR

**iOS 有两套互相竞争的依赖管理器：SPM 是 Apple 内建的，CocoaPods 是第三方外挂的。它们解决同一个问题，但集成方式完全不同。**

- CocoaPods 需要手动跑 `pod install`，SPM 完全没有「安装依赖」这一步，Xcode 打开项目就自动做了
- CocoaPods 会生成**第二个 Xcode 项目**（`Pods.xcodeproj`），再用 `.xcworkspace` 把两个项目捆在一起，所以入口必须换成 workspace
- SPM 把依赖直接写进 `project.pbxproj`，不需要额外项目，入口还是原来的 `.xcodeproj`
- 根本差异是**格式所有权**：Apple 能直接扩展自己的项目文件格式，第三方工具只能在外面套一层
- Flutter 项目里总要跑 `pod install`，是 Flutter 工具链替你选的，不是 iOS 开发的必然

---

## 1. 它们是竞品，不是上下游

最容易产生的误解是「iOS 依赖都得 `pod install`，某些项目比较特殊」。

实际上 CocoaPods 和 Swift Package Manager（SPM）是**两个互相竞争的工具**，做的是同一件事。一个项目通常只用其中一套（也可以混用，见第 6 节）。

在纯 SPM 的项目里跑 `pod install`，不是「多此一举」，而是**直接报错**——没有 `Podfile`，CocoaPods 根本不知道该装什么。

| | CocoaPods | SPM |
| --- | --- | --- |
| 出身 | 第三方，Ruby 写的 | Apple 官方 |
| 诞生时间 | 2011 年 | 2015 年（2019 年集成进 Xcode） |
| 安装方式 | 单独装 gem | 随 Xcode 自带 |
| 集成方式 | 外挂 | 内建 |

---

## 2. 操作上的对应关系

如果你熟悉 CocoaPods（或者 npm），这张表能快速建立映射：

| 概念 | CocoaPods | SPM | npm 类比 |
| --- | --- | --- | --- |
| 声明依赖 | `Podfile` | `project.pbxproj` 内部 | `package.json` |
| 锁定版本 | `Podfile.lock` | `Package.resolved` | `package-lock.json` |
| 安装依赖 | `pod install` | **无需手动**，Xcode 自动 | `npm install` |
| 更新依赖 | `pod update` | File → Packages → Update to Latest | `npm update` |
| 依赖存放位置 | `Pods/`（项目内，需 gitignore） | `DerivedData/SourcePackages/`（项目外） | `node_modules/` |
| 打开项目 | 必须用 `.xcworkspace` | 直接用 `.xcodeproj` | — |

注意「声明依赖」那一行：SPM **没有独立的清单文件**。依赖声明直接住在 Xcode 项目文件里面，这是理解后面所有差异的钥匙。

版本约束的写法两者都遵循 [[语义版本控制 SemVer]]。SPM 里最常见的 `upToNextMajorVersion`（升到下个大版本之前）等价于 npm 的 `^`。

---

## 3. CocoaPods 为什么必须用 .xcworkspace

先分清两个概念：

- **`.xcodeproj` 是一个项目**：记录有哪些源文件、哪些 target、各种 build settings
- **`.xcworkspace` 是一个容器**：本身几乎不含内容，作用是把若干个 `.xcodeproj` 装在一起，让 Xcode 知道「这几个项目要一起构建、可以互相引用产物」

关系类似文件夹和文件——workspace 不是项目，是装项目的盒子。

### pod install 到底做了什么

关键的一点，可能和直觉不同：**`pod install` 会生成第二个完整的 Xcode 项目**。

它在 `Pods/` 下建出 `Pods.xcodeproj`，每个 pod 是里面的一个 target，编译成静态库或 framework。你的依赖不是被塞进你自己的项目，而是活在一个平行的、CocoaPods 自己管理的项目里。

于是问题来了：你的 `App.xcodeproj` 要用 `Pods.xcodeproj` 编出来的东西，但两个独立的 `.xcodeproj` 之间无法可靠地表达依赖关系——Xcode 不知道该先编哪个、产物放哪、头文件搜索路径怎么串。

**解决办法是给它们找一个共同的父容器**：

```text
App.xcworkspace          ← pod install 生成的容器（你要打开的）
├── App.xcodeproj        ← 你原本的项目
└── Pods/
    └── Pods.xcodeproj   ← pod install 生成的依赖项目
```

这就是「从外部包住」的字面含义：CocoaPods 没有把依赖写进你项目的**内部**，而是在你的项目**外面**套了一层更大的容器，让两个项目变成兄弟节点。你原本打开的 `.xcodeproj` 从此降级成容器里的一员。

### 那个经典报错

这也解释了为什么 CocoaPods 项目里直接打开 `.xcodeproj` 会报一堆 `No such module`：

你打开的是孤立的那半边，`Pods.xcodeproj` 根本没被加载，依赖自然全部找不到。

---

## 4. SPM 为什么不需要这一层

因为 **Apple 拥有 `.xcodeproj` 这个格式**，可以直接扩展它。SPM 的依赖声明就写在 `project.pbxproj` 里，作为原生的一等公民：

```text
5E17A0030000000000000003 /* XCRemoteSwiftPackageReference "sentry-cocoa" */ = {
    isa = XCRemoteSwiftPackageReference;
    repositoryURL = "https://github.com/getsentry/sentry-cocoa";
    requirement = {
        kind = upToNextMajorVersion;
        minimumVersion = 9.19.1;
    };
};
```

`XCRemoteSwiftPackageReference` 是 Xcode 原生认识的类型。本地包则用 `XCLocalSwiftPackageReference`：

```text
ABC0000000000000000000001 /* XCLocalSwiftPackageReference "../Packages/MyFeatureKit" */ = {
    isa = XCLocalSwiftPackageReference;
    relativePath = ../Packages/MyFeatureKit;
};
```

没有第二个项目，没有容器，Xcode 读到这段就知道该去哪拉代码。

CocoaPods 作为第三方工具做不到这件事——它没法往 Apple 的格式里发明新类型让 Xcode 认账，只能用「在外面套一层」这种迂回办法。

### 一句话对比

| | 做法 | 入口 |
| --- | --- | --- |
| CocoaPods（外挂） | 另起一个项目装依赖，用 workspace 把两边捆住 | `.xcworkspace` |
| SPM（内建） | 依赖直接声明在项目文件里，Xcode 原生解析 | `.xcodeproj` |

---

## 5. 为什么 Flutter 项目里总在跑 pod install

这是 Flutter 架构决定的，跟个人习惯无关。

Flutter 插件（`camera`、`shared_preferences` 之类）的 iOS 原生部分是以 **podspec** 形式分发的。Flutter 工具链会自动生成 `ios/Podfile`，把 Flutter engine 和各插件的 iOS 端接进去，`flutter build ios` 内部就会调 `pod install`。

换句话说：**那不是你在做 iOS 依赖管理，是 Flutter 替你选好了 CocoaPods。**

Flutter 官方近年一直在推进对 SPM 的支持，长期看这块会慢慢迁移，但存量项目现在基本还是 CocoaPods。

---

## 6. 怎么判断一个项目用哪套

接手任何 iOS 项目，先看目录：

| 看到什么 | 用哪套 | 该怎么做 |
| --- | --- | --- |
| `Podfile` + `.xcworkspace` | CocoaPods | 跑 `pod install`，**必须**打开 `.xcworkspace` |
| 只有 `.xcodeproj`，没有 Podfile | SPM | 什么都不用敲，打开就行 |
| 两者都有 | 混用 | `pod install` 照跑，SPM 部分 Xcode 自己管 |

第三种在老项目迁移中途很常见——不是错误状态，两套可以共存。

### 现代项目为什么倾向 SPM

- 主流库（Firebase、GRDB、Sentry、Amplitude 等）都已提供官方 SPM 支持
- 少一个 Ruby 工具链依赖，新人上手不用先折腾 gem 环境
- 支持**本地包**：可以把 app 代码拆成 `Packages/XxxKit/`，用 `Package.swift` 显式声明模块边界和依赖方向。CocoaPods 做本地模块化要笨重得多

---

## 相关文章

- [[Xcode SPM 依赖解析失败排查]] - SPM 依赖没下全时的诊断方法
- [[语义版本控制 SemVer]] - `upToNextMajorVersion` 等版本约束的语义
- [[lock 文件]] - `Package.resolved` / `Podfile.lock` 的同类概念
- [[npm i & npm ci]] - 「按需补齐」与「清空重装」的对照

---

## 参考资料

- Apple Developer: Swift Packages in Xcode
- CocoaPods 官方文档 <https://cocoapods.org>
- Flutter 官方文档：iOS 平台集成与 Swift Package Manager 迁移进展
- 实践来源：公司 iOS 项目（纯 SPM，29 个包）的依赖排查
