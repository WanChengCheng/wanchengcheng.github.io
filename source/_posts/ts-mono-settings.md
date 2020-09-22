---
title: monorepo基础设置指南
date: 2020-09-18 11:35:53
tags: javascript, typescript, mono-repo
---

## 简介

在 JS 社区中，很多重量级的库和框架都使用 mono-repo 这种形式来组织代码，也可以说是项目逐渐复杂以后的一个必然选择。从 2017 年开始，公司 WEB 相关项目的代码组织方式也逐渐从由单个对项目组织慢慢过渡到了 monorepo。

写这篇文章的目的，一个是总结一下这些年用 monorepo 的各种坑以及一些感受，另一个也是记录一下设置的过程。值得注意的是，JS 社区发展和变化很快，就说 monorepo 这个事情，17 年到现在随着工具功能的迭代，就已经有了不少的变化，不过以后如果有什么新的东西也会更新在这里（大概 😏 ）。

## 方案

首先，monorepo 是有许多不同的方案的，各方案有一些细微的区别，具体体现在对依赖的管理上。

很多文章讲 monorepo 都会提到一个叫 [lerna](https://github.com/lerna/lerna) 的工具。lerna 应该是比较早的多项目管理工具，可以很方便的进行多项目的构建、测试、发布等各种功能。lerna 可以搭配 npm，也可以搭配 yarn，其自身通过 symlinks 支持项目之间的代码本地复用。

我们都知道每个 package 都有自己的依赖声明，依赖安装在 package.json 同目录的 node_modules 中。monorepo 有个特点是包含多个 package，而且 monorepo 自己也是一个相对比较特殊的'package'（有自己的 package.json）。monorepo 依赖管理的核心就是如何管理各个 package 依赖，以及如何实现「本地复用」。这个「本地复用」指的是各子项目之间可以直接引用、复用代码，而不需要把包发布到 npm。

最开始 lerna 的方案比较直接，就是各个 package 都有一个自己的 node_modules，然后`lerna bootstrap`可以分析各 package 之间的本地依赖，通过 symlinks 自动链接。这个方案好处是简单直接，可以保证每个包的依赖相互独立，基本不出幺蛾子。然而有个问题：很多被重复依赖的包被安装很多次，而且这种重复依赖会很多。比如几乎所有包含业务逻辑的 package 可能都要安装 axios、momentjs 这类依赖，然后这类依赖继续往下会有更多的重复。

为了解决这个问题， lerna 后续增加了 hoisting 的功能。所谓「hoisting」就是可以把各 package 重复的「公共依赖」提升到根目录。 另外一个解决方案就是 [yarn-workspace](https://classic.yarnpkg.com/en/docs/workspaces/)。 yarn-workspace 的 hoisting 更进一步：不仅仅是把项目共同的依赖「提升」到 monorepo root，而是整个 repo 只有根目录有 node_modules。同样是利用 symlinks，yarn-workspace 的这种方案可以最大限度的减少包的重复安装问题。但是这种方式也会导致一些包工作不正常，所以有配套的 noHoist 配置，可以选择性的把一些包安装在项目本地，这里不展开，详情可以去看 yarn-workspace 的文档。

所以我们总结一下， monorepo 事实上有 4 种方案：

- lerna + yarn
- lerna + npm
- yarn-workspace
- lerna + yarn-workspace

前两种就是使用 lerna 来管理依赖，包括 hoisting 也是 lerna 提供。后两种核心是利用 yarn-workspace 来管理依赖，lerna 只作为一种可选的工具库，方便开发部署各 package 用的，本身已经不是必须。

这里都是以社区用的比较多的 lerna 来说，另外 lerna 还有一个 alternative：[bolt](https://github.com/boltpkg/bolt) 。不过我没有用过，谈不上什么经验。Bolt 从使用程度上来看和 lerna 还不是一个数量级。感兴趣的可以自己去 [openbase.io](https://openbase.io/) 上看一下。

我个人倾向的是第四种方案，也就是 lerna + yarn-workspace。使用 yarn-workspace 来管理依赖，然后使用 lerna 提供的很多便捷功能。

这篇文章对各种不同的方案做了一些性能测试:

[Why Lerna and Yarn Workspaces is a Perfect Match for Building Mono-Repos – A Close Look at Features and Performance](https://doppelmutzi.github.io/monorepo-lerna-yarn-workspaces/)。

这篇文章推荐的也是 lerna + yarn-workspace 的方案。

使用 yarn-workspace 的话，需要使用的是 yarn 1.x 的「经典」版本，目前来看 yarn 2.x 的做法太过激进，批评的声音很多，怕坑太多还是先观望吧。

## yarn-workspace + lerna monorepo 实施

值得注意的一点是，由于项目间共享的依赖都会被提升到 root 或者直接安装在 root（或者像 yarn-workspace 这样几乎所有依赖都安装在根目录），可能出现在某些 package 中不用声明也能直接使用一些依赖（这些包存在于其他某些项目的依赖中）。这些包在 monorepo 中测试运行正常，但是单独打包发布或者部署以后可能会出现某依赖找不到的问题。实际使用中需要注意一下，为了尽量避免这种问题推荐的做法是：

- 在 workspace root 只安装开发相关依赖；
- 使用 [eslint-plugin-import](https://github.com/benmosher/eslint-plugin-import) 的 lint 规则对依赖进行检查

同时也因为这个机制，我们可以把各 packages 都需要用到的依赖直接安装在 monorepo 的根目录，比如 eslint、babel、webpack 等各种可以在各项目之间「共享」的基础开发依赖。这样就不用在每个项目里都把 eslint，typescript，各种 plugin 啥的来一遍。

## 示例

文章接下来，我以我最常用和习惯的配置为蓝本，做一个实际的例子供大家参考。同时也算是一个备忘吧。

首先，做一些说明，后续的设置会和这几个假设相关：

- 使用 vscode 作为 IDE。
- 使用最新版 typescript 作为主要开发语言。
- 使用 eslint 来规范项目的代码风格。
- 使用比较流行的 airbnb eslint config 作为基础 lint 规则。
- 使用 prettier 来对代码和各种源文件做格式化，统一风格。
- 项目中会同时包含前端项目（以 react 生态为主）和 nodejs （node latest LTS - 12.x +）的项目。
- 使用上面提到过的 lerna + yarn-workspace 作为 monorepo 方案。

安装 yarn，安装 lerna（推荐全局安装，运行少打几个字，很方便），安装 vscode 和插件这些乱七八糟有手就行的就不啰嗦了。

### 创建项目

```bash
# create monorepo dir
mkdir ts-mono

# quick init with lerna
cd ts-mono
lerna init

# ignore node_modules, dist, lib director
echo "\nnode_modules/\ndist/\nlib/" >> .gitignore
```

这时项目基本是这样的：

```
ts-mono
├── lerna.json
├── package.json
└── packages
└── .gitignore
```

### 启用 yarn 作为包依赖管理工具，启用 yarn-workspace

按照 lerna 文档上的说明（[参考](https://github.com/lerna/lerna/blob/ef8380930742eb35bca1034808ceda2909852761/commands/bootstrap/README.md#--use-workspaces)）。我们需要在`lerna.json`中添加 `npmClient: "yarn"` 以及 `useWorkspaces: true`。

```bash
# 打开lerna.json，添加如下配置
{
  "packages": [
    "packages/*"
  ],
  "npmClient": "yarn",
  "useWorkspaces": true,
  "version": "0.0.0"
}
```

按照 yarn-workspace 文档上的说明（[参考](https://classic.yarnpkg.com/en/docs/workspaces/#toc-how-to-use-it)），我们需要在 monorepo 根目录下的 package.json 里增加「workspaces」的相关配置。

```bash
# package.json
{
    ...
      "workspaces": {
          "packages": [ "packages/*" ],
          "nohoist": []
       },
    ...
}
```

这种写法已经考虑了 nohoist 的场景，所以和最简单的情形不太一样（`workspaces: ["packages/*"]`），这里的 nohoist 上面提到过，目前可以不用管，不过相信以后你会用到:P。

### 安装 VSCode 插件，配置 editorconfig, prettier

基于前述假设，推荐安装 vscode 的几个插件：

- dbaeumer.vscode-eslint
  - 可以在 vscode 中显示 lint 错误
- esbenp.prettier-vscode
  - 可以通过 `format document`命令或者保存的时候自动对代码进行格式化
- editorconfig.editorconfig
  - 覆盖用户的编辑器设定，对源代码文件和其他各种类型的文件做一个 indent、max line 等的基础设置

**配置 editorconfig**

于 monorepo 根目录创建配置文件 `.editorconfig`， editorconfig 插件会读取该文件。

```bash
# file: .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
insert_final_newline = true

[*.{js,jsx,ts,tsx,es6,sh,rb}]
indent_size = 2
max_line_length = 80
trim_trailing_whitespace = true

[*.{py}]
indent_size = 4
max_line_length = 80
trim_trailing_whitespace = true

[*.{css,scss,less,html,json,yml,yaml}]
indent_size = 2
trim_trailing_whitespace = true

```

**配置 prettier**

在 monorepo 根目录下创建 `prettier.config.js`文件，prettier 插件会读取该配置文件。

```js
module.exports = {
  printWidth: 100,
  singleQuote: true,
  trailingComma: "all",
};
```

注意 prettier 插件目前是不需要你在项目中安装 prettier 作为开发依赖的，应该是自带了。

### 配置 typescript

接下来应该是配置 eslint，然而配置 eslint 之前需要先配置一下 typescript，因为我们主开发语言是 typescript，所以需要替换 eslint 的 parser，ts 的 parser 依赖 typescript，所以我们需要把 monorepo 的 ts 先配置好。

typescript 配置 tsconfig.json 可以直接用 tsc 生成：

```bash
# in ts-mono/
yarn add -D typescript

tsc --init
```

如果你没有全局安装 tsc，需要换成`npx tsc`。

我们的思路是在 monorepo 的根目录下丢一个 tsconfig.json，然后 packages/下面的其他 package 会有自己的 tsconfig.json,然后 extends monorepo 根目录下的这个配置。由于各 package 都会有自己的 tsconfig，所以根目录下的 tsconfig 只是定个调，不需要特别纠结，这里给出一个例子：

```json
{
  "compilerOptions": {
    "skipLibCheck": true,
    // node 12 => ES2019
    "target": "ES2019",
    "module": "esnext",
    "lib": ["ESNext"],
    "strict": true,
    "baseUrl": "./",
    "esModuleInterop": true,
    "paths": {},
    "forceConsistentCasingInFileNames": true /* Disallow inconsistently-cased references to the same file. */
  }
}
```

tsconfig 主要还是需要参考 typescript 官方的[配置说明文档](https://www.typescriptlang.org/tsconfig)，这里面`skipLibCheck`, `strict`, `esModuleInterop`, `forceConsistentCasingInFileNames` 是比较关键的配置型。

### **配置 eslint**

editorconfig 和 prettier 的配置简单直接，然而 eslint 就没那么省心了，值得单独拉出来多说几句。eslint 在使用的时候碰到过不少问题，总结出几点需要注意的：

1. vscode-eslint 和 prettier 插件不一样：prettier 插件装完后可以作为 IDE 功能直接使用，eslint 插件需要自己安装 eslint 依赖，eslint 插件说自己会尝试寻找使用你安装的 eslint。
2. **不推荐**全局安装 eslint， vscode-eslint 插件在文档中声明会优先使用你本地的 eslint。这样有个好处是出了问题比较容易定位。推荐把 eslint 作为开发依赖直接安装在 monorepo 的根目录，因为所有项目都需要。
3. 如果使用 typescript，需要`@typescript-eslint/parser`来把 ts 代码解析成 ast，同时建议安装`@typescript-eslint/eslint-plugin`，包含了一些 ts 语言限定的规则。
4. prettier 和 eslint 的规则可能会**互相冲突**。 prettier 推荐的做法是使用他们提供的 eslint 配置，这个配置会关掉所有跟 format 相关的规则，也就是由 prettier 来复杂代码格式统一的问题（formatter），eslint 则主要检查代码的写法（linter）。现在比较新的语言如 go、rust 基本都自带 formatter 了，不像 JS 发挥空间这么大……
5. 由于语言相同，monorepo 中的各 package 的 eslint 配置多多少少都是有相似的。但是具体到不同的项目，又会有很多差别。比如客户端项目需要 react 相关的插件和规则，后端项目则完全不需要。解决这种 eslint 配置「共享」的问题，有两种思路：
   - 第一种就是「[Configuration Cascading and Hierarchy](https://eslint.org/docs/user-guide/configuring#configuration-cascading-and-hierarchy)」，也就是 Cascading：父目录放一个 eslintrc,子目录放一个 eslintrc，在 lint 子目录的时候 eslint 会「合并」两个配置。
     这个方式实际使用的时候并没有那么灵活，而且 vscode-eslint 插件对其支持的不是很好，对于比较复杂的结构还需要配置`workingDirectories`，实际使用发现本来正常工作的 eslint 会突然莫名其妙的在某个项目突然失效……
   - 第二种则是按照 Eslint 推荐的「[Shareable Configs](https://eslint.org/docs/developer-guide/shareable-configs)」，解释一下就是：发布一个 eslint-config，里面包含各类规则，然后在其他项目里直接 extends 即可。
     由于我们使用 monorepo，所以这个配置项目可以是一个单独的 package，而且也没有必要发布到 npm，直接本地依赖即可，而且我们可以针对 react 项目、nodejs 项目定制不同的规则，各 pacakge 在使用时还可以再修改定制。我个人比较推荐这种方式，虽然看起来可能会麻烦一些，但是后续扩展起来不容易出问题。

上面说过推荐使用`airbnb`的规则作为基础。airbnb 的 eslint 配置并没有 typescript 版本，所以用的是社区的：[eslint-config-airbnb-typescript](https://www.npmjs.com/package/eslint-config-airbnb-typescript)

然后 eslint-config-airbnb 本身依赖：

- `eslint-plugin-import`, 上面提到过这个插件
- `eslint-plugin-jsx-a11y`，跟 acc 相关的，国内可能不太在意，不过属于 peer dependencies，还是装了吧。
- `eslint-plugin-react`
- `eslint-plugin-react-hooks`

以及我们上述提过的，为了支持 typescript 必须安装的

- `@typescript-eslint/parser`
- `@typescript-eslint/eslint-plugin`

以及我们上述提过的，为了解决 prettier 和 eslint rules 之间可能冲突问题的

- `eslint-config-prettier`

**安装 eslint 相关依赖**

上面也说过推荐在根目录安装 eslint 相关依赖，所以总结一下，在 root 根目录下执行:

```bash
# in ts-mono/
yarn add -WD eslint eslint-config-airbnb-typescript \
    eslint-config-airbnb-typescript \
    eslint-config-prettier \
    eslint-plugin-import \
    eslint-plugin-jsx-a11y \
    eslint-plugin-react \
    eslint-plugin-react-hooks \
    @typescript-eslint/eslint-plugin \
    @typescript-eslint/parser

touch .eslintignore
```

**创建一个 eslint-config package**
一般我们在 monorepo 里新建一个 package，可以用`lerna create`来快速生成，不过 lerna 默认生成发布到 npm 的 package，有很多需要改的，所以也可以简单点：

```bash
# in /ts-mono
cd packages
# in /ts-mono/packages
mkdir eslint-config && cd eslint-config
# in /ts-mono/packages/eslint-config
yarn init -y
#
touch base.js react.js node.js
```

这里稍微调整一下，把 main 入口改成 base.js，另外把包名改成`@flip/eslint-config`。当然这里的 scope 自己替换成别的。如果不加 scope，按照 eslint 的命名规范，package 应该叫`eslint-config-yourcomp`之类的更符合规范。

这些配置都比较简单，不用上 ts，直接用 js 就行，整个 package 包含三个文件：`base.js`写最基础的 eslint 配置，然后写一个前端项目专用的配置`react.js`，一个后端项目用的配置`node.js`。

```bash
ts-mono
├── .editorconfig
├── .eslintignore
├── .gitignore
├── lerna.json
├── package.json
├── packages
│   └── eslint-config
│       ├── base.js
│       ├── node.js
│       ├── package.json
│       └── react.js
├── prettier.config.js
└── yarn.lock

2 directories, 11 files
```

**编写基础配置 base.js**

这个 eslint 的写法和自己实际项目需求有关， eslint 配置还请大家阅读 eslint 自身的文档，这里面列几个要点，并给出一个参考。

- 由于主语言是 typescript，所以 parser 需要替换为`@typescript-eslint/parser`
- parserOptions.ecmaVersion 按照需要进行设置，比如`2020`
- extends 需要依次继承:`airbnb-typescript`、`prettier`来解决前面说过的冲突。
- plugins 需要包含前面提过的`@typescript-eslint/eslint-plugin`, `eslint-plugin-import`。

这里给一个基于 airbnb-typescript 规则的参考：

```js
module.exports = {
  parser: "@typescript-eslint/parser",
  parserOptions: {
    ecmaVersion: 2020,
  },
  extends: [
    // recommended by 'eslint-plugin-import
    "plugin:import/recommended",
    "plugin:import/typescript",
    // preferred eslint config base
    "airbnb-typescript",
    // prevent format rules conflict
    "prettier",
  ],
  settings: {},
  // leave the plugins to the package level settings
  plugins: ["import", "@typescript-eslint"],
  rules: {
    // prefer named export!
    "import/prefer-default-export": "off",

    // just as bad as "max components per file"
    // "max-classes-per-file": "off",

    // Missing yarn workspace support
    "import/no-extraneous-dependencies": "off",
  },
};
```

这里我们先注释掉`max-classes-per-file`这条规则，方便接下来的测试。

**测试 base.js 配置**

我们新建一个 nodejs 的 package，故意写一个违反规则的代码文件使用上面定义的配置来检查代码。

```bash
# in ts-mono/packages
mkdir proj-a && cd proj-a
# in ts-mono/packages/proj-a
yarn init -y && mkdir src && touch src/index.ts && touch tsconfig.json && touch .eslintrc.js
```

我们需要手动的修改一下 package.json，加入一个本地开发依赖：我们刚刚创建的`@flip/eslint-config`。

```json
// package.json for 'proj-a'
{
  "name": "proj-a",
  "version": "1.0.0",
  "main": "index.js",
  "devDependencies": {
    "@flip/eslint-config": "*"
  },
  "license": "MIT"
}
```

接下来 monorepo 的结构大致是这样的：

```bash
ts-mono
├── .editorconfig
├── .eslintignore
├── .gitignore
├── lerna.json
├── package.json
├── packages
│   ├── eslint-config
│   │   ├── base.js
│   │   ├── node.js
│   │   ├── package.json
│   │   ├── react.js
│   │   └── yarn.lock
│   └── proj-a
│       ├── .eslintrc.js
│       ├── package.json
│       ├── src
│       │   └── index.ts
│       └── tsconfig.json
├── prettier.config.js
├── tsconfig.json
└── yarn.lock

4 directories, 17 files
```

proj-a 的 eslint 配置如下（继承@flip/eslint-config）：

```js
// .eslintrc.js
module.exports = {
  extends: ["@flip/eslint-config"],
  parserOptions: {
    project: "./tsconfig.json",
  },
};
```

注意一下由于我们使用@typescript-eslint/parser, 这里的`parserOptions`是必须设置的。

proj-a 的 tsconfig 配置如下（继承 monorepo 根目录下的 tsconfig）：

```json
// tsconfig.json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {}
}
```

我们在 proj-a 中故意写一个违背上面提到的`max-classes-per-file`规则的代码……

```js
export class A {}

export class B {}
```

如果配置正常，vscode 应该是能够正常反应出 lint 错误：
![lint-error](./lint-error.png)

如果没有正常显示，检查一下 Eslint 的输出（可以使用 vscode 命令：Eslint: Show Output Channel）看看有什么问题。

接下来可以去`eslint-config`项目，把`base.js`里`max-classes-per-file`规则的注释重新打开，proj-a 项目中的 lint 报错就消除（修改了 eslint 项目后可能需要重启一下 vscode 客户端，其实理论上是需要重启一下 eslint 的插件，但是还没找到重启的方法所以直接重启 vscode 😅）。

**react 专用的 lint base**

react 专用的 eslint 配置直接继承我们之前测试过的 base，然后加上 react 相关的插件和配置，示例：

```js
// eslint-config/react.js
module.exports = {
  extends: ["@flip/eslint-config/base", "plugin:react-hooks/recommended"],
  plugins: ["react", "react-hooks"],
  rules: {
    "react/jsx-filename-extension": ["error", { extensions: [".js", ".tsx"] }],
    "react/jsx-props-no-spreading": "off",
  },
};
```

**nodejs 环境的 lint base**

nodejs 项目用的 eslint 配置同样是继承 base，最基础的只需要一行：

```js
// eslint-config/node.js
module.exports = {
  extends: ["@flip/eslint-config/base"],
};
```

具体使用上面的例子已经 cover 了，总结一下就是：

- 在项目中声明依赖`"@flip/eslint-config": "*"`
- 创建项目的`.eslintrc.js`，然后 extends`@flip/eslint-config`或者具体某个配置：`@flip/eslint-config/node`, `@flip/eslint-config/react`
- 添加`tsconfig.json`继承 monorepo 根目录的配置，做出自己的修改
- 配置`.eslintrc.js parserOptions`

到这里一个基础版的 monorepo 就设置完了。接下来有空可能会写一些 walkthroughs 包括在 monorepo 里添加 react 组件项目，cra 项目，nextjs 项目以及纯后端项目。
