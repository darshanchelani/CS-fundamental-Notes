## Overview

Let's explores **closures** and **function factories** in JavaScript. It demonstrates how functions can retain access to variables from their outer scope even after the outer function has returned. The `startCompany` function returns a closure that captures the inner `ca` function, which in turn accesses the outer `name` argument when called. The `eternal` function shows a more advanced closure pattern: it returns an object containing two functions (`zomato` and `blinkit`) that share private variables (`guestName` and `count`). Each call to `eternal` creates a separate closure with its own private state, allowing multiple independent “instances”.

The code also mentions `useMemo` (a React hook) and `cups.map`, hinting at performance optimization and array iteration, but these are not executed.

---

## Step‑by‑Step Analysis

### 1. Function Factory with Closures

```javascript
function startCompany() {
  function ca(name) {
    return `Name of your company is ${name}`;
  }
  return ca;
}

const getMeAcompany = startCompany();
const myCompanyName = getMeAcompany("Zomato");
```

- `startCompany` is a function that returns another function `ca`.
- When `startCompany()` is called, it returns the inner `ca` function. This returned function is stored in `getMeAcompany`.
- Calling `getMeAcompany("Zomato")` invokes `ca` with the argument `"Zomato"`, returning the string `"Name of your company is Zomato"`.
- This demonstrates a **function factory**: a function that creates and returns another function. The inner function `ca` does not capture any outer variables (it uses its own argument), so it’s a simple example of a closure that doesn't need to preserve state.

### 2. Closure with Private State

```javascript
function eternal(guest) {
  const guestName = guest;
  let count = 0;

  function zomato() {
    console.log(`Hi ${guestName}, from zomato`);
  }

  function blinkit() {
    if (count == 1) return;
    console.log(`Hi ${guestName}, from blinkit`);
    count++;
  }
  // zomato();
  // blinkit();
  return {
    zomato,
    blinkit,
  };
}
```

- `eternal` takes a `guest` parameter and stores it in a constant `guestName`.
- It declares a mutable variable `count` (initialised to `0`).
- It defines two inner functions: `zomato` and `blinkit`.
- Both inner functions reference the outer variables `guestName` and `count` (closure). They form a **closure** because they “close over” these variables, preserving them even after `eternal` returns.
- The function returns an object containing references to `zomato` and `blinkit`. This object is the public interface to the private state.

**Important:** The two inner functions are **not called** inside `eternal` (they are commented out). They are only returned.

### 3. Creating Multiple Instances

```javascript
const hitesh = eternal("hitesh");
const piyush = eternal("Piyush");

hitesh.blinkit();
hitesh.blinkit();
hitesh.blinkit();
```

- Each call to `eternal` creates a **new closure** with its own `guestName` and `count` variables.
- `hitesh` gets an object with `zomato` and `blinkit` functions bound to `guestName = "hitesh"` and a `count` starting at 0.
- `piyush` gets a separate closure with `guestName = "Piyush"` and its own `count`.
- Calling `hitesh.blinkit()` invokes the `blinkit` function inside that closure.
- The `blinkit` function checks `count`: if `count === 1`, it returns without logging; otherwise it logs and increments `count` to 1.
- Therefore, the first call to `blinkit` logs the message; subsequent calls (within the same closure) do nothing.

**Output from the three calls:**

```
Hi hitesh, from blinkit
```

Only the first call logs; the next two calls hit the `return` condition.

### 4. Commented Code

```javascript
useMemo(); // Not defined – would cause ReferenceError if executed
const cups = ["green", "blue", "red"];
cups.map; // Just a property reference, not executed
```

- `useMemo` is a React hook (or a custom function). In this context, it’s not defined, so the line would throw an error if uncommented. The code likely intends to show the concept of memoization (caching results to avoid recomputation).
- `cups.map` is a property access; it does nothing by itself. The code doesn’t call `map`, so it’s just a placeholder.

---

## Why Each `console.log` Executes

The only `console.log` that actually runs is inside `hitesh.blinkit()`. Because:

- The `zomato` function is never called.
- The `blinkit` function is called three times, but the first call logs, and the subsequent ones return early.
- The `startCompany` functions are not logged; they only return strings.

Thus, the output is:

```
Hi hitesh, from blinkit
```

(No other logs appear.)

---

## Key Concepts Explained

### Closures

- A closure is a function that retains access to variables from its outer scope even after the outer function has returned.
- In `eternal`, the returned `zomato` and `blinkit` functions form closures over `guestName` and `count`. Each call to `eternal` creates a new closure with its own copy of those variables.

### Function Factory

- `startCompany` and `eternal` are both function factories: they return functions (or objects containing functions) that can be called later.

### Private State via Closures

- By returning an object that exposes only certain functions, you can hide internal variables. In `eternal`, `guestName` and `count` are not directly accessible from outside; they can only be manipulated via the returned `zomato` and `blinkit` functions. This is a classic way to achieve encapsulation in JavaScript.

### Multiple Instances

- Each call to `eternal` creates a separate closure. This is similar to creating multiple “instances” of a class, each with its own private data.

### `useMemo` and Performance

- The comment `useMemo()` is a nod to React’s `useMemo` hook, which caches the result of an expensive computation and only recomputes when dependencies change. In plain JavaScript, you can implement memoization using closures (e.g., caching previous results). This is a separate concept but related to the idea of “remembering” state.

### Array `map`

- `cups.map` is a method that transforms an array by applying a function to each element. The code does not call it, but it’s a reminder of array iteration methods.

---

## Interview Discussion

### Common Questions

1. **What is a closure? Give an example.**  
   A closure is a function that remembers its lexical scope even when executed outside that scope. The `eternal` function returns an object with `blinkit` that remembers `guestName` and `count`.

2. **How do closures help in data encapsulation?**  
   They allow you to create private variables that cannot be accessed directly from outside, only through the returned functions.

3. **What will the code log?**  
   `"Hi hitesh, from blinkit"` only once.

4. **Why does `piyush.blinkit()` not log anything?**  
   It would log its own message if called, but the code only calls `hitesh.blinkit()`. Each closure has its own `count`, so `piyush.blinkit()` would log the first time it is called.

5. **What is the difference between the closures in `startCompany` and `eternal`?**  
   `startCompany` returns a function that doesn’t capture any outer variables; it’s a simple factory. `eternal` returns an object with two functions that both close over the same private variables, allowing shared state between them.

### Best Practices

- Use closures to encapsulate state and create private variables.
- Be mindful that closures keep references to outer variables, which can lead to memory leaks if not handled carefully (e.g., large objects retained inadvertently).
- For multiple independent instances, use a factory function that returns a new closure each time.
- In modern JavaScript, classes provide a more familiar syntax for encapsulation, but closures remain a powerful tool.

### Follow‑Up Questions

- **What would happen if we called `hitesh.zomato()`?**  
  It would log `"Hi hitesh, from zomato"` (since it doesn’t have a guard). It would run every time, sharing the same `guestName` but not affecting `count`.

- **How would you make `blinkit` only run once per guest?**  
  The current code already does that by incrementing `count` and returning when `count == 1`.

- **Can we modify `count` from outside?**  
  No, because `count` is not exposed. It’s private to the closure.

---
