---
title: 在实际项目中使用 zustand
date: 2024-03-09
---

<BlogCover src="/images/using-zustand-in-realworld-project/cover.jpg" />

# 在实际项目中使用 zustand

在项目重构进行技术选型时，选择了 zustand 作为项目的状态管理工具，随着项目需求迭代，以及阅读相关文章与源码，总结了一些实践经验

## 使用实践

### 将数据与函数分开定义

<!-- <Callout
  type="success"
  content="利用函数引用不会改变的特点，将修改数据状态的函数统一放在 action 或者 asyncAction 字段下"
/> -->

- 数据状态是随时会变化，而修改数据状态的函数不会。
- 在定义一个 store 结构的时候，建议将数据状态和函数分开定义，修改状态的同步函数与异步函数可以分别放在 **action** 和 **asyncAction** 字段下

```tsx
interface State {
  count: number
  // 同步操作
  action: {
    setCount: (payload: number) => void
  }

  // 异步操作
  asyncAction: {
    setCountAsync: () => void
  }
}
```

### 抽离常用 hook

<!-- <Callout
  type="success"
  content="业务代码里常用的数据，如维度树节点，用户信息等，应该统一抽离成 hook，放在文件夹下"
/> -->

- 在项目里，store 文件夹大概是这样组织的
  - 每一个 store 在 /hooks 下会有对应一个目录，存放对应一些常用的 hook

```JSON
├── model  // store文件夹
│    ├── hooks
│          ├── user-info                 // 用户信息相关hook
│              ├── useUserInfoAction.ts  // 修改用户信息的函数合集
│              ├── useUserId.ts          // 获取用户id
│              ├── useUsername.ts        // 获取用户名
                // ... ...
│              ├── index.ts              // 统一导出
├── user-info.ts                  // 用户信息store, 逻辑统一收敛在这里
├── navigation.ts                 // 导航store
├── index.ts                      // 统一导出
```

#### useXXXAction

- 结合前面说到函数引用是不会改变的，所以可以写一个 `useXXXAction` 统一将 action 字段下的函数统一导出
- 在业务代码使用时，直接使用解构语法即可，也不需要担心造成组件的 rerender

```tsx title="useUserInfoAction.ts" {4}
import { useStore } from '@/store'

export const useUserInfoAction = () => {
  return useStore((state) => state.action)
}
```

```tsx title="Component.tsx" {5}
import { useUserInfoAction } from '@/store/hooks/user-info'

export const Component = () => {
  // 通过解构按需获取函数
  const { setUserId, setUsername } = useUserInfo()
  // ...
}
```

#### useXXXState

- 在项目里频繁使用的 state，可以写一个 `useXXXState` 的 hook，避免在业务代码里重复书写 selector 语法

```ts title="useUserId.ts" {5}
import { useStore } from '@/store'

// 用户ID会频繁被使用到
export const useUserId = () => {
  return useStore((state) => state.userId)
}
```

```tsx {4,9}
import { useUserId } from '@/store/hooks/user-info'

export const ComponentA = () => {
  const userId = useUserId()
  // ... ...
}

export const ComponentB = () => {
  const userId = useUserId()
  // ... ...
}
```

### 精确引用数据

<!-- <Callout
  type="error"
  content="不要直接使用 useStore 与解构语法，组件会对 store 的所有数据进行订阅，造成不必要的重复渲染"
/> -->

- 在项目里，App.tsx 一开始是使用解构语法获取 `userId`，造成了 10+ 次渲染

```tsx title="App.tsx" {2}
const App: React.FC<AppProps> = (props) => {
  const { userInfo } = useUserInfoStore()

  const renderCount = useRef(0)
  console.log('app rerender', renderCount.current++)
  // ... ...
}
```

- 优化后：直接使用 `useUserId` 订阅更新，App 仅会渲染 2 次

```tsx title="App.tsx" {2}
const App: React.FC<AppProps> = (props) => {
  const userId = useUserId()

  const renderCount = useRef(0)
  console.log('app rerender', renderCount.current++)
  // ... ...
}
```

### 获取最新值

<!-- <Callout
  type="error"
  content="zustand 中的 useStore.getState() 能直接获取最新值，在很多时候往往有妙用"
/> -->

#### untrack

- 在某些场景里，我们希望在 `useEffect `或者 `useMemo` 里拿到某个数据的最新值，但是又不希望它作为依赖，直接调用 `useStore.getState()` 可以起到 类似 Solid.js [untrack](https://www.solidjs.com/tutorial/reactivity_untrack) 的作用

- 比如在项目中，stateA 和 stateB 同时作为 fetch 请求的依赖，但是 stateB 是 stateA 的一个 computed 值，我们希望仅将 stateB 作为依赖，其变化时去请求接口，但又要求在传参时能拿到 stateA 的最新值

![untrack](/images/using-zustand-in-realworld-project/untrack.png)

```tsx {5,11}
export const useFetchParams = () => {
  const stateB = useStateB()

  const fetchParams = useMemo(() => {
    const stateA = useStore.getState().stateA

    return {
      stateA,
      stateB,
    }
  }, [stateB])

  return fetchParams
}
```

#### 确保接口依赖正确

- 问题
  - 一些组件的数据需要请求接口，而接口会依赖多个入参。我们往往会在组件里调用多个 hook 订阅所有依赖的更新，当依赖逐渐增多时，组件与接口的状态会变得不可控

- 方案

  - 组件不通过 hook 订阅相关入参的数据更新，在外层统一进行订阅，做状态与请求处理
  - 每次请求时通过 `useStore.getState()` 拿到最新的值，确保请求的入参正确

- 例子
  - 在项目中筛选的下拉选项请求依赖多个入参
  - 通过外部 `useStore.subscribe` 等 hook 清空组件状态
  - 组件内部通过其他库如 formily 的 reaction 做请求驱动，直接使用 `useStore.getState()` 获取值，保证入参正确

![filter](/images/using-zustand-in-realworld-project/filter.png)

### 简单的表单联动

<!-- <Callout
  type="success"
  content="基于 zustand 可以做简单的表单联动，无需 formily 等表单解决方案"
/> -->

#### 值联动

- 最简单的一个场景就是 表单项A的值变化，表单项B的值需要清空

- 可以将表单项A的 value 存储在 store 中，通过 `useStore.subscribe()` 订阅对应依赖的更新，主动调用表单的 `onValueChange` 函数进行清空

![form-clear-value](/images/using-zustand-in-realworld-project/form-clear-value.png)

```tsx title="useClearValueAfterFilterChange.ts" {3,13-15}
export const useClearValueAfterFilterChange = (
  filterDeps: string[],
  onValueChange?: (value?: string) => void,
) => {
  useEffect(() => {
    return useFormStore.subscribe(
      (state) => state.filter,
      (currentFilter, prevFilter) => {
        if (filterDeps.length === 0) {
          return
        }

        const prevFilterDepsValue = filterDeps.map((dep) => prevFilter[dep])
        const currentFilterDepsValue = filterDeps.map((dep) => currentFilter[dep])

        if (!isEqual(prevFilterDepsValue, currentFilterDepsValue)) {
          onValueChange?.(undefined)
        }
      },
    )
  }, [filterDeps, onValueChange])
}
```

#### 下拉选项联动

- 稍微复杂的一个场景是表单项A的值变化，表单项B的值需要清空，下拉选项需要重新请求更新

- 在将 表单项A 的 value 存储在 store 的基础上，借助 useRequest deps 属性，可以订阅其更新，并重新发起请求

![form-update-options](/images/using-zustand-in-realworld-project/form-update-options.png)

```tsx title="useFetchOptions.ts" {2,3,13}
export const useFetchOptions = (requestApi: string, filterDeps: string[] = []) => {
  const stateA = useStateA()
  const deps = useMemo(() => [stateA], [stateA])

  const requestRes = useRequest(
    async () => {
      try {
        return fetch('xxx', { stateA })
      } catch (error) {
        return []
      }
    },
    { refreshDeps: deps },
  )

  return requestRes
}
```

### 与第三方框架结合

<!-- <Callout
  type="success"
  content="利用直接调用 getState() 和 setState() 的特性，可以将 React 与其他第三方框架例如 canvas 结合"
/> -->

- 问题
  - 在项目场景里，列表是基于 canvas 绘制的，点击事件是发生在 canvas 画布中，但渲染内容的 state 来自于 React，相当于需要有一个桥梁来告诉 React 去更新视图
  - 最简单直接的方法就是将 React 的 `setState` 方法传到 Table 的 column render 函数中，但是这样的场景一旦多起来，代码会变得不可维护

- 方案
  - 将有关数据的 state 存储在 **zustand** 中
  - canvas 监听到 onClick 事件后，通过 `useStore.getState().actions` 能直接拿到对应的 `setState()` 方法调用
  - 数据的改变会触发 React 更新，从而重新驱动 column render 函数执行，使 canvas 视图获得更新

![canvas](/images/using-zustand-in-realworld-project/canvas.png)

## 源码阅读

传入 **zustand** create API 的函数，会经过内部的数个函数，最后由 `createStoreImpl` 负责 store 的创建

1. createImpl
2. createStore
3. createStoreImpl

```tsx
const createImpl = <T,>(createState: StateCreator<T, [], []>) => {
  const api = typeof createState === 'function' ? createStore(createState) : createState
  // ... ...
}

const create = (<T,>(createState: StateCreator<T, [], []> | undefined) =>
  createState ? createImpl(createState) : createImpl) as Create

const createStore = ((createState) =>
  createState ? createStoreImpl(createState) : createStoreImpl) as CreateStore

// create store here
const createStoreImpl: CreateStoreImpl = (createState) => {
  // ... ...
}
```

### 创建 store

- store 的创建逻辑封装在 `createStoreImpl` 函数内
- 首先定义了如下方法，对应不同职责
  - setState 供外部修改状态
  - getState 供外部获取状态
  - subscribe 利用事件监听机制，在状态更新时触发视图更新
  - destroy 销毁时删除所有事件监听

```tsx
const getState: () => TState = () => state
const subscribe: (listener: Listener) => () => void = (listener) => {
  listeners.add(listener)
  // Unsubscribe
  return () => listeners.delete(listener)
}
// ... ...
```

- 然后将上述数个方法放在 api 对象中，调用我们在 create API 提供的函数，传入参数 (setState, getState, api) 创建出 state 状态

```ts
const api = { setState, getState, subscribe, destroy }
state = (createState as PopArgument<typeof createState>)(setState, getState, api)
```

- 最后将 api 对象透传到最外层，供 `createImpl` 捕获使用

### set

我们在回调函数内使用的 set 函数，接收 `partial` 和 `replace` 两个参数，主要会做以下事情

1. 判断 partial 是否为函数，如果是，传入当前 state 并调用，得到 nextState；否则直接取 partial 为nextState
2. 将 nextState 与 state 比较，如果不等，那么针对 object 数据类型使用 `Object.assign` 合并；否则直接赋值
3. 其中如果 replace = true，那么直接将 nextState 覆盖状态
4. 最后通知视图更新

```ts
const setState: SetStateInternal<TState> = (partial, replace) => {
  const nextState =
    typeof partial === 'function' ? (partial as (state: TState) => TState)(state) : partial
  if (nextState !== state) {
    const previousState = state
    state =
      replace ?? typeof nextState !== 'object'
        ? (nextState as TState)
        : Object.assign({}, state, nextState)
    listeners.forEach((listener) => listener(state, previousState))
  }
}
```

### get

我们在外部获取的 `get` 函数比较简单，只是单纯的返回状态的函数

```ts
const getState: () => TState = () => state
```

### 创建 state

最后将我们在代码里传入的函数与源码结合，实际形式是这样的，zustand 拿着 createState 函数，传入 set 等方法创建出 state

```ts
// 实际使用时传入的函数
const createState = (set) => ({
  data: 0,

  action: {
    setData: (payload: number) => {
      set({ data: payload });
    }
  },
})

export const useStore = create<State>(createState);

// 源码内部运行
const setState = // ...
const getState = // ...
const api = // ...
const state = createState(setState, getState, api)
```

### subscribe

经过上面几个步骤，只是单纯创建了状态，还没有与 React 建立联系。`createStoreImpl` 函数将整个 api 对象透传到了 `createImpl` 函数中

里面会生成 `useBoundStore` 函数，也就是最终通过 create API 获取的 hook

其会调用内部的 useStore 函数，利用 React 官方提供的 [useSyncExternalStoreWithSelector](https://reactjs.org/docs/hooks-reference.html#usesyncexternalstore) 让 api.subscribe 针对 selector 进行事件监听，与视图层进行关联，达到更新目的

```ts
export function useStore<TState, StateSlice>(
  api: WithReact<StoreApi<TState>>,
  selector: (state: TState) => StateSlice = api.getState as any,
  equalityFn?: (a: StateSlice, b: StateSlice) => boolean,
) {
  const slice = useSyncExternalStoreWithSelector(
    api.subscribe,
    api.getState,
    api.getServerState || api.getState,
    selector,
    equalityFn,
  )
  useDebugValue(slice)
  return slice
}

const useBoundStore: any = (selector?: any, equalityFn?: any) => useStore(api, selector, equalityFn)
```

## 参考

- [https://tkdodo.eu/blog/working-with-zustand](https://tkdodo.eu/blog/working-with-zustand)
- [https://react-tracked.js.org/docs/tutorial-zustand-01/](https://react-tracked.js.org/docs/tutorial-zustand-01/)
