+++
title = "Making Sense of React Hooks"
date = "2025-12-14"
tags = [
    "react",
    "hooks",
    "vue"
]
+++

Recently, the company I work for switched their frontend web framework from Vue.js to React.js. I think it’s fair to say that React has a harder learning curve 📈 than Vue.js. Vue is also arguably smarter when it comes to change detection and re-rendering as you don’t have to worry about passing in dependencies to computed/watched/side effects. With React ⚛️ on the other hand, it’s a lot easier to shoot yourself in the foot if you overlook the nuances of the framework. One of the distinct features in React.js are hooks.

There is nothing quite like React hooks 🪝 when it comes to how they compose logic although parallels can be drawn with Angular Signals and the Vue Composition API. In this article, I’ll be going over what React hooks are, the 10 most commonly used React hooks and how to make your own React hooks.

## What are hooks?

![Spring Beans meme](/images/making-sense-react-hooks/meme.jpg)

Is it a thing used to catch people’s attention? The chorus of a song? Or is it maybe a metal instrument that can potentially be strapped to an arm to catch fairies with? The answer is yes to all of those but in the context of React, the definition is such: a special function that lets you use React features like state, side effects and context without writing a class component. They “hook” into React’s internal lifecycle system so you can manage stateful logic inside simple functions.

Before Hooks, React components were primarily written as class components if you needed state or lifecycle methods which came with a lot of issues. Class components in general require a lot of boilerplate code such as a constructor, _this.state_, _this.setState_, binding methods to _this_, etc. Sharing stateful logic was also more complex as it required higher-order components (HOCs) which made code harder to read and maintain.

Pair this with confusing lifecycle methods like _componentDidMount_, _componentDidUpdate_, _componentWillUnmount_, etc and React devs were stuck in a perpetual state of “Wrapper Hell”. Kind of like “Callback Hell” 🌋 but with class components. Javascript programmers are already pretty scared of classes and OOP so this was just a nail in the coffin. Some of us still deny that classes were added in ES6 and pretend they don’t exist at all. But that’s a discussion for another day.

So in brief, React Hooks were introduced to simplify component logic while keeping everything functional and modular.

## The Rules of Hooks

React Hooks come with a small but strict set of rules which should always be adhered to. These rules aren’t arbitrary, they exist because hooks rely on the order in which they are called to correctly work with state and effects.

### Rule 1: Only call hooks at the top level

Hooks must not be called inside loops, conditions or nested functions. React expects hooks to be called in the same order on every render. If that order changes, React cannot reliably associate hook calls with their state. Here’s a bad and good example:

**Bad:**

```js
if (isLoggedIn) {
  useEffect(() => {}, []);
}
```

**Good:**

```js
useEffect(() => {
  if (isLoggedIn) {
    // do something
  }
}, [isLoggedIn]);
```

### Rule 2: Only call hooks from React functions

Hooks can only be used inside React function components or custom hooks (which we will dabble in later). You cannot use hooks in regular JavaScript functions, class components or event handlers defined outside a React component.

### Why these rules exist

Hooks work by position, not by name. Behind the scenes, React maintains an internal list associated with each function component and iterates through this list using a pointer when the component renders. This is very different from Vue’s Composition API, which tracks dependencies automatically. React trades some safety for explicitness and predictability, which is why plugins like _eslint-plugin-react-hooks_ are practically mandatory in real-world projects.

### The 5 Most Commonly Used Hooks

React.js ships with many hooks but most applications will only require the use of a core subset.

### 1. useState

This is the most basic hook. It simply allows function components to hold local state.

```js
const [count, setCount] = useState(0);
```

useState is a simple hook but easy to misuse. Overusing it for derived state or deeply nested objects can quickly lead to messy components.

### 2. useEffect

useEffect is used for side effects such as data fetching and subscriptions.

```js
useEffect(() => {
  fetchData();
}, []);
```

This is also where many developers struggle. Dependency arrays are explicit and can be unforgiving. If you miss a dependency, you risk stale values however, add too many and you may trigger unnecessary re-renders and/or crash the call stack.

### 3. useContext

useContext allows you to access values from a React Context without the need for prop drilling.

```js
const theme = useContext(ThemeContext);
```

useContext is great for shared data that many components need, but it’s not meant for complex and frequently changing state. For that, a dedicated state management solution like Redux is better.

### 4. useRef

useRef stores a mutable value that persists across renders without causing a re-render.

```js
const inputRef = useRef(null);
```

It’s commonly used for accessing DOM elements, but it’s also useful for storing timers, previous values or escape hatches (data that doesn’t naturally belong in React state) where state would be overkill.

### 5. useMemo (and useCallback)

These hooks are used for memoization. Pretty self explanatory 😉

```js
const expensiveValue = useMemo(() => compute(), [deps]);
```

They can improve performance, but premature memoization often makes code harder to read with little real benefit. Measure first and then optimise.

## How to Make Your Own Custom Hook

Custom hooks are where React Hooks really shine. A custom hook is simply a function that calls other hooks and encapsulates reusable logic.

By convention, custom hooks must start with the word use. The following is an example of a hook that fetches data from an API.

```js
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(url)
      .then((res) => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);

  return { data, loading, error };
}
```

This allows components to stay clean and declarative:

```js
const { data, loading, error } = useFetch("/api/users");
```

Custom hooks solve the exact problem that Higher-Order Components and render props attempted to solve but without wrapper hell. They promote reuse, separation of concerns and composability all while keeping components readable.

And that covers all the essential theory behind React Hooks! We explored what hooks are, the challenges of wrapper hell and how hooks simplify our workflow. We also went through the rules of hooks in React, five of the most commonly used hooks and how to create custom hooks. Mastering these concepts will make it much easier to write clean, reusable and maintainable React code.
