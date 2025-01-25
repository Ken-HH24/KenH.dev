---
title: Building a port of sonner using Solid.js
date: 2024-02-25
---

<BlogCover src="/images/build-solid-sonner-toast/cover.jpg" />

# Building a port of sonner using Solid.js

_**This Article is translated by ChatGPT**_

[sonner](https://sonner.emilkowal.ski/) is an excellent toast component, and the best way to understand its implementation is to get hands-on and implement it yourself.

However, repeating the implementation using React may be a bit tedious. Therefore, I chose to use Solid.js for this task (even though there is already [solid-sonner](https://github.com/wobsoriano/solid-sonner)).

Nevertheless, I still learned something from building the component on my own.

You can visit the **[site](https://solid-sonner-toast-website.vercel.app/docs/getting-started)** and **[source code](https://github.com/Ken-HH24/solid-sonner-toast)** here

## State management

[sonner](https://sonner.emilkowal.ski/) officially uses the **Observer Pattern** to manage the state of toasts. In React components, the state changes are subscribed and the view is updated through the use of useEffect.

However, for Solid.js, leveraging fine-grained reactivity, I can directly use `createStore()` from anywhere to manage the state of toasts.

When the `toast()` function is executed, the view can be updated by calling `setToasts()`.

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

The main distinction between Solid.js and React lies in Solid.js executing the function only once.

Therefore, to make a variable reactive, it is necessary to wrap it in a function so that Solid.js can update the changes in view exactly.

For example, if you assign `props.position.split("-")` to the variable `coords` within the function and then use it in rendering, this variable won't automatically update with `props.position`.

- To ensure reactivity, you need to wrap such assignments in a function in Solid.js.

```tsx
const Toaster = (props) => {
  const coords = props.position.split('-')

  // won't change with props.position
  return <div>{coords[0]}</div>
}
```

- To make variables reactive in Solid.js, it's necessary to wrap them in functions and execute them within the UI so that Solid.js recognizes them as reactive.

```tsx
const Toaster = (props) => {
  const coords = () => props.position.split('-')

  // update when props.position changes
  return <div>{coords()[0]}</div>
}
```

## animate the toast

[sonner](https://sonner.emilkowal.ski/) utilizes `CSS variables` and `data-\*` attributes to break down the animation into four steps:

1. Initial: `opacity=0` && `translateY(100%)`, at this point, the `toast` is invisible, executing the entrance animation.
2. Mounting: `opacity=1` and `translateY(0)`, now the `toast` is displayed at the corresponding position.
3. Removal: `opacity=0` and `translateY(100%)`, the `toast` is marked for removal, triggering the animation.
4. Unmount: The DOM element corresponding to the `toast` is unmounted.

One pitfall encountered during the implementation is the need to pay attention to whether the variable in `data-\*={variable}` is undefined. It is recommended to use `Boolean()` to avoid this issue.

```tsx
<div
  // use Boolean to avoid undefined
  data-expanded={Boolean(props.expanded)}
/>
```

## JSX

When I implemented it for the first time, I naturally wrote down the following code just like React.

```tsx
const SuccessIcon = <svg>{/* ... ... */}</svg>

const Toast = () => {
  return <div>{SuccessIcon}</div>
}
```

But when there are multiple `toasts`, only one `<SuccessIcon />` is displayed.

In the Solid.js playground, we can see that it is compiled into the following code:

```tsx
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

`<SuccessIcon />` is a pre-created element. When multiple toasts are rendered, Solid.js uses `document.appendChild()` to duplicate this element onto multiple toast components. In the end, only the last `toast` retains ownership.

To fix this issue, you can rewrite `SuccessIcon` as a function, allowing Solid.js to render different instances.

```tsx
const SuccessIcon = () => <svg>{/* ... ... */}</svg>
```

## Website

Solid.js doesn't have lots of meta-frameworks as React. **Astro** is often the first choice for building docs site. But I give a shot of [SolidStart](https://start.solidjs.com/getting-started/what-is-solidstart) to build the official website.

[SolidStart](https://start.solidjs.com/getting-started/what-is-solidstart) doesn't introduce too many complex concepts. It utilizes Nitro as the server (which is also the server for Nuxt) which achieves the goal of ecosystem sharing.

### MDX

SolidStart has provided out-of-the-box **MDX** support. You just need to write `.mdx` in the routes directory to render the content.

However, personally, I prefer the combination of [Contentlayer](https://contentlayer.dev/) and [Next.js](https://nextjs.org/), which involves:

1. Writing documents in the contents directory.
2. Generating content using Contentlayer.
3. In the [slug].tsx file, fetching the corresponding content and rendering it using `useMDXComponent()`.

Unfortunately, Contentlayer officially only supports integration with Next.js, and support for other frameworks such as Vite is still in the planning stage.

#### How Contentlayer and mdx-bundler do

In reference to the `useMDXComponent()` function in [Contentlayer](https://contentlayer.dev/) and the source code of [mdx-bundler](https://github.com/kentcdodds/mdx-bundler), achieving the mentioned effect involves two main steps:

1. Compiling the MDX file into **IIFE** code (Immediately Invoked Function Expression).
2. In the [slug].tsx file, obtaining the code for the corresponding article and executing it using `new Function()` to get the rendered component.

```js title="mdx-bundler" {3}
function getMDXExport(code, globals) {
  const scope = { React, ReactDOM, _jsx_runtime, ...globals }
  const fn = new Function(...Object.keys(scope), code)
  return fn(...Object.values(scope))
}
```

This approach allows dynamic rendering of MDX content within the specified framework, facilitating a flexible and efficient integration of Markdown-based content into your application.

#### Implementing with Rollup

The approach is quite clear:

1. Implement bundling based on Rollup (initially attempted with esbuild, but struggled with the configuration when integrating with Solid.js 😭). Each MDX file is a separate entry point, and the final output is a JSON artifact similar to Contentlayer.
2. Use `@mdx-js/rollup` and `babel` to compile MDX into IIFE code, appending `return Component` at the end of the code.
3. In the **renderChunk** phase, obtain the final compiled code and use `this.emitFile()` to output a JSON file.
4. In the **generateBundle** phase, remove the JS files from the bundle; the final artifact should not include them.

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

The `rollup.conffig.ts` is

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

#### Handle JSX compilation to Babel

The implementation process, of course, has its challenges. For instance, when the jsx option of the `@mdx-js/rollup` plugin is not enabled, it can result in non-reactive Solid.js code.

It can be observed that in the object passed to the `Tab` component, the **getter** modifier is missing before the children property.

```ts title="getting-startted.json" {3}
children: [
  d(Tab, {
    value: 'pnpm',
    children: d(_components.figure, {
      // ... ...
    }),
  }),
]
```

When the plugin has the **jsx** option enabled, JSX is handed over to Babel for compilation. Leveraging Solid.js's preset, the components are compiled to be reactive.

```ts title="rollup.config.ts" {3}
mdx({
  jsxImportSource: 'solid-jsx',
  jsx: true,
  // ... ...
}),
```

The compilation result is:

```ts title="getting-startted.json" {3}
return [createComponent(Tab, {
          value: "pnpm",
          get children() {
          // ... ...
        }})]
```

#### Comp is not a function

Encountered another issue at runtime, namely the error **Comp is not a function**. The solution was stumbled upon while addressing the previous problem.

The issue stemmed from `createComponent()` expecting function components, but the native MDX elements like **h1** are represented as strings. When using `jsx=false`, the compiled functions are compatible with string representations, creating a contradiction with the solution to the previous problem.

Therefore, there are two alternative solutions:

1. After obtaining the Component with `new Function()`, pass `components` attributes into component and rewrite string tags like h1 as function components.
2. Add a Babel plugin to rewrite the compiled `_components` object.

Ultimately, due to time constraints, the first approach was adopted.

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

#### Add additional information to MDX

When using Rollup to package and output JSON, if you want to include additional information such as a Table of Contents (TOC), you can handle this in the `transform` phase.

During this stage, inject the information into the meta, and then retrieve it together in the `renderChunk` phase. The approach of generating TOC is referenced from [shadcn-ui](https://github.com/shadcn-ui/ui).

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

## Deploy

Regarding deployment, it's relatively straightforward. Nitro provides out-of-the-box configurations, and I opted for Vercel as the deployment platform. Therefore, only an additional `presets` configuration needs to be added.

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
