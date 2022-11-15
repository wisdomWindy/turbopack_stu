# 什么是turbopack

turbopack 是一个增量捆绑器和构建系统，针对JavaScript和Typescript进行了优化，由Rust编写。

不使用原生ESModule，而是把源代码捆绑到开发服务器，提高开发服务器的启动速度

## 应用场景

turbopack适用于大型应用

## 优势

- 比Vite快10倍，比webpack快700倍：
  - 高度优化的机器代码
  - 低级增量计算引擎，允许缓存到单个函数的级别

- 增量计算：最大化速度，最小化完成的工作
- 优化开发服务器的启动时间：构建一个惰性资产图仅计算请求的资源


Vite的劣势：
- 扩展问题：大量的级联请求影响启动速度

## 劣势

## 概念

### monorepo

monorepo是单个代码库中许多不同应用程序和包的集合。

#### 工作空间

每个应用程序都会在自己的工作空间中工作，工作空间通常由包管理器管理。


将要配置为工作区的文件夹添加到根package.json文件中的工作区字段。该字段包含全局形式的工作区文件夹列表

```json
// package.json
{
  "workspaces": [
    "docs",
    "apps/*",
    "packages/*"
  ]
}
```
以上配置的工作目录如下

project-directory
  - docs
  - apps
    - xxx
  - packages
    - xxx
- package.json

##### 命名工作空间

我们可以为一个工作空间命名，通常在package.json中的name字段配置
```json
// package.json
{
  "name":""
}
```

命名工作空间可以用于：
- 为被安装的包指定工作空间
- 通过工作空间的name在其他工作空间中引用
- 把指定name的工作空间发布到npm

##### 工作空间的依赖

要在另一个工作区中使用一个工作区，需要使用它的名称将其指定为依赖项

```json
{
  "dependencies": {
    "shared-utils": "*"
  }
}
```

### 增量计算

### 懒打包

### 缓存任务

turbopack将执行的每一条脚本称之为任务，它会对每一个任务的结果进行缓存

当进行build任务时，先到缓存中读取被构建的文件的缓存，如果找不到就进行构建，构建完毕把结果写入缓存

### 缓存入口

当指定入口文件发生变化时重新执行任务

```json
// turbo.json
{
  "pipeline":{
    "build":{
      "input":["src/*.js"]
    }
  }
}
```

### 缓存输出

覆盖默认的缓存输出行为

```json
{
  "pipline":{
    "build":{
      "output":["dist/"]
    }
  }
}
```

### 禁用缓存

```json
{
  "pipeLine":{
    "dev":{
      "cache":false
    }
  }
}
```

### 基于环境变量的缓存变更

```json
{
  "pipeline":{
    "test":{
      "env":["VUE_APP"]
    }
  }
}
```

### 远程缓存

turbopack的任务缓存是缓存在本地，因此在每台新的电脑上都需要重新执行。当使用CI服务器时，仍然会重复执行任务，因此需要将缓存存储到远程以便共享。

远程缓存是turbopack使用vercel平台实现，因此需要注册vercel账号

#### 使用远程缓存
1. 先登录账号

```shell
npx turbo login
```
2. 创建链接，链接到远程缓存

```shell
npx turbo link
```

在上传缓存之前turbo还提供了签名功能，当下载缓存时先进性可靠性和完整性校验，如果通过则下载否则不下载。开启此功能需要一下配置

```json
{
  "$schema": "https://turbo.build/schema.json",
  "remoteCache": {
    "signature": true
  }
}
```

### 环境变量

turbo允许使用.env文件定义环境变量，在使用环境变量之前需要安装dotenv-cli

```shell
npm install dotenv-cli
```
在目录中创建.env文件在文件中声明变量

在package.json文件中配置命令

```json
{
  "scripts":"dotenv -- turbo run dev"
}
```

### 日志

turbopack引擎会缓存控制台的输出日志，当再次执行相同任务时，会从缓存读取输出

### hashing

1. 构建一个代码库当前全局状态的hash
2. 添加给定工作区任务的更过因素
3. 一旦turbo在执行中遇到给定工作区的任务，它会检查缓存（本地和远程）是否有匹配的哈希。如果匹配，它会跳过执行该任务，将缓存的输出移动或下载到位，并立即重放之前记录的日志。如果缓存中没有任何东西（本地或远程）与计算的哈希匹配，turbo将在本地执行任务，然后使用哈希作为索引缓存指定的输出
4. 给定任务的哈希值在执行时作为环境变量TURBO_HASH注入。此值可用于标记输出或标记Dockerfile等

**在turbo v0.6.10中，turbo在使用npm或pnpm时的散列算法与上述略有不同。当使用这些包管理器中的任何一个时，turbo将在其每个工作区任务的散列算法中包含锁文件的散列内容。它不会像当前的纱线实现那样解析/计算出所有依赖项的解析集**

## 配置文件

turbopack以turbo.json和package.json文件作为打包配置文件，默认根目录是package.json文件所在的目录

pipeline：pipeline配置声明了哪些任务相互依赖

# 原理

要想使性能更快需要
 - 做更少的工作
 - 并行工作

turbopack在这两方面进行优化，turbopack创建一个共享的构建引擎，构建引擎允许函数并行调用，构建引擎缓存函数调用后的结果。

Turbopack的开发模式根据收到的请求构建应用程序导入和导出的最小图，并且仅捆绑必要的最小代码。
这种策略使Turbopack在首次启动开发服务器时速度极快。我们只计算渲染页面所需的代码，然后将其以单个块的形式发送到浏览器。在大规模上，这最终比原生ESM快得多


# turborepo

turborepo是turbopack提供的用于运行turbopack任务的工具。Turboreso可以通过了解我们任务之间的依赖关系来安排我们的任务以获得最大速度
通常情况下我们使用yarn或者npm运行打包任务，但是这些工具只能一个一个任务的运行，turborepo可以同时运行多个任务

我们可以在turbo.json中配置以下字段

```json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"]
    },
    "test": {
      "outputs": []
    },
    "lint": {
      "outputs": []
    }
  }
}
```

^build意味着构建必须在该工作空间运行之前在依赖项中运行

我们可以这样运行 turbo run lint test build 当我们运行它时，Turboreso将在所有可用的CPU上执行尽可能多的任务。

lint和test都立即运行，因为它们在turbo.json中没有指定依赖项