
> [[https://juejin.cn/post/7089058201814958116]]
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
