#工程化 #npm #index

# Node 包管理器索引

> Library/Node包管理器 总索引：本目录只放**版本号与命令**这一层的细节，包管理器之间的对比与 node_modules 结构演进在 [[1.03 npm、yarn 与 pnpm]]

## 文章清单

| 文章 | 一句话定位 |
|---|---|
| [[语义版本控制 SemVer]] | `主版本.次版本.修订号` 各自代表什么，以及 `^` `~` 范围符的含义 |
| [[npm i & npm ci]] | `npm ci` 按 lock 文件精确还原、适合 CI/CD；`npm i` 会按范围更新并改写 lock |

---

## 跨目录关联

- 三个包管理器的结构差异与 pnpm 为什么快 → [[1.03 npm、yarn 与 pnpm]]
- lock 文件、依赖三兄弟与 package.json 字段 → [[1.02 package.json 与依赖管理]]
- 多包仓库里的 workspace 协议与版本发布 → [[1.04 Monorepo 与工作区]]
- Monorepo 版本管理的落地经验 → [[1.02 Monorepo 落地实践]]

---

## 维护说明

本目录是「命令与版本号」层的细节笔记，不重复 Library/工程化 里的原理叙述。新增时挂进上表写一句定位；某个包管理器的笔记攒到三篇以上再考虑开子目录（现有 `Npm/` 即此例）。
