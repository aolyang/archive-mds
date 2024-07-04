When we start a new project, syntax check and style format is important but not easy to config.

That is because, before ESLint 9, it had many conflicts between IDE/Editor, prettier, and ESLint. Now ESLint9 disabled & deprecated some confict rules, and enabled Flat config as default.
(ESLint 9.0 stable version published in [2024/4/6](https://www.npmjs.com/package/eslint?activeTab=versions), and developed quickly)

In this tutorial, I have created a [Gist](https://gist.github.com/aolyang/8ad9c14209b069806eac45b5927d00de) based on ESLint 9 for `Vue3 + TypeScript`, supported `Template` and `JSX`.

1. Vue3 + TypeScript.
2. The Official recommended config.
3. Syntax check by ESLint, and style format by Stylistic.
4. Conflicts disabled by Stylistic preset `disable-legacy` config.
5. Extra plugin `simple-import-sort`, and shows how eslint9 is compact with plugins (old)

If you are a pro with ESLint, just straightforward to the [Gist](https://gist.github.com/aolyang/8ad9c14209b069806eac45b5927d00de) content to save your time, please leave a comment to improve the config.

Let's get started!

## How to Config ESLint 8

+ to extend preset configs, before eslint 9, you need:
```json
{
    "extends": [
        "plugin:@typescript-eslint/recommended",
        "prettier",
        "plugin:vue/vue3-essential",
        "eslint:recommended"
    ]
}
```
+ to use eslint plugins, before eslint 9, you need:
```json
{
    "plugins": [
        "@typescript-eslint",
        "simple-import-sort",
        "prettier"
    ]
}
```
+ then you can config eslint rules:
```json
{
    "rules": {
        "prettier/prettier": ["..."],
        "@typescript-eslint/no-var-required": "...",
        "simple-import-sort/imports": ["..."]
    }
}
```
+ Oh, there are some hidden configs you need to know ([parser options](https://eslint.org/docs/v8.x/use/configure/parser)):
```json
{
    "parser": "espree" // default JavaScript parser provided by eslint
}
```
if you want eslint to support TypeScript, you need add config like this:
(also, this base config is provided by `@typescript-eslint/parser` plugin)
```json
{
    "parser": "@typescript-eslint/parser",
    "parserOptions": { "sourceType": "module" },
    "plugins": [ "@typescript-eslint" ]
}
```
to support Vue SFC `template` syntax, the hidden base config behind `eslint-plugin-vue` is:
```json
{
    "parser": "vue-eslint-parser",
    // for JavaScript
    "parserOptions": { "module": 2020, "sourceType": "module" },
    "plugins": [ "vue" ]
}
```
to support JSX syntax:

```json
{
    "parser": "@typescript-eslint/parser", // this is not only one choice
    "parserOptions": {
        "ecmaFeatures": { "jsx": true }
    }
}
```
According to the above, you can get a resolved ESLint 8 config for vue3 like below:
```json
{
    "parser": "eslint-plugin-vue",
    "parserOptions": {
        // <script lang="ts" /> to enable TypeScript in Vue SFC
        "parser": "@typescript-eslint/parser",
        "sourceType": "module",
        "ecmaFeatures": { "jsx": true }
    },
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
        "prettier/prettier": [ "..." ],
        "@typescript-eslint/no-var-required": "...",
        "simple-import-sort/imports": [ "..." ]
    }
}
```
It's aready looks like a simple config, but:
1. parser config is hiddened by Domain Specific Language (DSL) plugin, you need to know the base config.
2. the preset config name you want to extend is loaded hiddenly by ESLint, you may lose some info.
3. you need write each DSL rules together, it's not easy to maintain.
4. syntax and style rules are mixed and make conflicts with prettier.

## Really simple flat config in ESLint 9

1. Create a config file, I recommend `eslint.config.mjs` to use ESM module.
    ```js
    export default [
      // your config here
    ]
    ```
2. In ESLint 9, parser and DSL support upgrade to [languageOptions](https://eslint.org/docs/latest/use/configure/language-options) which more clearly:

    ```js
    export default [
        {
            files: ["**/*.{js,mjs,cjs,ts,mts,jsx,tsx}"],
            languageOptions: {
                // common parser options, enable TypeScript and JSX
                parser: "@typescript-eslint/parser",
                parserOptions: {
                    sourceType: "module"
                }
            }
        },
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
        }
    ]
    ```
3. Config code running environment, `globals` holding a bunch of flags for `Browser` and `Node.js`:
   (you can find details in `node_modules/globals/globals.json`)
    ```js
    import globals from "globals"

    export default [
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
4. add preset configs
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
5. fixup old config rules and change some values
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
6. add a third-party plugin.
    > To configure plugins inside of a configuration file, use the plugins key, which contains an object with properties representing plugin namespaces and values equal to the plugin object.

    `simple-import-sort` is one of my favorite plugins I really recommend. Group and sort imports can make your code more readable.

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
7. What's more, use [stylistic](https://eslint.style/guide/getting-started) to handle non-syntax code style format:
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
Finally, you can get a really simple flat config ([github Gist](https://gist.github.com/aolyang/8ad9c14209b069806eac45b5927d00de)):
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
Luckily, ESLint provided a friendly CLI command to generate most of the config:
```bash
npm init @eslint/config@latest
# or, if you aready installed ESLint
npx eslint --init
```
Except custommized rules and **stylistic**.

**End**

## Links

1. ESLint 9 Realease Note: https://eslint.org/blog/2024/04/eslint-v9.0.0-released/
2. Migration Guide: https://eslint.org/docs/latest/use/migrate-to-9.0.0
3. Languate Options: https://eslint.org/docs/latest/use/configure/language-options
4. Gists: https://gist.github.com/aolyang/8ad9c14209b069806eac45b5927d00de
5. Stylistic: https://eslint.style/guide/getting-started
