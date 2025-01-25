---
title: 使用 Solid.js 模仿实现 sonner
date: 2024-02-25
description: 记录使用 Solid.js 实现 sooner 的过程
---

<BlogCover src="/images/build-solid-sonner-toast/cover.jpg" />

# 使用 Solid.js 模仿实现 sonner

[sonner](https://sonner.emilkowal.ski/) 是一个优秀的 toast 组件，想了解其实现的最好方法就是动手自己实现一遍，但是重复使用 React 实现未免太过枯燥

所以我选择了使用 Solid.js，尽管 Github 上已经存在了[solid-sonner](https://github.com/wobsoriano/solid-sonner)，但仍在实现的过程中学到了一些东西，你可以在这里访问 [文档](https://solid-sonner-toast-website.vercel.app/docs/getting-started) 与 [源码]((https://github.com/Ken-HH24/solid-sonner-toast))

## 状态管理

[sonner](https://sonner.emilkowal.ski/) 官方使用了 **Observer Pattern** 去管理 `toasts` 的状态，在 React Component 里通过 useEffect 去订阅变化并更新视图

但是对于 Solid.js，利用 fine grained reactivity，可以直接外部的任意地方使用 `createStore()` 去管理 toasts 的状态

只需 `toast()` 函数被执行时，通过 `setToasts()` 即可驱动视图的更新即可

```ts
class ToastState {
  toasts: ToastT[]
  setToasts: SetStoreFunction<ToastT[]>

  constructor() {
    const [toasts, setToasts] = createStore<ToastT[]>([])
    this.toasts = toasts
    this.setToasts = setToasts
  }

  addToast = (toast: ToastT) => {
    this.setToasts(
      produce((toasts) => {
        toasts.unshift(toast)
      }),
    )
  }

  createToast = (data: ExternalToast & { message?: string | JSXElement }) => {
    // ... ...
  }

  message = (message: string | JSXElement, data?: ExternalToast) => {
    this.createToast({ ...data, message })
  }

  // ... ...
}
```

## Reactivity

Solid.js 和 React 的不同主要在于 Solid.js 仅执行一次组件函数

所以如果想变量是响应式的，需要额外使用函数包裹，Solid.js 才会将变化更新到视图

比如在函数内将 `props.position.split("-")` 赋值给 `coords` 变量，然后在渲染时使用，该变量并不会跟随 `props.position` 的改变

```tsx
const Toaster = (props) => {
  const coords = props.position.split('-')

  // won't change with props.position
  return <div>{coords[0]}</div>
}
```

需要额外使用箭头函数包裹，然后在 UI 里执行，Solid.js 才知道它是响应式的

```tsx
const Toaster = (props) => {
  const coords = () => props.position.split('-')

  // update when props.position changes
  return <div>{coords()[0]}</div>
}
```

## 动画

[sonner](https://sonner.emilkowal.ski/) 利用 css变量 以及 `data-*` 将动画拆分成了四步

1. 初始：`opacity=0` && `translateY(100%)`，此时 `toast` 不可见，执行进场动画
2. 挂载：`opacity=1` and `translateY(0)`，此时 `toast` 显示到对应位置
3. 移除：`opacity=0` and `translateY(100%)`，此时 `toast` 被标记移除，执行动画
4. 卸载：`toast` 对应的 DOM 元素被卸载

在实现过程中踩的一个坑是需要注意 `data-*={variable}` 中的 `variable` 是否为 undefined，最好使用 `Boolean()` 规避这个问题

```tsx
<div
  // use Boolean to avoid undefined
  data-expanded={Boolean(props.expanded)}
/>
```

## JSX

当我第一次实现的时候，很自然地写下以下代码

```tsx
const SuccessIcon = <svg>{/* ... ... */}</svg>

const Toast = () => {
  return <div>{SuccessIcon}</div>
}
```

但是当有多个 toast 时，仅有一个 `<SuccessIcon />` 正常显示

在 Solid.js 的 playground 可以看到被编译为以下代码

```tsx {1,4,6}
import { template as _$template } from 'solid-js/web'
import { insert as _$insert } from 'solid-js/web'

const _tmpl$ = /*#__PURE__*/ _$template(`<svg />`),
  _tmpl$2 = /*#__PURE__*/ _$template(`<div>`)
const SuccessIcon = _tmpl$()

const Toast = () => {
  return (() => {
    const _el$2 = _tmpl$2()
    _$insert(_el$2, SuccessIcon)
    return _el$2
  })()
}
```

`<ScuuessIcon />` 是一个被创建好的元素，当有多个 toast 渲染时，Solid.js 使用 `document.appendChild()` 将该元素重复渲染到多个 `toast` 上，最终仅有最后一个 `toast` 得到了所有权

修复这个问题则是将 `SuccessIcon` 改写为函数的形式，让 Solid.js 渲染不同的实例

```tsx
const SuccessIcon = () => <svg>{/* ... ... */}</svg>
```

## 官网

Solid.js 没有像 React 一样有很多开箱即用的建站框架，Astro 通常是第一选择，但是也许是为了折腾，我使用了 [SolidStart](https://start.solidjs.com/getting-started/what-is-solidstart) 去打造官网

[SolidStart](https://start.solidjs.com/getting-started/what-is-solidstart) 没有太多复杂的概念，其底层使用了 [Nitro](https://nitro.unjs.io/) 作为 Server（该框架也是 [Nuxt](https://nuxt.com/) 的 Server），这样也达到了生态共享的目的

### MDX

SolidStart 已经提供了开箱即用的 **MDX** 支持，只需要在 **routes** 目录下书写 `.mdx` 即可渲染内容

但我个人其实比较喜欢 [Contentlayer](https://contentlayer.dev/) 与 [Next.js](https://nextjs.org/) 的组合方式，即

1. 在 contents 目录下书写文档
2. 利用 Contentlayer 生成内容
3. 在 [slug].tsx 里拿到对应的内容，使用 `useMDXComponent()` 渲染

但很可惜 Contentlayer 官方仅提供了与 [Next.js](https://nextjs.org/) 的结合方式，其他框架例如 Vite 都仍在计划中

#### Contentlayer 与 mdx-bundler

参考 [Contentlayer](https://contentlayer.dev/) 里 `useMDXComponent()` 和 [mdx-bundler](https://github.com/kentcdodds/mdx-bundler) 的源码，要达到上述效果，二者都做了一下事情

1. 将 **MDX** 文件编译为 **iife** 形式的代码
2. 在 [slug].tsx 里拿到对应文章的代码，执行 `new Function()` 得到渲染的 Component

```js title="mdx-bundler" {3}
function getMDXExport(code, globals) {
  const scope = { React, ReactDOM, _jsx_runtime, ...globals }
  const fn = new Function(...Object.keys(scope), code)
  return fn(...Object.values(scope))
}
```

#### 基于 rollup 实现

目前思路已经很明确了，那就是：

1. 基于 rollup 实现打包（最初是使用 esbuild，但是与 Solid.js 结合的配置没有弄出来😭），每个 mdx 文件都分别作为入口，最后输出的是与 Contentlayer 相似的 json 产物
2. 使用 `@mdx-js/rollup` 和 `babel` 将 MDX 编译为 IIFE代码 `return Component`
3. 在 **renderChunk** 阶段拿到最后编译好的代码，使用 `this.emitFile()` 输出 json 文件
4. 在 **generateBundle** 阶段将 js 文件从 bundle 里删除，产物里无需包含

第三与第四步我分别写了两个 rollup plugin 去实现

```ts title="inject-mex-data" {10-17}
export const injectMDXData = () => {
  return {
    name: 'inject-mdx-data',
    renderChunk(code, chunk) {
      const moduleId = chunk.facadeModuleId || ''
      const moduleMeta = this.getModuleInfo(moduleId)?.meta
      const contentData = moduleMeta?.contentData || {}

      this.emitFile({
        type: 'asset',
        fileName: `${chunk.name}.json`,
        source: JSON.stringify({
          code,
          ...contentData,
        }),
      })
    },
  }
}
```

最后的 `rollup.config.ts` 为

```ts title="rollup.config.ts"
const rollupOptions = inputs.map(
  (input): RollupOptions => ({
    input: input.path,
    output: {
      dir: 'contents',
      name: 'Component',
      format: 'iife',
      footer: '; return Component',
    },
    plugins: [
      injectMDXData(),
      nodeResolve(),
      mdx({
        jsxImportSource: 'solid-jsx',
        jsx: true,
        // ... ...
      }),
      typescript(),
      babel({
        presets: ['solid'],
        babelHelpers: 'bundled',
        extensions: ['.js', '.jsx', '.cjs', '.mjs', '.md', '.mdx', '.tsx', 'ts'],
      }),
      renderMDXData(),
    ],
  }),
)
```

#### JSX 交由 babel 编译

实现的过程当然也不是一帆风顺，例如 `@mdx-js/rollup` 插件的 **jsx** 选项没有开启，这个时候会导致编译出的 Solid.js 代码失去响应式

可以看到在传给 `Tab` 组件的对象里，children 前都缺少了 **getter** 修饰

```ts title="getting-startted.json" {3}
children: [d(Tab, {
          value: "pnpm",
          children: d(_components.figure, {
          // ... ...
          })})]
```

当插件开启 **jsx** 选项后，jsx 交由了 babel 编译，利用 Solid.js 的 preset 将组件编译为响应式

```ts title="rollup.config.ts" {3}
mdx({
  jsxImportSource: 'solid-jsx',
  jsx: true,
  // ... ...
}),
```

编译的结果为：

```ts title="getting-startted.json" {3}
return [createComponent(Tab, {
          value: "pnpm",
          get children() {
          // ... ...
        }})]
```

#### Comp is not a function

遇到的另外一个问题是运行时报错 **Comp is not a function**，解决方法是在解决上一个问题的同时无意发现的

原因就是 `createComponent()` 接收的都是函数组件，但是 MDX 原生的如 **h1** 为字符串，在开启 **jsx=false** 时，编译出的函数会兼容字符串形式，但是又与上一个问题的解决方法矛盾了

因此有两种其他解决方法
1. 在调用 `new Function()` 得到 Component 后，传入 **components**，将 h1 这些字符串标签改写为函数组件
2. 增加 babel 插件将之前编译得到的 `_components` 对象改写

最后碍于时间的关系，采用了第一种方法

```tsx title="mdx-component.tsx" {2,12}
const components = {
  a: (props: any) => <a {...props}>{props?.children}</a>,
  // ...
}

const MDXComponent = (props: MDXComponentProps) => {
  const component = createMemo(() => {
    if (props.mdx) {
      // ... ...
      const fn = new Function(...Object.keys(scope), code)
      const Comp = fn(...Object.values(scope))?.default
      return () => <Comp components={components} />
    }
    return undefined
  })
  return <Dynamic component={component()} />
}
```

#### 给 MDX 增加其他信息
在使用 rollup 打包输出 json 的时候还希望包含一些额外的信息，如 toc，这个可以放在 **transform** 的阶段去做，将信息注入 meta 里，然后在 **renderChunk** 的时候一并拿出来，具体做法参考了 [shadcn-ui](https://github.com/shadcn-ui/ui) 生成 TOC 的做法

```ts title="inject-mdx-data.ts" {8-9, 12}
async transform(code, id) {
  if (/.(mdx|md)$/.test(id)) {
    const result = await unified()
      // ...

    const contentData = {
      raw: code,
      toc: result.data?.toc,
      matter: result.data?.matter,
    }

    return { meta: { contentData } }
  }
},
```

## 部署
关于部署比较简单，Nitro 提供了开箱即用的配置，我选择了 **vercel** 作为部署平台，因此只需要增加一个 presets 配置即可

```ts title="vite.config.ts" {4}
export default defineConfig({
  start: {
    server: {
      preset: 'vercel',
      prerender: {
        routes: ['/example', '/docs/getting-started'],
      },
    },
    extensions: ['mdx', 'md'],
  },
  // ... ...
})
```