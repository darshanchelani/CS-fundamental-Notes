## Overview

Let's understand the asynchronous programming in JavaScript using **Promises**. It shows how to create promises, handle resolution/rejection, and chain them. Additionally, the commented‑out section touches on the **microtask queue** and how promise callbacks are executed after the current synchronous code. The incomplete chain at the end hints at how promises can be combined, but it would cause a syntax error if run.

The main concepts illustrated are:

- Promise creation with `new Promise`
- Error handling with `.catch`
- Chaining promises
- Microtask vs macrotask (setTimeout) ordering
- Simple asynchronous operations: `setTimeout` simulation

---

## Step‑by‑Step Analysis

### 1. Commented Microtask Example

```javascript
// console.log("Swastik");
// Promise.resolve("resolveed value").then((v) => {
//   console.log("Microtask ", v);
// });
// console.log("Avishek");
```

- If uncommented, the output would be:
  - `"Swastik"`
  - `"Avishek"`
  - `"Microtask resolved value"`

**Why this order?**  
The `Promise.resolve().then()` callback is a **microtask**. All synchronous code runs first, then microtasks are processed. The two `console.log` statements are synchronous, so they execute before the promise callback.

### 2. Function `boilWater`

```javascript
function boilWater(time) {
  return new Promise((res, rej) => {
    console.log("Krte h ji boil water");
    if (typeof time !== "number" || time < 0) {
      rej(new Error("ms must be in number and greater than zero"));
    }
    setTimeout(() => {
      res("Ubal gya");
    }, time);
  });
}
```

- Logs `"Krte h ji boil water"` **synchronously** when the promise executor runs.
- Validates the `time` parameter; if invalid, rejects immediately.
- Otherwise, after `time` ms, resolves with `"Ubal gya"`.

### 3. Using `boilWater`

```javascript
boilWater(200)
  .then((msg) => console.log("Resolved: ", msg))
  .catch((err) => console.log("Rejected: ", err.message));
```

- Calls `boilWater(200)`.
- The executor logs `"Krte h ji boil water"` immediately.
- After 200ms, the promise resolves; the `.then` callback logs `"Resolved:  Ubal gya"`.
- If an error occurred (e.g., `boilWater("abc")`), the `.catch` would log the error message.

**Output (in order):**

1. `"Krte h ji boil water"` (immediate)
2. (after ~200ms) `"Resolved:  Ubal gya"`

### 4. Other Functions

```javascript
function grindLeaves() {
  return Promise.resolve("Leaves grounded");
}

function steepTea(time) {
  return new Promise((res) => {
    setTimeout(() => res("Steeped tea"), time);
  });
}

function addSugar(spoons) {
  return `Added ${spoons} sugar`;
}
```

- `grindLeaves` returns a promise already resolved with `"Leaves grounded"`.
- `steepTea` returns a promise that resolves after `time` ms with `"Steeped tea"`.
- `addSugar` is a **synchronous** function returning a string (not a promise).

### 5. Incomplete Chain (Syntax Error)

```javascript
grindLeaves().then(val);
```

- The `.then` handler is incomplete – it lacks a function body and a closing parenthesis. This would cause a `SyntaxError` if executed.
- The intended pattern might be:
  ```javascript
  grindLeaves()
    .then((val) => console.log(val))
    .then(() => steepTea(100))
    .then((msg) => console.log(msg))
    .then(() => addSugar(2))
    .then(console.log);
  ```
  This would chain the operations, mixing synchronous and asynchronous steps.

---

## Why Each `console.log` Executes

- **`console.log("Krte h ji boil water")`** – inside the promise executor, runs immediately (synchronously).
- **`console.log("Resolved: ", msg)`** – runs after the 200ms timeout, when the promise resolves.
- The microtask example is commented, so it doesn’t run.
- The incomplete chain would cause a syntax error, so it’s not executed.

---

## Key Concepts Explained

### 1. Promise States

- **Pending** – initial state.
- **Fulfilled** – operation completed successfully; `.then` callbacks run.
- **Rejected** – operation failed; `.catch` callbacks run.

### 2. Microtasks vs Macrotasks

- **Microtasks** (promise callbacks, `queueMicrotask`) run after the current synchronous code, before any macrotasks (e.g., `setTimeout`).
- The commented example demonstrates that promise callbacks are queued as microtasks and execute after the synchronous `console.log`s.

### 3. Promise Chaining

- `.then()` returns a new promise, allowing sequential async operations.
- If a `.then` callback returns a value, that value is wrapped in a resolved promise for the next `.then`.
- If it returns a promise, the next `.then` waits for that promise to settle.

### 4. Error Handling with `.catch`

- Catches rejections from any previous promise in the chain.
- It can also return a value to continue the chain after an error.

### 5. Mixing Sync and Async

- Functions like `addSugar` are synchronous; they can be placed in a `.then` chain and will be executed immediately (but still in the microtask queue).

---

## Interview Discussion

### Common Questions

1. **What is the difference between microtasks and macrotasks?**
   - Microtasks (e.g., promise callbacks) run before the next macrotask (e.g., `setTimeout`). This ensures promise resolutions are handled promptly.

2. **What happens if you call a promise‑returning function without handling the promise?**
   - The promise will still execute, but unhandled rejections can cause runtime errors or warnings.

3. **How do you convert the `boilWater` function to use `async/await`?**
   - Wrap the call in an `async` function:
     ```javascript
     async function makeTea() {
       try {
         const msg = await boilWater(200);
         console.log(msg);
       } catch (err) {
         console.log(err.message);
       }
     }
     ```

4. **Why does `Promise.resolve("Leaves grounded")` log nothing by itself?**
   - It only creates a resolved promise; to see the value, you must attach a `.then` handler.

### Best Practices

- Always handle promise rejections (use `.catch` or `try/catch` with `await`).
- Use `async/await` for cleaner code when sequential async operations are needed.
- Be aware of microtask timing when mixing synchronous and asynchronous code.
- Avoid starting a promise chain without attaching a final error handler.

### Follow‑Up Questions

- **What would be the output if we uncommented the microtask example?**  
  `"Swastik"`, `"Avishek"`, `"Microtask resolved value"`.

- **How would you fix the incomplete chain to chain `grindLeaves`, `steepTea`, and `addSugar`?**  
  Use `.then` callbacks that return the next promise or value, ensuring each step waits for the previous.

- **What is the purpose of `Promise.resolve`?**  
  To create an already‑fulfilled promise, useful for starting a chain or normalising a value into a promise.

---
