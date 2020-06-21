---
title: Strong Typed Redux
date: "2020-06-21T22:12:03.284Z"
description: "Strong typed wrapper for redux"
---

# Strong Typed Redux

## Introduction

There are several known approaches of adding a typed wrapper around the redux API.
In our current project we've developed an in-house approach that allows to create HOCs and hooks
at the same time. This allowed us to switch from class components to functional component
(or vice-versa) very easy, without having to rewrite the HOCs and hooks.

Besides trying to explain how this approach works. One aim of the article is to show
how to develop powerful typed interfaces around existing Javascript libraries, or at least
how we did it in this specific case.

[github.com/nicu-chiciuc/redux-with-connect](https://github.com/nicu-chiciuc/redux-with-connect).

### TL;DR

Below is an example of the pattern is use:

```typescript

/**
 * Create both the HOC and the hook at the same time
 */
export const [withConnectedTodos, useConnectedTodos] = bothConnect(
  ({ todos: { todos } }) => ({
    // map state to props
    mainTodos: todos.filter((todo) => todo.state !== "cancelled"),
    cancelledTodos: todos.filter((todo) => todo.state === "cancelled"),
  }),
  {
    // map dispatch to props
    markAsDone: markAsDoneAction,
    cancelTodo: cancelTodoAction,
    saveTodo: saveTodoAction,
  }
);

// Extract the inferred type without actually typing the props
export type WithConnectedTodos = ExtractConnect<typeof withConnectedTodos>;

// Inside another JSX file

/**
 * Props that will be received from the outside
 */
type OuterProps = { mainTitle: string };

/**
 * All the props that the component will have access to
 */
type Props = OuterProps & WithConnectedTodos;

class Todos extends React.Component<Props, State> {
...
}
// The return type will be `React.Component<OuterProps>`
export default withConnectedTodos(Todos);

// Or for a functional component

function TodosFn(props: OuterProps) {
  const [tempTodoTitle, setTodoTitle] = useState("");
  const todoProps = useConnectedTodos();
...
}

```

The `bothConnect` function works simliar to the default `connect` function from `react-redux`,
but it doesn't require explicit typing and creates both a HOC and a hook at the same time.
Also the `ExtractConnected` conditional type can extract the type of the props that will
be available:
![ExtractConnected](./extract-connect.png)

This allowed us to speed up some aspects of development by implicitly typing the HOCs
and by allowing to switch to a functional component if necessary.

## Intro to Redux

It is implied that you already know how redux works, if you don't,
this part should be like a very bad primer.
The main premise of Redux is that you have a global object
(not in the OOP sense, more like a record with data) usually called a Store
and you act as though it is immutable.
You can read values from it in you components and if you want to change it,
you pass a message (basically a JSON object) that contains a type field and some other data.
Based on that message you have some functions, called Reducers, that changes the Store.

There is much more to it and I recommend checking out [the official documentation](https://redux.js.org/).

### The sample project

The article is bassed on an internal article publishing inside the company I currently work for.
To make the code more palatable, I've created a
[very small project which uses Redux](https://github.com/nicu-chiciuc/redux-with-connect).
The logic of the project might seem a little ridiculous since it was not the aim of project.

# Simple types

Before going further imagine that in a different file you already defined the type of the Store and the types of the messages that can be passed to the store.

```typescript jsx
/// ./src/redux/index.ts
/**
 * The main type of the store
 */
export type StoreType = {
  todos: TodosStore
  checkboxes: CheckboxesStore
}

/**
 * A union type of all the possible messages
 */
export type MessageTypes = TodoMessages | CheckoxesMessages

const rootReducer = combineReducers<StoreType>({
  todos: todosReducer,
  checkboxes: checkboxReducer,
})

export const store = createStore(rootReducer)
```

Where `TodoMessages` and `CheckboxMessages` are [union types](https://www.typescriptlang.org/docs/handbook/advanced-types.html#union-types)

```typescript
/// ./src/redux/ReduxTodos.ts
export type TodoMessages =
  | {
      type: "TODOS/ADD_TODO"
      title: string
    }
  | {
      type: "TODOS/MARK_AS_DONE"
      id: string
    }
  | {
      type: "TODOS/CANCEL_TODO"
      id: string
    }
```

Most Redux tutorial would recommend defining action types as constant first.
That seems like a fear one would have in a world without static typing.

It looks kinda cryptic, but the basic idea is that a message can have a `type` from some set of types. And, for example if the `type` is `"TODOS/ADD_TODO"` the object should also contain a `title` property of type `string`. But this is more about business stuff, we're not concerned with that.

Let's get into the types.

Of course, the `connect` function from `react-redux` is _kinda_ typed, so you can tell it that you expect some stuff from it:

```typescript
const connectedTodos = connect(
  ({ todos: { todos } }: StoreType) => ({
    mainTodos: todos.filter(todo => todo.state !== "cancelled"),
    cancelledTodos: todos.filter(todo => todo.state === "cancelled"),
  }),
  {
    markAsDone: markAsDoneAction,
    cancelTodo: cancelTodoAction,
    saveTodo: saveTodoAction,
  }
)

export default connectedTodos(Todos)
```

As you can see we tell it that `store` has the type `StoreType` and now if we happen to change something in the StoreType, Typescript will scream until we fix everything we need to fix.

I guess this approach could work, but it seems too cumbersome, what if you forget to give `store` it's correct type. The `connect` function should know by default that `store` can only have one type. Unfortunately, the `react-redux` developers didn't know how our store will look and opening a PR each time we change something could be kinda annoying.

So here was the idea, what if, we create another function which will act exactly the same as `connect` but would always know that `store` has to have the type `StoreType` and that all the function in the second argument have to return an `MessageType`.

Below is the first commit to `bothConnect.ts`:

```typescript
import { ComponentType } from "react"
import { connect } from "react-redux"
import { StoreType } from "."

export type UseConnected<KnownProps> = <TProps>(
  Component: ComponentType<TProps & KnownProps>
) => ComponentType<TProps>

export type ExtractConnect<Something> = Something extends UseConnected<infer R>
  ? R
  : never

/**
 * This function works in the same way as the connect() from react-redux
 * but is more strict, knows the type of the store, and also allows to extract it's types very
 * easily using ExtractConnect
 */
export const useConnect = <PropsFromStore, PropsDispatch, TProps>(
  mapStateToProps: (store: StoreType) => PropsFromStore,
  mapDispatchToProps: PropsDispatch
): UseConnected<PropsFromStore & PropsDispatch> => <TProps>(
  Component: ComponentType<TProps & PropsFromStore & PropsDispatch>
): ComponentType<TProps> => {
  return connect(mapStateToProps, mapDispatchToProps)(Component as any)
}
```

There is some shady stuff going on, so let's unpack it.

First of all we'll have to dig into conditional types.

## Conditional Types

> [A conditional type selects one of two possible types based on a condition expressed as a type relationship test:](https://www.typescriptlang.org/docs/handbook/advanced-types.html#conditional-types)

```typescript
T extends U ? X : Y
```

I recommend trying [the Typescript Playground](https://www.typescriptlang.org/play/) to check how things work.

Conditional types are usually useful when you create typings for things that were initially written in JS and have a more complex interface. Or, when you want to write undecipherable code and then write blog posts about it.
![Conditional type example](./example01.png)

Basically we created the type `OtherType` conditionally based on whether we could assign `SomeType` to `red`. You might ask yourself, what does `extends` exactly mean in this context. In short, nobody really knows and you usually need to poke it until it makes sense. The way I finally started to think about it is that when writing `A extends B ?...` you're asking yourself (or the compiler) if you have some object of type `A` can you pass it to a function that accepts a type `B`? In practice though, you usually have to test things out and use the playground until things kinda work.

## [Inferring types](https://www.typescriptlang.org/docs/handbook/advanced-types.html#type-inference-in-conditional-types)

Imagine that you have a simple function:

```typescript
function simpleFunction(): boolean {
  return false
}
```

And somewhere else you have some variable (variable `something`), and you want to say that variable `something` should have the same type as the return type of the function `simpleFunction`.

Of course, in this case you can just say that:

```typescript
const something: boolean = true
```

But imagine that the function returned some other type and you don't really want to import another type and memorize it and whatnot. You just want to say that the variable `something` should always have the type as the return type of the `simpleFunction`.

```typescript
const something: ReturnType<typeof simpleFunction> = true
```

Somehow, `ReturnType` can extract a type from a different type

```typescript
type ReturnType<T> = T extends () => infer R ? R : never
```

This is (almost) the complete definition of `ReturnType`. Generally when thinking about generics in TS, think about them like functions that act on types instead of trying to fit in the OOP model (that usually helps with sleep too).

This means that, if you give me some type `T` and we somehow see that type `T` is some kind of function that return a `infer R`, then I return `R` otherwise return `never`. The actual definition of `ReturnType` in the document
![ReturnType](./example02.png)

`infer` only works with conditional types and the only (I think) use-case that I know of is to extract stuff from other types. These things only start to make sense after you try to use them and fail to do so for several months.

## `React.ComponentType<T>`

Another important thing is how to say that something is a react component.

Let's imagine that you wrote a component that besides the props that it expects from its parent, is also wrapped in a higher order component that passes it the prop `loading`. You can express that like so:

```typescript
function withLoading<TProps>(
  SomeComponent: React.ComponentType<TProps & { loading: boolean }>
): React.ComponentType<TProps> {
  return function WrappedComponent(mainProps: TProps) {
    return <SomeComponent {...mainProps} loading={false} />
  }
}

export default withLoading(ModalContainer)
```

There are other ways to do it, but this approach usually works the best.

> A lot of APIs use `Omit<T,K>`, `Exclud<T, U>` etc. But those approaches seem to cause more confusion down the road.

Basically the function `withLoading` can be passed some other component that has whatever props of type `TProps` and also MUST have a prop `loading` of type `boolean` and will return another component that shouldn't be passed the prop `loading` since we send it internally.

It looks kinda cryptic but that's life.

# `withConnect`

Below is the first iteration of `withConnect`

```typescript
import { ComponentType } from "react"
import { connect } from "react-redux"
import { StoreType } from "."

export type UseConnected<KnownProps> = <TProps>(
  Component: ComponentType<TProps & KnownProps>
) => ComponentType<TProps>

export type ExtractConnect<Something> = Something extends UseConnected<infer R>
  ? R
  : never

/**
 * This function works in the same way as the connect() from react-redux
 * but is more strict, knows the type of the store, and also allows to extract it's types very
 * easily using ExtractConnect
 */
export const useConnect: <PropsFromStore, PropsDispatch, TProps>(
  mapStateToProps: (store: StoreType) => PropsFromStore,
  mapDispatchToProps: PropsDispatch
) => UseConnected<PropsFromStore & PropsDispatch> = connect
```

Given some Component that has props of type `TProps` together with `PropsFromStore` and `PropsDispatch` it will return a component that has to be passed just `TProps`.

In this case, instead of `{loading: boolean}` we have some types that are also generic.

Since `connect` is a function that returns a function, things look ugly.

`mapStateToProps` is the function that is passed our `StoreType` and returns something from it. That something is not just `any`, it's a generic type that we're saving for later usage.

Same thing with `mapDispatchToProps`: it's an object and we save it's type.

This HOC (Higher-order component) could just replace the usage of `connect` since it doesn't interfere with its definition in any way.

`connect` is a function that gets a `mapStateToProps` and a `mapDispatchtoProps` and returns another function that should get a react component which should have some props that should include the `PropsFromStore` and `PropsDispatch` and it will return a component that has to be passed just the `TProps` without the props from redux.

This is a handful, but the definition is much much terse than the one in the official package:
![Connect definition](./connect_definition.png)

It goes on and on. If we decide to not use `defaultProps`, we can get away with defining very specific types that are complex, but not extremely complex.

So why the need to define a `UseConnected<T>`?

The idea was that instead of defining the types that should be given from the `connect` function, I would like to have a inferred type so that I wouldn't need to give the types explicitly. Check this out:
![Type inferrence](./type_inferrence.png)

Now we can define the `Props` without actually writing any types:

```typescript
type Props = { mainTitle: string } & ExtractConnect<typeof connectedTodos>
```

Basically what I wanted to do is just defined the very aptly-named `connectedTodos` and then let TS figure out the types and set them using `ExtractConnect<typeof connectedTodos>`.

If we don't use `UseConnected` TS tells us the whole type and it's kinda hard to figure out what's going on:

Also it's more clear to implement the ExtractConnected thingy:

```typescript
export type ExtractConnect<Something> = Something extends UseConnected<infer R>
  ? R
  : never
```

If we decide to change what exactly is `UseConnected` in the future, maybe it will be a more complex type of function, this thing will still work.

All in all, the goal was to trick TS into doing stuff for us, instead of us defining types all the time.

## Function overloading

Back to our problem. You probably noticed that we are force to ALWAYS pass both arguments to useConnected. That could be an ok down-side, but it's kinda annoying. It would be much easier if TS would take care of that for us because when using connect you have 3 obvious choices:

```typescript
connect(mapStateToProps)
connect(null, mapDispatchToProps)
connect(mapStateToProps, mapDispatchToProps)
```

I couldn't find a good solution for this so I just gave default arguments to the type parameters:

```typescript
import { ComponentType } from "react"
import { connect } from "react-redux"
import { StoreType } from "../store"

export type UseConnected<OtherProps> = <TProps>(
  Component: ComponentType<TProps & OtherProps>
) => ComponentType<TProps>

export type ExtractConnect<Something> = Something extends UseConnected<infer R>
  ? R
  : never

/**
 * This function works in the same way as the connect() from react-redux
 * but is more strict, knows the type of the store, and also allows to extract it's types very
 * easily using ExtractConnect
 */
export const withConnect = <PropsFromStore = {}, PropsDispatch = {}>(
  mapStateToProps: null | ((store: StoreType) => PropsFromStore),
  mapDispatchToProps?: PropsDispatch
) => {
  return connect(mapStateToProps, mapDispatchToProps) as UseConnected<
    PropsFromStore & PropsDispatch
  >
}
```

But after some time I realized that [function overloading](https://www.typescriptlang.org/docs/handbook/functions.html#overloads) can be used. I guess there could be other options (using unions of tuples for the arguments but I guess that didn't really click at the time).

```typescript
// prettier-ignore
export function withConnect<PropsDispatch extends ObjectWithMessages = {}>(mapStateToProps: null, mapDispatchToProps: PropsDispatch): WithConnected<PropsDispatch>;
// prettier-ignore
export function withConnect<PropsFromStore extends {} = {}>(mapStateToProps: (store: StoreType) => PropsFromStore, mapDispatchToProps?: undefined): WithConnected<PropsFromStore>;
// prettier-ignore
export function withConnect<PropsFromStore extends {} = {}, PropsDispatch extends ObjectWithMessages = {}>(mapStateToProps: (store: StoreType) => PropsFromStore, mapDispatchToProps: PropsDispatch): WithConnected<PropsFromStore & PropsDispatch>;

export function withConnect<
  PropsFromStore extends {} = {},
  PropsDispatch extends ObjectWithMessages = {}
>(
  mapStateToProps: null | ((store: StoreType) => PropsFromStore),
  mapDispatchToProps?: PropsDispatch
) {
  return connect(mapStateToProps, mapDispatchToProps)
}
```

The idea behind function overloads is that, when creating types for some functions, you can say that if the function gets a particular parameter it will return a different output, for example:
![Function overloading](./overloading.png)

It's not really an overload in the OOP sense, you still have just one function, but you can define multiple definitions for it and TS will handle the rest. The problem is that the function implementation has to support a union of all the possible arguments for all its overloads.

In the case of `withConnect` I was trying to say that you can call it in 3 different ways (as mentioned before) and it will act a little bit different in each case. If you didn't pass anything for the second argument, the function will return a simpler definition instead of doing something silly.

## The time of `hooks`

Around that time hooks appeared on the horizon and they came we new stuff. Instead of using `connect` they adviced to use `useActions` and `useSelector`. Defining wrappers for these was simpler since you didn't need overloads and all that crap.

```typescript
export function useConnect<T>(mapStateToProps: (store: StoreType) => T): T {
  return useSelector(mapStateToProps)
}

export function useActions<T extends {}>(actions: T): T {
  const dispatch = useDispatch()

  const boundDispatches = Object.entries<() => void>(actions).map(
    ([prop, value]) => {
      return [prop, bindActionCreators(value, dispatch)]
    }
  )

  return Object.fromEntries(boundDispatches)
}
```

Notice my very nice choice of reusing the same name for `useConnect` and making the matters even more confusing.

Of course `react-redux` removed `useActions` and advised the usage of `useDispatch`:

But we shouldn't really care what Abramov says.

I wanted to use the hook things just as I was already using the HOC things. Even `useConnect` and `useActions` seemed like too many functions. I wanted to define things just as I was defining them for HOCS.

So I created a function that could be passed the same arguments as `withConnect`:

```typescript
// prettier-ignore
export function createUseConnect<PropsDispatch extends ObjectWithMessages = {}>(mapStateToProps: null, mapDispatchToProps: PropsDispatch): UseConnected<PropsDispatch>;
// prettier-ignore
export function createUseConnect<PropsFromStore extends {} = {}>(mapStateToProps: (store: StoreType) => PropsFromStore, mapDispatchToProps?: undefined): UseConnected<PropsFromStore>;
// prettier-ignore
export function createUseConnect<PropsFromStore extends {} = {}, PropsDispatch extends ObjectWithMessages = {}>(mapStateToProps: (store: StoreType) => PropsFromStore, mapDispatchToProps: PropsDispatch): UseConnected<PropsFromStore & PropsDispatch>;

export function createUseConnect<
  PropsFromStore extends {} = {},
  PropsDispatch extends ObjectWithMessages = {}
>(
  mapStateToProps: null | ((store: StoreType) => PropsFromStore),
  mapDispatchToProps?: PropsDispatch
) {
  // This is done for performance reasons so that redux does not rerender the methods that don't care about some stuff

  if (mapStateToProps && mapDispatchToProps) {
    return () => ({
      ...useSelect(mapStateToProps),
      ...useActions(mapDispatchToProps),
    })
  }

  if (mapStateToProps && !mapDispatchToProps) {
    return () => useSelect(mapStateToProps)
  }

  if (mapDispatchToProps && !mapStateToProps) {
    return () => useActions(mapDispatchToProps)
  }

  return () => ({})
}
```

Just as you would do something like

```typescript
const withWhatever = withConnect(null, {whatever => {type: 'something}});
```

and then wrap a component with it, I wanted to create a hook in the same way:

```typescript
const useWhatever = createUseConnect(null, {whatever => {type: 'something}});
```

And then use `useWhatever` as a hook inside a function component and not think about it.

The definition is not perfect of course because we kinda have 3 options for the function but TS thinks that we should also react to the case when someone passes something else so we return a hook that returns an object `{}`.

# `bothConnect`

As you might observe, the HOC and the hooks are defined in almost the same way so I was thinking, why should I define the same thing twice, it would be cooler if we could just say what we want it to have from redux and then make it exist. So that was the idea behind `bothConnect`

```typescript
export type WithModalProps = ExtractConnect<typeof withModal>

export const [withModal, useModal] = bothConnect(null, {
  openModal,
  closeModal,
})
```

Here both the HOC `withModal` and the hook `useModal` are created at the same time. And the type `WithModalProps` is extracted automagically from one of them. This seems like the minimal amount of code necessary.

`bothConnect` was more annoying to implement since TS doesn't seem to like overloaded functions very much even though it seems clear that they should work. I could of course used `any` or `@ts-ignore` but real men don't do it like that.

```typescript
// prettier-ignore
export function bothConnect<PropsDispatch extends ObjectWithMessages = {}>(mapStateToProps: null, mapDispatchToProps: PropsDispatch): BothTypeConnect<PropsDispatch>;
// prettier-ignore
export function bothConnect<PropsFromStore extends {} = {}>(mapStateToProps: (store: StoreType) => PropsFromStore): BothTypeConnect<PropsFromStore>;
// prettier-ignore
export function bothConnect<PropsFromStore extends {} = {}, PropsDispatch extends ObjectWithMessages = {}>(mapStateToProps: (store: StoreType) => PropsFromStore, mapDispatchToProps: PropsDispatch): BothTypeConnect<PropsFromStore & PropsDispatch>;

export function bothConnect<
  PropsFromStore extends {} = {},
  PropsDispatch extends ObjectWithMessages = {}
>(
  mapStateToProps: null | ((store: StoreType) => PropsFromStore),
  mapDispatchToProps?: PropsDispatch
) {
  if (mapStateToProps && mapDispatchToProps) {
    return [
      withConnect(mapStateToProps, mapDispatchToProps),
      createUseConnect(mapStateToProps, mapDispatchToProps),
    ]
  }

  if (mapStateToProps && !mapDispatchToProps) {
    return [
      withConnect(mapStateToProps, mapDispatchToProps),
      createUseConnect(mapStateToProps, mapDispatchToProps),
    ]
  }

  if (mapDispatchToProps && !mapStateToProps) {
    return [
      withConnect(mapStateToProps, mapDispatchToProps),
      createUseConnect(mapStateToProps, mapDispatchToProps),
    ]
  }

  // INFO: Practically the inner functions already check for null/undefined and theoretically it would've been
  // necessary just 1 single line of code but Typescript (or me) cannot understand how to handle inner overloaded
  // functions and as such this hacky way was needed

  throw new Error("bothConnect passed both arguments that are null/undefined")
}
```

As you can see, TS is very nice and forces me to define the same code several times, but alas. That is the price for having a nice kitchen, sometimes the plumbing is full of shit.

I also used a tuple because the return values didn't really have any meaningful implicit names.

Some things could be tidied up, but it kinda worked and adding new stuff wasn't very hard.

# Final words

Not all TS is like this. It usually happens when an older JS API is given new types, things get messy real fast. But it seems to be easier to keep the plumbing in a single place instead of letting it run all over the kitchen.

In the next part I will touch on more advanced problems that have arisen after some usage:

- How to type `redux-thunk`
- How to memoize functions returned from `useActions` so that components don't reload all the time
