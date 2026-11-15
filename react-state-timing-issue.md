---
title: "React State Timing Issue"
description: "Strange behaviour of react's useState hook"
date: "2025-02-26"
tags:
  - React
  - Hooks
  - useState
---

There is a lazy pizza delivery guy who takes orders but does not deliver immediately.

Imagine you order a pizza, and right after placing the order, you tell your friend:

> “The pizza is here!”

But in reality, it has not arrived yet, it is still being prepared.

```javascript
const [first, setFirst] = useState(0);
const [second, setSecond] = useState(0);

useEffect(() => {
  setFirst(10);
  setSecond(first); // `first` still holds the old value (0)
}, []);
```

Here, you might expect `second` to be set to `10`, but instead, it retains the old value of `first`.

Why? Because React state updates are asynchronous!

It is just like that pizza delivery guy.

Let’s take another example.

```javascript
const [count, setCount] = useState(0);

const handleClick = () => {
  setCount(count + 1);
  console.log(count); // Will log the old value!
};
```

When you call `setCount`, React does not update the state immediately. Instead, it schedules an update and triggers a re-render after the function execution completes.

If you click the button, you might expect the console to log `1`, but it logs `0`. This happens because React schedules the state update but does not apply it immediately.

### Why does React behave like this?

Because state updates are asynchronous in React.

Say we have this example:

```javascript
const [count, setCount] = useState(0);

const handleClick = () => {
  setCount(count + 1);
  setCount(count + 1);
};
```

We know that whenever a state variable changes, React performs a re-render.

Now, if state updates were synchronous, then for each `setCount`, React would re-render the component immediately after each update.

In the example above, two re-renders would happen if state updates were synchronous.

Imagine an event handler with thousands of `setCount` calls. In that case, React would re-render the component a thousand times, which would be highly inefficient in terms of optimization.

To optimize performance, React does not update the state immediately. Instead, it schedules the update and defers the actual state change until after the function execution completes.

This allows React to batch multiple updates and trigger a single re-render, rather than re-rendering the component multiple times.

### Why does React schedule state updates?

React follows a two-phase rendering process:

#### Render Phase (Reconciliation)

- React calculates the updates by comparing the Virtual DOM.
- Any `setState` calls do not immediately change state. Instead, React queues them for later execution.

#### Commit Phase

- React applies the updates and triggers a single re-render after all state updates in the function have been processed.

This design prevents unnecessary re-renders and optimizes performance by batching multiple state updates into one re-render.

### Let’s understand batch updates.

```javascript
const [count, setCount] = useState(0);

const handleClick = () => {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
};
```

`setCount(count + 1)` runs three times.

You might expect the final value of `count` to be `3`, but in reality the final value will be `1`. Why?

Since state updates are asynchronous and batched, all three updates read the initial `count` value `0`.

Effectively, all three calls do:

```javascript
setCount(0 + 1);
setCount(0 + 1);
setCount(0 + 1);
```

React batches them and applies the updates, but since all three updates calculate the same value (`1`), the final value is `1`.

### When does React batch state updates?

React automatically batches updates and performs a single re-render whenever there are multiple state updates inside event handlers, `useEffect` hooks, timeouts, and asynchronous operations.

#### Batched updates (Inside React event handlers)

```javascript
const handleClick = () => {
  setCount(count + 1);
  setText("Updated!");
};
```

React batches `setCount` and `setText` and performs a single re-render.

#### Batched updates (Inside `useEffect`)

```javascript
useEffect(() => {
  setCount(count + 1);
  setText("Changed!");
}, []);
```

React batches these updates and only re-renders once.

#### Batched updates (Inside timeouts)

```javascript
setTimeout(() => {
  setCount(count + 1);
  setText("Async update");
}, 1000);
```

React batches these updates inside timeouts.

### How do we properly handle these situations?

Use functional updates.

If we take one of our previous examples:

#### Without functional updates (Bad code)

```javascript
const [count, setCount] = useState(0);

const handleClick = () => {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
};
```

#### Using functional updates (Good code)

```javascript
const handleClick = () => {
  setCount(prev => prev + 1);
  setCount(prev => prev + 1);
  setCount(prev => prev + 1);
};
```

### What Happens Now?

The function `prev => prev + 1` uses the most recent state value.

Effectively executed as:

```javascript
setCount(0 + 1); // count = 1
setCount(1 + 1); // count = 2
setCount(2 + 1); // count = 3
```

Also, there is `flushSync` from `react-dom`.

```javascript
import { flushSync } from "react-dom";

const handleClick = () => {
  flushSync(() => {
    setCount(count + 1);
  });

  console.log(count); // Logs new value
};
```

`flushSync` ensures React processes the state update immediately, before the next render.

So, this is all about the state timing issue.

Have fun!