# Reselect
[![Travis][build-badge]][build]
[![npm package][npm-badge]][npm]
[![Coveralls][coveralls-badge]][coveralls]

他是一个为Redux(或者其他)而生的“数据选择器”的库，其灵感来自于[NuclearJS](https://github.com/optimizely/nuclear-js.git), [subscriptions](https://github.com/Day8/re-frame#just-a-read-only-cursor) in [re-frame](https://github.com/Day8/re-frame) and this [proposal](https://github.com/reduxjs/redux/pull/169) from [speedskater](https://github.com/speedskater)


* Selectors可以计算派生数据，可以让redux存储最小可执行状态树
* Selectors是高效的，只有当一个选择器的参数发生变化的时候，它才会重新计算
* Selectors是可组合的，任何一个单独的选择器都可以作为其他选择器的输入源
  
您可以在这里去做尝试与使用 [传送门](https://codepen.io/Domiii/pen/LzGNWj?editors=0010):

```js
import { createSelector } from 'reselect'

const shopItemsSelector = state => state.shop.items
const taxPercentSelector = state => state.shop.taxPercent

const subtotalSelector = createSelector(
  shopItemsSelector,
  items => items.reduce((acc, item) => acc + item.value, 0)
)

const taxSelector = createSelector(
  subtotalSelector,
  taxPercentSelector,
  (subtotal, taxPercent) => subtotal * (taxPercent / 100)
)

export const totalSelector = createSelector(
  subtotalSelector,
  taxSelector,
  (subtotal, tax) => ({ total: subtotal + tax })
)

let exampleState = {
  shop: {
    taxPercent: 8,
    items: [
      { name: 'apple', value: 1.20 },
      { name: 'orange', value: 0.95 },
    ]
  }
}

console.log(subtotalSelector(exampleState)) // 2.15
console.log(taxSelector(exampleState))      // 0.172
console.log(totalSelector(exampleState))    // { total: 2.322 }
```

## 内容列表

- [Reselect](#reselect)
  - [内容列表](#%e5%86%85%e5%ae%b9%e5%88%97%e8%a1%a8)
  - [安装](#installation)
  - [示例](#example)
    - [使用选择存储器的原因](#motivation-for-memoized-selectors)
      - [`containers/VisibleTodoList.js`](#containersvisibletodolistjs)
    - [创建选择存储器](#creating-a-memoized-selector)
      - [`selectors/index.js`](#selectorsindexjs)
    - [选择器的组成](#composing-selectors)
    - [Redux状态树和选择器的关联](#connecting-a-selector-to-the-redux-store)
      - [`containers/VisibleTodoList.js`](#containersvisibletodolistjs-1)
    - [React Props中使用选择器](#accessing-react-props-in-selectors)
      - [`components/App.js`](#componentsappjs)
      - [`selectors/todoSelectors.js`](#selectorstodoselectorsjs)
      - [`containers/VisibleTodoList.js`](#containersvisibletodolistjs-2)
    - [多个组建的实例通过Props共享选择器](#sharing-selectors-with-props-across-multiple-component-instances)
      - [`selectors/todoSelectors.js`](#selectorstodoselectorsjs-1)
      - [`containers/VisibleTodoList.js`](#containersvisibletodolistjs-3)
  - [API](#api)
    - [createSelector(...inputSelectors | [inputSelectors], resultFunc)](#createselectorinputselectors--inputselectors-resultfunc)
    - [defaultMemoize(func, equalityCheck = defaultEqualityCheck)](#defaultmemoizefunc-equalitycheck--defaultequalitycheck)
    - [createSelectorCreator(memoize, ...memoizeOptions)](#createselectorcreatormemoize-memoizeoptions)
      - [Customize `equalityCheck` for `defaultMemoize`](#customize-equalitycheck-for-defaultmemoize)
      - [Use memoize function from lodash for an unbounded cache](#use-memoize-function-from-lodash-for-an-unbounded-cache)
    - [createStructuredSelector({inputSelectors}, selectorCreator = createSelector)](#createstructuredselectorinputselectors-selectorcreator--createselector)
  - [FAQ](#faq)
    - [Q: 当输入状态发生变化时，选择器为何没有重新计算？](#q-why-isnt-my-selector-recomputing-when-the-input-state-changes)
    - [Q: 当输入状态保持不变，选择器为何重新计算？](#q-why-is-my-selector-recomputing-when-the-input-state-stays-the-same)
    - [Q: 怎样单独使用Reselect而不依赖redux？](#q-can-i-use-reselect-without-redux)
    - [Q: 怎样创建一个可以带参数的选择器？](#q-how-do-i-create-a-selector-that-takes-an-argument)
    - [Q: 如果默认的存储器方法不太适合我，我可以选择其他的方式吗？](#q-the-default-memoization-function-is-no-good-can-i-use-a-different-one)
    - [Q: 怎样测试选择器？](#q-how-do-i-test-a-selector)
    - [Q: Reselect怎样和Immutable.js配合使用？](#q-how-do-i-use-reselect-with-immutablejs)
    - [Q: 可以在多个组建实例中共享选择器吗？](#q-can-i-share-a-selector-across-multiple-component-instances)
    - [Q: 支持TypeScript语法吗？](#q-are-there-typescript-typings)
    - [Q: 怎样创建一个ramda选择器?](#q-how-can-i-make-a-curried-selector)
  - [项目相关](#related-projects)
    - [re-reselect](#re-reselect)
    - [reselect-tools](#reselect-tools)
    - [reselect-map](#reselect-map)
  - [License](#license)

- [项目相关](#related-projects)
- [License](#license)

## 安装
    npm install reselect

## 示例

如果你更喜欢视频教程，请点击[传送门](https://www.youtube.com/watch?v=6Xwo5mVxDqI).

### 使用选择存储器的原因
> 这个示例是通过[Redux Todos List example](http://redux.js.org/docs/basics/UsageWithReact.html)实现。

#### `containers/VisibleTodoList.js`

```js
import { connect } from 'react-redux'
import { toggleTodo } from '../actions'
import TodoList from '../components/TodoList'

const getVisibleTodos = (todos, filter) => {
  switch (filter) {
    case 'SHOW_ALL':
      return todos
    case 'SHOW_COMPLETED':
      return todos.filter(t => t.completed)
    case 'SHOW_ACTIVE':
      return todos.filter(t => !t.completed)
  }
}

const mapStateToProps = (state) => {
  return {
    todos: getVisibleTodos(state.todos, state.visibilityFilter)
  }
}

const mapDispatchToProps = (dispatch) => {
  return {
    onTodoClick: (id) => {
      dispatch(toggleTodo(id))
    }
  }
}

const VisibleTodoList = connect(
  mapStateToProps,
  mapDispatchToProps
)(TodoList)

export default VisibleTodoList
```

以上示例中, `mapStateToProps` 调用 `getVisibleTodos` 来计算 `todos`. 这样看起来不错，但是它有一个缺点: 当状态树发生变化时`todos` 每次都会被重新计算. 如果状态树非常大, 或者每次计算性能开销都非常大,每次的重复计算可能会导致性能问题。Reselect可以帮助你解决不必要的计算。

### 创建选择存储器

当`state.todos` 或 `state.visibilityFilter`数据发生变化时，而不是状态树的其他部分发生变化的时候，我们希望用一个带存储器功能的选择器代替`getVisibleTodos`去重新计算`todos`，

Reselect提供一个方法`createSelector`去创建存储选择器。 `createSelector`接收一组输入选择器并且将这些函数转化为参数。如果Redux状态树发生变化导致选择器的输入源数据发生改变，选择器将会调用转换函数将输入的参数所执行后的结果返回。如果这个结果和之前的选择器的结果一致，他将返回上一个计算结果，而不是重复调用这个转换函数。

我们定义一个存储选择器，并且命名为`getVisibleTodos`，用它来代替不带缓存功能的版本:

#### `selectors/index.js`

```js
import { createSelector } from 'reselect'

const getVisibilityFilter = (state) => state.visibilityFilter
const getTodos = (state) => state.todos

export const getVisibleTodos = createSelector(
  [ getVisibilityFilter, getTodos ],
  (visibilityFilter, todos) => {
    switch (visibilityFilter) {
      case 'SHOW_ALL':
        return todos
      case 'SHOW_COMPLETED':
        return todos.filter(t => t.completed)
      case 'SHOW_ACTIVE':
        return todos.filter(t => !t.completed)
    }
  }
)
```
在上面的例子中，`getVisibilityFilter` 和 `getTodos` 是选择器的输入源。他们是不带缓存功能的普通的选择器，因为他们没有转换所选择的数据。`getVisibleTodos`在另一边是一个缓存的选择器。它使用`getVisibilityFilter` 和 `getTodos` 这两个数据输入源，再加上转换函数计算出需要的 todos list。

### 选择器的组成

一个存储选择器本身可以当成另一个存储选择器的输入源。`getVisibleTodos`可以作为一个数据输入源给另一个选择器使用通过关键字做进一步的筛选：

```js
const getKeyword = (state) => state.keyword

const getVisibleTodosFilteredByKeyword = createSelector(
  [ getVisibleTodos, getKeyword ],
  (visibleTodos, keyword) => visibleTodos.filter(
    todo => todo.text.includes(keyword)
  )
)
```

### Redux状态树和选择器的关联

如果你在使用 [React Redux](https://github.com/reduxjs/react-redux), 你可以在`mapStateToProps()`里调用选择器的方法:

#### `containers/VisibleTodoList.js`

```js
import { connect } from 'react-redux'
import { toggleTodo } from '../actions'
import TodoList from '../components/TodoList'
import { getVisibleTodos } from '../selectors'

const mapStateToProps = (state) => {
  return {
    todos: getVisibleTodos(state)
  }
}

const mapDispatchToProps = (dispatch) => {
  return {
    onTodoClick: (id) => {
      dispatch(toggleTodo(id))
    }
  }
}

const VisibleTodoList = connect(
  mapStateToProps,
  mapDispatchToProps
)(TodoList)

export default VisibleTodoList
```

### React Props中使用选择器

> 本节内容介绍了假设我们允许扩展多个Todo Lists的方法。请注意此程序的完整实施过程需要改变reducers，components，actions等等，为简洁期间，我们暂时省略掉与本次内容不相关的内容。

这样我们看到选择器将Redux状态树作为参数接收，并且选择器也可以作为props传递。

这是一个渲染了三次`VisibleTodoList`的`App`组件的实例，每个实例都有一个`listId`的prop：

#### `components/App.js`

```js
import React from 'react'
import Footer from './Footer'
import AddTodo from '../containers/AddTodo'
import VisibleTodoList from '../containers/VisibleTodoList'

const App = () => (
  <div>
    <VisibleTodoList listId="1" />
    <VisibleTodoList listId="2" />
    <VisibleTodoList listId="3" />
  </div>
)
```
每个`VisibleTodoList`容器必须依赖`listId` prop来选择不同的状态切片，所以我们通过修改`getVisibilityFilter` 和 `getTodos`来接收props参数：

#### `selectors/todoSelectors.js`

```js
import { createSelector } from 'reselect'

const getVisibilityFilter = (state, props) =>
  state.todoLists[props.listId].visibilityFilter

const getTodos = (state, props) =>
  state.todoLists[props.listId].todos

const getVisibleTodos = createSelector(
  [ getVisibilityFilter, getTodos ],
  (visibilityFilter, todos) => {
    switch (visibilityFilter) {
      case 'SHOW_COMPLETED':
        return todos.filter(todo => todo.completed)
      case 'SHOW_ACTIVE':
        return todos.filter(todo => !todo.completed)
      default:
        return todos
    }
  }
)

export default getVisibleTodos
```

`props` can be passed to `getVisibleTodos` from `mapStateToProps`:

```js
const mapStateToProps = (state, props) => {
  return {
    todos: getVisibleTodos(state, props)
  }
}
```
所以，现在`getVisibleTodos`可以访问`props'，并且一切似乎都工作正常。

**但是这样有一个问题!**

在使用`getVisibleTodos`选择器和`VisibleTodoList`容器多次实例化的时候，这样并不会正确的使用缓存：

#### `containers/VisibleTodoList.js`

```js
import { connect } from 'react-redux'
import { toggleTodo } from '../actions'
import TodoList from '../components/TodoList'
import { getVisibleTodos } from '../selectors'

const mapStateToProps = (state, props) => {
  return {
    // WARNING: THE FOLLOWING SELECTOR DOES NOT CORRECTLY MEMOIZE
    todos: getVisibleTodos(state, props)
  }
}

const mapDispatchToProps = (dispatch) => {
  return {
    onTodoClick: (id) => {
      dispatch(toggleTodo(id))
    }
  }
}

const VisibleTodoList = connect(
  mapStateToProps,
  mapDispatchToProps
)(TodoList)

export default VisibleTodoList
```

一个选择器在用`createSelector`创建的时候，当它设置的参数和上一次设置的参数相同的时候，这个选择器会设置他的缓存大小为1并且只返回它的缓存。如果我们交替渲染`<VisibleTodoList listId="1" />` 和 `<VisibleTodoList listId="2" />`，共享的选择器会交替接收`{listId: 1}` 和 `{listId: 2}`作为他们的`props`参数。这样会导致每次调用的参数都不相同，因此选择器每次都会重新计算，而不是用缓存的数据。我们将在下一节中了解如何克服此限制。

### 跨多个组件实例共享选择器

> 这个章节的示例需要React Redux v4.3.0 或更高版本

> 另一种方法示例在[re-reselect](https://github.com/toomuchdesign/re-reselect)

To share a selector across multiple `VisibleTodoList` instances while passing in `props` **and** retaining memoization, each instance of the component needs its own private copy of the selector.

Let’s create a function named `makeGetVisibleTodos` that returns a new copy of the `getVisibleTodos` selector each time it is called:

#### `selectors/todoSelectors.js`

```js
import { createSelector } from 'reselect'

const getVisibilityFilter = (state, props) =>
  state.todoLists[props.listId].visibilityFilter

const getTodos = (state, props) =>
  state.todoLists[props.listId].todos

const makeGetVisibleTodos = () => {
  return createSelector(
    [ getVisibilityFilter, getTodos ],
    (visibilityFilter, todos) => {
      switch (visibilityFilter) {
        case 'SHOW_COMPLETED':
          return todos.filter(todo => todo.completed)
        case 'SHOW_ACTIVE':
          return todos.filter(todo => !todo.completed)
        default:
          return todos
      }
    }
  )
}

export default makeGetVisibleTodos
```

We also need a way to give each instance of a container access to its own private selector. The `mapStateToProps` argument of `connect` can help with this.

**If the `mapStateToProps` argument supplied to `connect` returns a function instead of an object, it will be used to create an individual `mapStateToProps` function for each instance of the container.**

In the example below `makeMapStateToProps` creates a new `getVisibleTodos` selector, and returns a `mapStateToProps` function that has exclusive access to the new selector:

```js
const makeMapStateToProps = () => {
  const getVisibleTodos = makeGetVisibleTodos()
  const mapStateToProps = (state, props) => {
    return {
      todos: getVisibleTodos(state, props)
    }
  }
  return mapStateToProps
}
```

If we pass `makeMapStateToProps` to `connect`, each instance of the `VisibleTodoList` container will get its own `mapStateToProps` function with a private `getVisibleTodos` selector. Memoization will now work correctly regardless of the render order of the `VisibleTodoList` containers.

#### `containers/VisibleTodoList.js`

```js
import { connect } from 'react-redux'
import { toggleTodo } from '../actions'
import TodoList from '../components/TodoList'
import { makeGetVisibleTodos } from '../selectors'

const makeMapStateToProps = () => {
  const getVisibleTodos = makeGetVisibleTodos()
  const mapStateToProps = (state, props) => {
    return {
      todos: getVisibleTodos(state, props)
    }
  }
  return mapStateToProps
}

const mapDispatchToProps = (dispatch) => {
  return {
    onTodoClick: (id) => {
      dispatch(toggleTodo(id))
    }
  }
}

const VisibleTodoList = connect(
  makeMapStateToProps,
  mapDispatchToProps
)(TodoList)

export default VisibleTodoList
```

## API

### createSelector(...inputSelectors | [inputSelectors], resultFunc)

Takes one or more selectors, or an array of selectors, computes their values and passes them as arguments to `resultFunc`.

`createSelector` determines if the value returned by an input-selector has changed between calls using reference equality (`===`). Inputs to selectors created with `createSelector` should be immutable.

Selectors created with `createSelector` have a cache size of 1. This means they always recalculate when the value of an input-selector changes, as a selector only stores the preceding value of each input-selector.

```js
const mySelector = createSelector(
  state => state.values.value1,
  state => state.values.value2,
  (value1, value2) => value1 + value2
)

// You can also pass an array of selectors
const totalSelector = createSelector(
  [
    state => state.values.value1,
    state => state.values.value2
  ],
  (value1, value2) => value1 + value2
)
```

It can be useful to access the props of a component from within a selector. When a selector is connected to a component with `connect`, the component props are passed as the second argument to the selector:

```js
const abSelector = (state, props) => state.a * props.b

// props only (ignoring state argument)
const cSelector =  (_, props) => props.c

// state only (props argument omitted as not required)
const dSelector = state => state.d

const totalSelector = createSelector(
  abSelector,
  cSelector,
  dSelector,
  (ab, c, d) => ({
    total: ab + c + d
  })
)

```

### defaultMemoize(func, equalityCheck = defaultEqualityCheck)

`defaultMemoize` memoizes the function passed in the func parameter. It is the memoize function used by `createSelector`.

`defaultMemoize` has a cache size of 1. This means it always recalculates when the value of an argument changes.

`defaultMemoize` determines if an argument has changed by calling the `equalityCheck` function. As `defaultMemoize` is designed to be used with immutable data, the default `equalityCheck` function checks for changes using reference equality:

```js
function defaultEqualityCheck(currentVal, previousVal) {
  return currentVal === previousVal
}
```

`defaultMemoize` can be used with `createSelectorCreator` to [customize the `equalityCheck` function](#customize-equalitycheck-for-defaultmemoize).

### createSelectorCreator(memoize, ...memoizeOptions)

`createSelectorCreator` can be used to make a customized version of `createSelector`.

The `memoize` argument is a memoization function to replace `defaultMemoize`.

The `...memoizeOptions` rest parameters are zero or more configuration options to be passed to `memoizeFunc`. The selectors `resultFunc` is passed as the first argument to `memoize` and the `memoizeOptions` are passed as the second argument onwards:

```js
const customSelectorCreator = createSelectorCreator(
  customMemoize, // function to be used to memoize resultFunc
  option1, // option1 will be passed as second argument to customMemoize
  option2, // option2 will be passed as third argument to customMemoize
  option3 // option3 will be passed as fourth argument to customMemoize
)

const customSelector = customSelectorCreator(
  input1,
  input2,
  resultFunc // resultFunc will be passed as first argument to customMemoize
)
```

Internally `customSelector` calls the memoize function as follows:

```js
customMemoize(resultFunc, option1, option2, option3)
```

Here are some examples of how you might use `createSelectorCreator`:

#### Customize `equalityCheck` for `defaultMemoize`

```js
import { createSelectorCreator, defaultMemoize } from 'reselect'
import isEqual from 'lodash.isEqual'

// create a "selector creator" that uses lodash.isEqual instead of ===
const createDeepEqualSelector = createSelectorCreator(
  defaultMemoize,
  isEqual
)

// use the new "selector creator" to create a selector
const mySelector = createDeepEqualSelector(
  state => state.values.filter(val => val < 5),
  values => values.reduce((acc, val) => acc + val, 0)
)
```

#### Use memoize function from lodash for an unbounded cache

```js
import { createSelectorCreator } from 'reselect'
import memoize from 'lodash.memoize'

let called = 0
const hashFn = (...args) => args.reduce(
  (acc, val) => acc + '-' + JSON.stringify(val),
  ''
)
const customSelectorCreator = createSelectorCreator(memoize, hashFn)
const selector = customSelectorCreator(
  state => state.a,
  state => state.b,
  (a, b) => {
    called++
    return a + b
  }
)
```

### createStructuredSelector({inputSelectors}, selectorCreator = createSelector)

`createStructuredSelector` is a convenience function for a common pattern that arises when using Reselect. The selector passed to a `connect` decorator often just takes the values of its input-selectors and maps them to keys in an object:

```js
const mySelectorA = state => state.a
const mySelectorB = state => state.b

// The result function in the following selector
// is simply building an object from the input selectors
const structuredSelector = createSelector(
   mySelectorA,
   mySelectorB,
   (a, b) => ({
     a,
     b
   })
)
```

`createStructuredSelector` takes an object whose properties are input-selectors and returns a structured selector. The structured selector returns an object with the same keys as the `inputSelectors` argument, but with the selectors replaced with their values.

```js
const mySelectorA = state => state.a
const mySelectorB = state => state.b

const structuredSelector = createStructuredSelector({
  x: mySelectorA,
  y: mySelectorB
})

const result = structuredSelector({ a: 1, b: 2 }) // will produce { x: 1, y: 2 }
```

Structured selectors can be nested:

```js
const nestedSelector = createStructuredSelector({
  subA: createStructuredSelector({
    selectorA,
    selectorB
  }),
  subB: createStructuredSelector({
    selectorC,
    selectorD
  })
})

```

## FAQ

### Q: Why isn’t my selector recomputing when the input state changes?

A: Check that your memoization function is compatible with your state update function (i.e. the reducer if you are using Redux). For example, a selector created with `createSelector` will not work with a state update function that mutates an existing object instead of creating a new one each time. `createSelector` uses an identity check (`===`) to detect that an input has changed, so mutating an existing object will not trigger the selector to recompute because mutating an object does not change its identity. Note that if you are using Redux, mutating the state object is [almost certainly a mistake](http://redux.js.org/docs/Troubleshooting.html).

The following example defines a simple selector that determines if the first todo item in an array of todos has been completed:

```js
const isFirstTodoCompleteSelector = createSelector(
  state => state.todos[0],
  todo => todo && todo.completed
)
```

The following state update function **will not** work with `isFirstTodoCompleteSelector`:

```js
export default function todos(state = initialState, action) {
  switch (action.type) {
  case COMPLETE_ALL:
    const areAllMarked = state.every(todo => todo.completed)
    // BAD: mutating an existing object
    return state.map(todo => {
      todo.completed = !areAllMarked
      return todo
    })

  default:
    return state
  }
}
```

The following state update function **will** work with `isFirstTodoCompleteSelector`:

```js
export default function todos(state = initialState, action) {
  switch (action.type) {
  case COMPLETE_ALL:
    const areAllMarked = state.every(todo => todo.completed)
    // GOOD: returning a new object each time with Object.assign
    return state.map(todo => Object.assign({}, todo, {
      completed: !areAllMarked
    }))

  default:
    return state
  }
}
```

If you are not using Redux and have a requirement to work with mutable data, you can use `createSelectorCreator` to replace the default memoization function and/or use a different equality check function. See [here](#use-memoize-function-from-lodash-for-an-unbounded-cache) and [here](#customize-equalitycheck-for-defaultmemoize) for examples.

### Q: Why is my selector recomputing when the input state stays the same?

A: Check that your memoization function is compatible with your state update function (i.e. the reducer if you are using Redux). For example, a selector created with `createSelector` that recomputes unexpectedly may be receiving a new object on each update whether the values it contains have changed or not. `createSelector` uses an identity check (`===`) to detect that an input has changed, so returning a new object on each update means that the selector will recompute on each update.

```js
import { REMOVE_OLD } from '../constants/ActionTypes'

const initialState = [
  {
    text: 'Use Redux',
    completed: false,
    id: 0,
    timestamp: Date.now()
  }
]

export default function todos(state = initialState, action) {
  switch (action.type) {
  case REMOVE_OLD:
    return state.filter(todo => {
      return todo.timestamp + 30 * 24 * 60 * 60 * 1000 > Date.now()
    })
  default:
    return state
  }
}
```

The following selector is going to recompute every time REMOVE_OLD is invoked because Array.filter always returns a new object. However, in the majority of cases the REMOVE_OLD action will not change the list of todos so the recomputation is unnecessary.

```js
import { createSelector } from 'reselect'

const todosSelector = state => state.todos

export const visibleTodosSelector = createSelector(
  todosSelector,
  (todos) => {
    ...
  }
)
```

You can eliminate unnecessary recomputations by returning a new object from the state update function only when a deep equality check has found that the list of todos has actually changed:

```js
import { REMOVE_OLD } from '../constants/ActionTypes'
import isEqual from 'lodash.isEqual'

const initialState = [
  {
    text: 'Use Redux',
    completed: false,
    id: 0,
    timestamp: Date.now()
  }
]

export default function todos(state = initialState, action) {
  switch (action.type) {
  case REMOVE_OLD:
    const updatedState =  state.filter(todo => {
      return todo.timestamp + 30 * 24 * 60 * 60 * 1000 > Date.now()
    })
    return isEqual(updatedState, state) ? state : updatedState
  default:
    return state
  }
}
```

Alternatively, the default `equalityCheck` function in the selector can be replaced by a deep equality check:

```js
import { createSelectorCreator, defaultMemoize } from 'reselect'
import isEqual from 'lodash.isEqual'

const todosSelector = state => state.todos

// create a "selector creator" that uses lodash.isEqual instead of ===
const createDeepEqualSelector = createSelectorCreator(
  defaultMemoize,
  isEqual
)

// use the new "selector creator" to create a selector
const mySelector = createDeepEqualSelector(
  todosSelector,
  (todos) => {
    ...
  }
)
```

Always check that the cost of an alternative `equalityCheck` function or deep equality check in the state update function is not greater than the cost of recomputing every time. If recomputing every time does work out to be the cheaper option, it may be that for this case Reselect is not giving you any benefit over passing a plain `mapStateToProps` function to `connect`.

### Q: Can I use Reselect without Redux?

A: Yes. Reselect has no dependencies on any other package, so although it was designed to be used with Redux it can be used independently. It is currently being used successfully in traditional Flux apps.

> If you create selectors using `createSelector` make sure its arguments are immutable.
> See [here](#createselectorinputselectors--inputselectors-resultfunc)

### Q: How do I create a selector that takes an argument?

A: Keep in mind that selectors can access React props, so if your arguments are (or can be made available as) React props, you can use that functionality. [See here](#accessing-react-props-in-selectors) for details.

Otherwise, Reselect doesn't have built-in support for creating selectors that accepts arguments, but here are some suggestions for implementing similar functionality...

If the argument is not dynamic you can use a factory function:

```js
const expensiveItemSelectorFactory = minValue => {
  return createSelector(
    shopItemsSelector,
    items => items.filter(item => item.value > minValue)
  )
}

const subtotalSelector = createSelector(
  expensiveItemSelectorFactory(200),
  items => items.reduce((acc, item) => acc + item.value, 0)
)
```

The general consensus [here](https://github.com/reduxjs/reselect/issues/38) and [over at nuclear-js](https://github.com/optimizely/nuclear-js/issues/14) is that if a selector needs a dynamic argument, then that argument should probably be state in the store. If you decide that you do require a selector with a dynamic argument, then a selector that returns a memoized function may be suitable:

```js
import { createSelector } from 'reselect'
import memoize from 'lodash.memoize'

const expensiveSelector = createSelector(
  state => state.items,
  items => memoize(
    minValue => items.filter(item => item.value > minValue)
  )
)

const expensiveFilter = expensiveSelector(state)

const slightlyExpensive = expensiveFilter(100)
const veryExpensive = expensiveFilter(1000000)
```

### Q: The default memoization function is no good, can I use a different one?

A: We think it works great for a lot of use cases, but sure. See [these examples](#customize-equalitycheck-for-defaultmemoize).

### Q: How do I test a selector?

A: For a given input, a selector should always produce the same output. For this reason they are simple to unit test.

```js
const selector = createSelector(
  state => state.a,
  state => state.b,
  (a, b) => ({
    c: a * 2,
    d: b * 3
  })
)

test("selector unit test", () => {
  assert.deepEqual(selector({ a: 1, b: 2 }), { c: 2, d: 6 })
  assert.deepEqual(selector({ a: 2, b: 3 }), { c: 4, d: 9 })
})
```

It may also be useful to check that the memoization function for a selector works correctly with the state update function (i.e. the reducer if you are using Redux). Each selector has a `recomputations` method that will return the number of times it has been recomputed:

```js
suite('selector', () => {
  let state = { a: 1, b: 2 }

  const reducer = (state, action) => (
    {
      a: action(state.a),
      b: action(state.b)
    }
  )

  const selector = createSelector(
    state => state.a,
    state => state.b,
    (a, b) => ({
      c: a * 2,
      d: b * 3
    })
  )

  const plusOne = x => x + 1
  const id = x => x

  test("selector unit test", () => {
    state = reducer(state, plusOne)
    assert.deepEqual(selector(state), { c: 4, d: 9 })
    state = reducer(state, id)
    assert.deepEqual(selector(state), { c: 4, d: 9 })
    assert.equal(selector.recomputations(), 1)
    state = reducer(state, plusOne)
    assert.deepEqual(selector(state), { c: 6, d: 12 })
    assert.equal(selector.recomputations(), 2)
  })
})
```

Additionally, selectors keep a reference to the last result function as `.resultFunc`. If you have selectors composed of many other selectors this can help you test each selector without coupling all of your tests to the shape of your state.

For example if you have a set of selectors like this:

**selectors.js**
```js
export const firstSelector = createSelector( ... )
export const secondSelector = createSelector( ... )
export const thirdSelector = createSelector( ... )

export const myComposedSelector = createSelector(
  firstSelector,
  secondSelector,
  thirdSelector,
  (first, second, third) => first * second < third
)
```

And then a set of unit tests like this:

**test/selectors.js**

```js
// tests for the first three selectors...
test("firstSelector unit test", () => { ... })
test("secondSelector unit test", () => { ... })
test("thirdSelector unit test", () => { ... })

// We have already tested the previous
// three selector outputs so we can just call `.resultFunc`
// with the values we want to test directly:
test("myComposedSelector unit test", () => {
  // here instead of calling selector()
  // we just call selector.resultFunc()
  assert(myComposedSelector.resultFunc(1, 2, 3), true)
  assert(myComposedSelector.resultFunc(2, 2, 1), false)
})
```

Finally, each selector has a `resetRecomputations` method that sets
recomputations back to 0.  The intended use is for a complex selector that may
have many independent tests and you don't want to manually manage the
computation count or create a "dummy" selector for each test.

### Q: How do I use Reselect with Immutable.js?

A: Selectors created with `createSelector` should work just fine with Immutable.js data structures.

If your selector is recomputing and you don't think the state has changed, make sure you are aware of which Immutable.js update methods **always** return a new object and which update methods only return a new object **when the collection actually changes**.

```js
import Immutable from 'immutable'

let myMap = Immutable.Map({
  a: 1,
  b: 2,
  c: 3
})

 // set, merge and others only return a new obj when update changes collection
let newMap = myMap.set('a', 1)
assert.equal(myMap, newMap)
newMap = myMap.merge({ 'a': 1 })
assert.equal(myMap, newMap)
// map, reduce, filter and others always return a new obj
newMap = myMap.map(a => a * 1)
assert.notEqual(myMap, newMap)
```

If a selector's input is updated by an operation that always returns a new object, it may be performing unnecessary recomputations. See [here](#q-why-is-my-selector-recomputing-when-the-input-state-stays-the-same) for a discussion on the pros and cons of using a deep equality check like `Immutable.is` to eliminate unnecessary recomputations.

### Q: Can I share a selector across multiple component instances?

A: Selectors created using `createSelector` only have a cache size of one. This can make them unsuitable for sharing across multiple instances if the arguments to the selector are different for each instance of the component. There are a couple of ways to get around this:

* Create a factory function which returns a new selector for each instance of the component. There is built-in support for factory functions in React Redux v4.3 or higher. See [here](#sharing-selectors-with-props-across-multiple-component-instances) for an example.

* Create a custom selector with a cache size greater than one.

### Q: Are there TypeScript Typings?

A: Yes! They are included and referenced in `package.json`. They should Just Work™.

### Q: How can I make a [curried](https://github.com/hemanth/functional-programming-jargon#currying) selector?

A: Try these [helper functions](https://github.com/reduxjs/reselect/issues/159#issuecomment-238724788) courtesy of [MattSPalmer](https://github.com/MattSPalmer)

## Related Projects

### [re-reselect](https://github.com/toomuchdesign/re-reselect)

Enhances Reselect selectors by wrapping `createSelector` and returning a memoized collection of selectors indexed with the cache key returned by a custom resolver function.

Useful to reduce selectors recalculation when the same selector is repeatedly called with one/few different arguments.

### [reselect-tools](https://github.com/skortchmark9/reselect-tools)

[Chrome extension](https://chrome.google.com/webstore/detail/reselect-devtools/cjmaipngmabglflfeepmdiffcijhjlbb?hl=en) and [companion lib](https://github.com/skortchmark9/reselect-tools) for debugging selectors.

* Measure selector recomputations across the app and identify performance bottlenecks
* Check selector dependencies, inputs, outputs, and recomputations at any time with the chrome extension
* Statically export a JSON representation of your selector graph for further analysis

### [reselect-map](https://github.com/HeyImAlex/reselect-map)

Can be useful when doing **very expensive** computations on elements of a collection because Reselect might not give you the granularity of caching that you need. Check out the reselect-maps README for examples.

**The optimizations in reselect-map only apply in a small number of cases. If you are unsure whether you need it, you don't!**

## License

MIT

[build-badge]: https://img.shields.io/travis/reduxjs/reselect/master.svg?style=flat-square
[build]: https://travis-ci.org/reduxjs/reselect

[npm-badge]: https://img.shields.io/npm/v/reselect.svg?style=flat-square
[npm]: https://www.npmjs.org/package/reselect

[coveralls-badge]: https://img.shields.io/coveralls/reduxjs/reselect/master.svg?style=flat-square
[coveralls]: https://coveralls.io/github/reduxjs/reselect
