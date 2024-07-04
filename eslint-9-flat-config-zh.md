ESLint9之前，同时包含语法检查和一部分代码风格检查，这使得ESLint和编辑器、Prettier等工具之间总是会出现一些奇怪的冲突问题，因此出现了 eslint-plugin-prettier这样的诡异插件。

+ ESLint9终于在[2024/4/6(npm)](https://www.npmjs.com/package/eslint?activeTab=versions)正式发布，随之也稳定了**flat config**特性。
+ 同时，正式开始逐步废弃代码风格检查，专注语法检查，而风格检查逐渐由社区插件接替，如：[Stylistic](https://eslint.style/guide/getting-started)  
[\[ESLint9 Release Notes更多\]](https://eslint.org/blog/2024/04/eslint-v9.0.0-released/)

基于ESLint9的flat config， 配置文件犹如options api到composition api的转变，更加清晰、简洁，同时也更加灵活。

本文将从Parser、Extend预配置文件、添加和自定义插件三大部分展示ESLint9之前和现在Flat Config的差异，同时，实现：

+ Vue3(SFC template + JSX) + TypeScript
+ 添加ESLint官方、框架官方配置文件
+ 由ESLint负责语法检查，Stylistic处理冲突并负责风格检查
+ 添加第三方插件`simple-import-sort`，展示ESLint9如何更加紧凑的支持插件（旧）

如果你是ESLint老手，直接查看[Gist](https://gist.github.com/aolyang/8ad9c14209b069806eac45b5927d00de)内容以节省时间，欢迎留言改进配置和纠正文章错误。

## 在ESLint9之前的options风格

### ESLint extends解释
+ 使用预配置文件，ESLint插件名称简写会自动去node_module中去查找：
    + 如果是`eslint:xxx`，查找eslint内置规则
    + 如果是`plugin:xxx/xxx`，最后一个`/`之前的为插件名
        + 比如：`plugin:vue/vue3-essential`，会查找`eslint-plugin-vue`，从导出的module里拿key为`vue3-essential`的配置
        + 比如：`plugin:@typescript-eslint/recommended`，会查找`@typescript-eslint/eslint-plugin`，从导出的module里拿key为`recommended`的配置  
    + 如果没有前缀，也会自动拼接`eslint-plugin`前缀（[更多详情规则](https://juejin.cn/post/6923141007663955982)）：
        + 比如：`prettier`会查找`eslint-config-prettier`
        + 比如：`@vue/typescript`会查找`@vue/eslint-config-typescript`
```json
{
    "extends": [
        "eslint:recommended",
        "plugin:@typescript-eslint/recommended",
        "plugin:vue/vue3-essential",
        "prettier"
    ]
}
```
### 插件和自定义规则
+ 添加第三方ESLint插件：
```json
{
    "plugins": [
        "@typescript-eslint",
        "simple-import-sort",
        "prettier"
    ]
}
```
+ 开始自定义规则：
```json
{
    "rules": {
        "prettier/prettier": ["..."],
        "@typescript-eslint/no-var-required": "...",
        "simple-import-sort/imports": ["..."]
    }
}
```
### 不可忽视的隐藏配置Parser

+ ESLint提供了JS的默认Parser：
```json
{
    "parser": "espree"
}
```

+ 如果你想支持TypeScript，需要添加如下配置：
```json
{
    "parser": "@typescript-eslint/parser",
    "parserOptions": { "sourceType": "module" },
    "plugins": [ "@typescript-eslint" ]
}
```

+ 如果你想支持JSX语法：
```json
{
    "parser": "@typescript-eslint/parser",
    "parserOptions": {
        "ecmaFeatures": { "jsx": true }
    }
}
```

+ 如果你想支持Vue SFC `template`语法：
```json
{
    "parser": "vue-eslint-parser",
    // for JavaScript
    "parserOptions": {
        "sourceType": "module" 
    },
    "plugins": [ "vue" ]
}
```

但是Parser的配置通常在插件的预配置里已经包含，所以你并不用再写，也正是如此，很多人不一定知道ESLint的Parser配置非常重要。

汇总下来，你的ESLint配置看起来大概像这样：
```json
{
    "extends": [
        "plugin:@typescript-eslint/recommended",
        "prettier",
        "plugin:vue/vue3-essential",
        "eslint:recommended"
    ],
    "plugins": [
        "@typescript-eslint",
        "simple-import-sort",
        "prettier"
    ],
    "rules": {
        "prettier/prettier": ["..."],
        "@typescript-eslint/no-var-required": "...",
        "simple-import-sort/imports": ["..."]
        // ...
    }
}
```

对ESLint不熟的人，可能无法定位哪个extends关键词对应哪个包，也可能不清楚rules前面的key是对应哪个插件，这样的配置显然不够清晰。

## ESLint9 Flag Config可以让配置更简洁清晰

flat config允许任何插件、预设能有自己的完备的`root level`的配置，ESLint再自动将这些配置合并：

```js
const plugin = {
    meta: {},
    configs: {},
    rules: {},
    processors: {}
};

export default plugin
```

使用flat config特性，我个人非常推荐ESM格式的`eslint.config.mjs`文件来管理ESLint配置

### file Parttern + LanguageOptions + Envs 定范围

```js
import globals from "globals"

export default [
    {
        files: ["**/*.{js,mjs,cjs,ts,mts,jsx,tsx}"],
        languageOptions: {
            // common parser options, enable TypeScript and JSX
            parser: "@typescript-eslint/parser",
            parserOptions: {
                sourceType: "module",
                ecmaFeatures: { jsx: true }
            }
        }
    },
    // 支持vue SFC
    {
        files: ["*.vue", "**/*.vue"],
        languageOptions: {
            parser: "vue-eslint-parser",
            parserOptions: {
                // <script lang="ts" /> to enable TypeScript in Vue SFC
                parser: "@typescript-eslint/parser",
                sourceType: "module"
            }
        }
    },
    // 不同平台环境下的功能开关
    {
        languageOptions: {
            globals: {
                ...globals.browser,
                ...globals.node
            }
        }
    }
]
```

### 加载插件和推荐预设

```js
import jsLint from "@eslint/js"
import tsLint from "typescript-eslint"
import vueLint from "eslint-plugin-vue"

export default [
    jsLint.configs.recommended,
    ...tsLint.configs.recommended,
    ...vueLint.configs["flat/essential"],
]
```

### fixup做旧插件兼容

```js
import { fixupConfigRules } from "@eslint/compat"
import pluginReactConfig from "eslint-plugin-react/configs/recommended.js"

export default [
    ...fixupConfigRules(pluginReactConfig),
    {
        rules: {
            "react/react-in-jsx-scope": "off"
        }
    }
]
```

### 添加第三方插件

对于没有提供什么预设的，个人的第三方插件，我个人推荐且以`eslint-plugin-simple-import-sort`为例：

```js
import pluginSimpleImportSort from "eslint-plugin-simple-import-sort"

export default [
    {
        plugins: {
            // key "simple-import-sort" is the plugin namespace
            "simple-import-sort": pluginSimpleImportSort
        },
        rules: {
            "simple-import-sort/imports": [
                "error",
                { groups: [ "..." ] }
            ]
        }
    }
]
```

### 使用Stylistic处理冲突和风格检查
```js
import stylistic from "@stylistic/eslint-plugin"

export default [
    // disable legacy conflict rules about code style
    stylistic.configs["disable-legacy"],
    // you can customize or use a preset
    stylistic.configs.customize({
        indent: 4,
        quotes: "double",
        semi: false,
        commaDangle: "never",

        jsx: true
    })
]
```

最终，ESLint配置可以如此简单地组合：

```js
import globals from "globals"
import { fixupConfigRules } from "@eslint/compat"
import pluginReactConfig from "eslint-plugin-react/configs/recommended.js"

import jsLint from "@eslint/js"
import tsLint from "typescript-eslint"
import vueLint from "eslint-plugin-vue"
import stylistic from "@stylistic/eslint-plugin"

export default [
    // config parsers
    {
        files: ["**/*.{js,mjs,cjs,ts,mts,jsx,tsx}"],
    },
    {
        files: ["*.vue", "**/*.vue"],
        languageOptions: {
            parserOptions: {
                parser: "@typescript-eslint/parser",
                sourceType: "module"
            }
        }
    },
    // config envs
    {
        languageOptions: {
            globals: { ...globals.browser, ...globals.node }
        }
    },
    // syntax rules
    jsLint.configs.recommended,
    ...tsLint.configs.recommended,
    ...vueLint.configs["flat/essential"],
    ...fixupConfigRules(pluginReactConfig),
    // code style rules
    stylistic.configs["disable-legacy"],
    stylistic.configs.customize({
        indent: 4,
        quotes: "double",
        semi: false,
        commaDangle: "never",

        jsx: true
    })
]
```

幸运地是，ESLint9提供了CLI命令可以直接生成大部分配置：
```bash
npm init @eslint/config@latest
# or, if you aready installed ESLint
npx eslint --init
```

**End**

## 参考

1. ESLint 9 Realease Note: https://eslint.org/blog/2024/04/eslint-v9.0.0-released/
2. Migration Guide: https://eslint.org/docs/latest/use/migrate-to-9.0.0
3. Languate Options: https://eslint.org/docs/latest/use/configure/language-options
4. Gists: https://gist.github.com/aolyang/8ad9c14209b069806eac45b5927d00de
5. Stylistic: https://eslint.style/guide/getting-started
