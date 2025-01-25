---
title: Using Zustand in real world projects
date: 2024-03-09
---

<BlogCover src="/images/using-zustand-in-realworld-project/cover.jpg" />

# Using Zustand in real world projects

<!-- import Callout from '@/components/callout' -->

_**This Article is translated by ChatGPT**_

During the technical selection for project refactoring, Zustand was chosen as the state management tool. 

With the iteration of project and reading relevant articles and source code, I have summarized some practical experiences.

## Usage in practice

### Separating the definition of data and functions

<!-- <Callout
  type="success"
  content="Using the immutability of function references, consolidating functions that modify data state under the action or asyncAction field."
/> -->

- The state of data is subject to constant change, while functions remain unchanged.
- When defining a store structure, it is recommended to separate the definition of data state and functions. Synchronous functions for modifying the state can be placed under the **action**, while asynchronous functions can be placed under the **asyncAction**.

```tsx {3-5,7-9}
interface State {
  count: number
  action: {
    setCount: (payload: number) => void
  }

  asyncAction: {
    setCountAsync: () => void
  }
}
```

### Extracting commonly used hooks

<!-- <Callout
  type="success"
  content="Commonly used data in code, such as user information, etc., should be extracted into hooks and placed in a folder."
/> -->

- In the project, the organization of the store folder is approximately as follows:
  - Each store will have a corresponding directory in **/hooks**, storing some commonly used hooks related to it.

```JSON
├── model  // store folder
│    ├── hooks
│          ├── user-info                 // the collection of hooks that are relevant to user's info
│              ├── useUserInfoAction.ts  // functions that modify user's state
│              ├── useUserId.ts          // get user id
│              ├── useUsername.ts        // get username
                // ... ...
│              ├── index.ts
├── user-info.ts                  // the logic of processing user's info
├── navigation.ts                 // the logic of navigation
├── index.ts
```

#### useXXXAction

- Considering the earlier discussion that function references do not change, you can create a `useXXXAction` to uniformly export functions under the action field.

- When using in code, simply use destructuring syntax, and there is no need to worry about causing rerenders of components.

```tsx title="useUserInfoAction.ts" {4}
import { useStore } from '@/store'

export const useUserInfoAction = () => {
  return useStore((state) => state.action)
}
```

```tsx title="Component.tsx" {4}
import { useUserInfoAction } from '@/store/hooks/user-info'

export const Component = () => {
  const { setUserId, setUsername } = useUserInfo()
  // ...
}
```

#### useXXXState

- For frequently used states in the project, you can create a `useXXXState` hook to avoid repetitive writing of selector syntax in business code.

```ts title="useUserId.ts" {4}
import { useStore } from '@/store'

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

### Precisely reference data
<!-- 
<Callout
  type="error"
  content="Avoid using useStore directly with destructuring syntax, as it causes components to subscribe to all data in the store, leading to unnecessary re-renders."
/> -->

In the initial implementation of App.tsx, using destructuring to obtain `userId` resulted in more than 10 renders.

```tsx title="App.tsx" {2}
const App: React.FC<AppProps> = (props) => {
  const { userInfo } = useUserInfoStore()

  const renderCount = useRef(0)
  console.log('app rerender', renderCount.current++)
  // ... ...
}
```

- After optimization: Directly using `useUserId` to subscribe and update, App will only render twice.

```tsx title="App.tsx" {2}
const App: React.FC<AppProps> = (props) => {
  const userId = useUserId()

  const renderCount = useRef(0)
  console.log('app rerender', renderCount.current++)
  // ... ...
}
```

### Get the latest value

<!-- <Callout
  type="success"
  content="useStore.getState() in Zustand allows direct retrieval of the latest value, proving to be quite useful in many scenarios."
/> -->

#### untrack

- In certain scenarios, there might be a desire to obtain the latest value of a specific data inside `useEffect` or `useMemo` without including it as a dependency. Directly calling `useStore.getState()` can serve a purpose similar to Solid.js [untrack](https://www.solidjs.com/tutorial/reactivity_untrack).

- In the project, for example, where stateA and stateB both serve as dependencies for a fetch request, and stateB is a computed value based on stateA, you might want to use only stateB as a dependency to trigger the API request when it changes. However, you also need to ensure that you can obtain the latest value of stateA when passing parameters. In such a case, you can utilize `useStore.getState()` to access the latest value of stateA without including it as a direct dependency.

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

#### Ensure correct API dependencies

- Problem:
  - Some components require data from API requests, and these requests depend on multiple parameters. Often, we call multiple hooks in the component to subscribe to updates for all dependencies. As the dependencies increase, the state management between the component and the API becomes uncontrollable.

- Solution:
  - Instead of subscribing to data updates for relevant parameters through hooks within the component, perform a unified subscription at the outer layer for state and request handling.
  - When making requests, `use useStore.getState()` to obtain the latest values, ensuring that the request parameters are correct.

- Example:
  - When filtering dropdown options in the project with API requests depending on multiple parameters:
  - Use external hooks like `useStore.subscribe` to clear component state.
  - Internally, leverage other libraries like [formily](https://formilyjs.org/)'s reaction for request-driven behavior, and directly use `useStore.getState()` to get the latest values, ensuring correct parameter usage.

![filter](/images/using-zustand-in-realworld-project/filter.png)

### Simple form reaction

_With Zustand, you can achieve simple form reaction without the need for additional form solutions like [formily](https://formilyjs.org/)._

#### reaction with field value

- In the simplest scenario, when the value of Form Field A changes, the value of Form Field B needs to be cleared.
- You can store the value of Form Field A in the Zustand store, subscribe to the corresponding dependencies using `useStore.subscribe()`, and actively call the form's `onValueChange` function to clear the value.

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

#### reaction with dropdown options

- In a more complex scenario where the value of Form Field A changes, and as a result, the value of Form Field B needs to be cleared, and dropdown options need to be updated:

- Building upon storing the value of Form Field A in the Zustand store, you can utilize the deps property of `useRequest` to subscribe to its updates and trigger a new request.

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

### Combining with third-party frameworks
<!-- 
<Callout
  type="success"
  content="By using the direct use of getState() and setState() in Zustand, you can seamlessly integrate React with other third-party frameworks, such as canvas."
/> -->

- Problem:

  - In a project scenario where the list is drawn based on a canvas, and click events occur within the canvas, but the state for rendering content comes from React, there is a need for a bridge to inform React to update the view.
  - The simplest and direct approach is to pass React's `setState` method to the render function of the Table's columns. However, in such scenarios, as they multiply, the code can become difficult to maintain.

- Solution:
  - Store the state related to data in **Zustand**.
  - After the canvas detects the onClick event, use `useStore.getState()`.actions to directly access the corresponding `setState()` method.
  - Changes in the data trigger React updates, thereby re-executing the column render function, ensuring that the canvas view is updated.

![canvas](/images/using-zustand-in-realworld-project/canvas.png)

## The source code

The function passed to the **Zustand** create API goes through several internal functions, ultimately handled by `createStoreImpl` for store creation:

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

### create store

- The logic for creating the store is encapsulated in the `createStoreImpl` function.
- First, the following methods are defined, corresponding to different responsibilities:
  - setState: Allows external modification of the state.
  - getState: Allows external retrieval of the state.
  - subscribe: Utilizes an event listening mechanism to trigger view updates when the state is updated.
  - destroy: Deletes all event listeners when the store is destroyed.

```tsx
const getState: () => TState = () => state
const subscribe: (listener: Listener) => () => void = (listener) => {
  listeners.add(listener)
  // Unsubscribe
  return () => listeners.delete(listener)
}
// ... ...
```

- Then, these methods are placed in an api object. The function provided in the create API is invoked with the parameters (setState, getState, api) to create the state.

```ts
const api = { setState, getState, subscribe, destroy }
state = (createState as PopArgument<typeof createState>)(setState, getState, api)
```

- Finally, the api object is passed through to the outermost layer to be captured and utilized by `createImpl`.

### set

The set function used in the callback receives `partial` and `replace` as parameters and primarily performs the following:

1. Checks if partial is a function. If it is, it is called with the current state, resulting in nextState. If not, partial is directly assigned to nextState.
2. Compares nextState with the current state. If they are not equal, it uses Object.assign to merge for object data types; otherwise, it is assigned directly.
3. If replace is true, nextState directly overrides the state.
4. Finally, notifies the view for an update.

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

The get function, obtained externally, is simple; it merely returns the function for accessing the state.

```ts
const getState: () => TState = () => state
```

### create state

Finally, combining the functions passed in the code with the source code, the actual form is as follows: Zustand takes the `createState` function and creates the state by passing in methods like set.

```ts
const createState = (set) => ({
  data: 0,

  action: {
    setData: (payload: number) => {
      set({ data: payload });
    }
  },
})

export const useStore = create<State>(createState);

const setState = // ...
const getState = // ...
const api = // ...
const state = createState(setState, getState, api)
```

### subscribe

After the previous steps, the state is created, but there is no connection with the React yet. The `createStoreImpl` function passes the entire api object to the createImpl function.

Inside, the `useBoundStore` function is generated, which is the hook obtained through the create API. It calls the internal useStore function and uses React's official useSyncExternalStoreWithSelector](https://reactjs.org/docs/hooks-reference.html#usesyncexternalstore) to let api.subscribe listen for events related to the selector, establishing a connection with the view layer to achieve updates.

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

## reference

- [https://tkdodo.eu/blog/working-with-zustand](https://tkdodo.eu/blog/working-with-zustand)
- [https://react-tracked.js.org/docs/tutorial-zustand-01/](https://react-tracked.js.org/docs/tutorial-zustand-01/)
