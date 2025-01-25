---
title: 项目工程化配置
date: 2024-11-23
---

工作中经常会遇到需要对项目进行工程化配置的情况，前端通常离不开 husky, commitlint, lint-staged, eslint 等工具。恰好最近在一个新项目有一个重新练手的机会，所以做了以下整理

## husky

[husky](https://typicode.github.io/husky/) 的定位就像其在官网里所写的那样，在 commit，push 等 git 生命周期中执行 lint，测试等命令。

执行完 `npx husky init` 命令后会在项目根目录下生成一个 `.husky` 目录，在该目录下有一个 `pre-commit` 文件，该文件就是在 git commit 前执行的命令。如果配置上 `npm run test` 命令，那么在执行 git commit 前会先执行 `npm run test` 命令。

如果想添加其他命令，那么可以在 .husky 目录下继续新建文件，例如 `commit-msg`，然后添加相应的命令，git 所有 hook 可以在这里查看 [git hooks](https://git-scm.com/docs/githooks#_hooks)

## commitlint

[commitlint](https://commitlint.js.org/) 的作用是保证提交的 commit 符合规范。而不是产生如 `测试提交` 这样完全没有意义的 commit。

配置的方法也比较简单，在安装好相关依赖后，在项目根目录下新建一个 `commitlint.config.js` 文件进行配置

### rules

`rules` 为数组形式，最多接收三个参数，Level，Applicable 和 Value。分别表示提示等级，是否开启 以及 规则具体配置

所有 rule 的详细配置可查看 [commitlint rules](https://commitlint.js.org/reference/rules.html)，例如对 `type` 进行配置：

::: code-group

```json [.commitlintrc] {3-7}
{
  "rules": {
    "type-enum": [
      2,
      "always",
      ["fix", "build", "chore", "ci", "docs", "feat", "perf", "refactor", "revert", "style", "test", "type"]
    ]
  }
}
```

:::

## lint-staged

[lint-staged](https://github.com/lint-staged/lint-staged#readme) 的作用是让 eslint, prettier 等工具只对暂存区的文件进行检查，而不是对整个项目进行检查。这样能节省时间，同时避免产生不必要的更改。

通常的做法是在 `package.json` 或者新建 `.lintstagedrc` 文件进行命令配置。然后在 `pre commit` 用 `lint-staged` 命令进行替代。例如：

::: code-group

```json [.lintstagedrc]
{
  "*.{ts,tsx,js,jsx}": ["eslint --max-warnings=0 --fix"]
}
```

:::

此时在提交 commit 的时，会对 **暂存区内** `.ts` 等文件执行 `eslint --max-warnings=0 --fix` 命令，而不是整个仓库

## eslint

[eslint](https://eslint.org/) 相信很多人都比较熟悉了，作用就是检查代码质量，用各种规则做约束。

自从升级到 v9 后，eslint 的配置方式发生了比较大的变化，官方称其为[flat configuration](https://eslint.org/docs/latest/use/configure/configuration-files)。将旧版本的配置文件进行升级也很简单，官方提供迁移命令，可解决大部分问题：

```bash
npx @eslint/migrate-config .eslintrc.json
```

如果有插件规则在迁移至 v9 之后无法正常使用，可以尝试使用 [eslint-plugin-compat](https://github.com/amilajack/eslint-plugin-compat) 兼容。

而原来的 `extends` 配置则是被废弃。可以通过 `FlatCompat.extends` 进行转换

```js
import { FlatCompat } from '@eslint/eslintrc';
import { fixupConfigRules } from '@eslint/compat';

const compat = new FlatCompat({
  baseDirectory: __dirname,
  recommendedConfig: js.configs.recommended,
  allConfig: js.configs.all,
});

const originalConfig = compat.extends('xxx-rules');

export default [
  ...fixupConfigRules(originalConfig).map(config => ({
    ...config,
    files: ['**/*.ts', '**/*.tsx'],
  })),
];
```
