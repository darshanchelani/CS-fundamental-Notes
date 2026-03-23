## Overview

Let's understand **JavaScript Promises** and **async/await** for handling asynchronous operations. It covers creating promises, promise states (pending, fulfilled, rejected), chaining, static methods (`Promise.resolve`, `Promise.allSettled`), and error handling with `try/catch` in async functions. The code includes both commented examples (which are not executed) and a live part that logs a resolved promise and then a rejected promise after a delay.

---

## Step‑by‑Step Analysis

### 1. Commented Promise Example (Not Executed)

```javascript
// const promise = new Promise((res, rej) => {
//   setTimeout(() => {
//     // res("Chaicode");
//     rej(new Error("Chaicode"));
//   }, 2000);
// });
// console.log(promise);
// promise
//   .then((data) => {...})
//   .catch((error) => {...})
//   .then(console.log);
```

- This is a typical promise that rejects after 2 seconds with an error.
- The `.then` chain is commented out, so it’s not executed.
- It shows how to create a promise with `new Promise`, and how to handle resolution/rejection with `.then` and `.catch`.

### 2. `Promise.resolve` – Immediate Resolution

```javascript
const turant = Promise.resolve("Turant");
console.log(turant);
```

- `Promise.resolve` returns a promise that is already **fulfilled** with the value `"Turant"`.
- `console.log(turant)` prints the promise object. In most browsers/Node, it shows `Promise { 'Turant' }` (or `Promise { <fulfilled> }`).
- **Output:** `Promise { 'Turant' }` (the exact representation may vary).

### 3. `Promise.allSettled` – Not Logged

```javascript
const allPromise = Promise.allSettled([
  Promise.resolve("Chai"),
  Promise.resolve("Code"),
  Promise.reject("Error"),
]);
// allPromise.then(console.log);
```

- `Promise.allSettled` waits for all promises to settle (fulfill or reject) and returns a promise that resolves to an array of result objects.
- The `.then(console.log)` is commented, so no output.
- This demonstrates how to handle multiple promises without short‑circuiting on rejection.

### 4. Promise That Rejects After Delay

```javascript
const hPromise = new Promise((res, rej) => {
  setTimeout(() => {
    // res("Masterji");
    rej(new Error("Masterji"));
  }, 3000);
});
```

- `hPromise` is a promise that will reject with an `Error` after 3 seconds.
- The `res` line is commented, so it always rejects.

### 5. Async Function with `try/catch`

```javascript
async function nice() {
  try {
    const result = await hPromise;
    console.log(result);
  } catch (error) {
    console.log("Error aa gya ji", error.message);
  }
}
nice();
```

- `async function` returns a promise.
- Inside, `await hPromise` waits for the promise to settle. If it fulfills, the value is assigned to `result` and logged. If it rejects, the `catch` block runs.
- Here, after 3 seconds, `hPromise` rejects, so the `catch` executes.
- **Output (after 3 seconds):** `"Error aa gya ji Masterji"`

---

## Why Each `console.log` Executes and Its Output

| Code                                            | Output                     | Timing          |
| ----------------------------------------------- | -------------------------- | --------------- |
| `console.log(turant)`                           | `Promise { 'Turant' }`     | Immediately     |
| `console.log("Error aa gya ji", error.message)` | `Error aa gya ji Masterji` | After 3 seconds |

The other `console.log` calls are inside commented code and do **not** execute.

---

## Key Concepts Explained

### 1. Promise States

- **Pending** – initial state, neither fulfilled nor rejected.
- **Fulfilled** – operation completed successfully, resolved with a value.
- **Rejected** – operation failed, rejected with a reason.

### 2. Creating Promises

- `new Promise((resolve, reject) => { ... })` – manual creation.
- `Promise.resolve(value)` – immediately fulfilled promise.
- `Promise.reject(error)` – immediately rejected promise.

### 3. Chaining with `.then` and `.catch`

- `.then(onFulfilled, onRejected)` – handles fulfillment (and optionally rejection).
- `.catch(onRejected)` – handles rejection only.
- Chaining returns new promises, allowing sequential async operations.

### 4. `Promise.allSettled`

- Takes an iterable of promises and returns a promise that resolves after all input promises have settled (fulfilled or rejected).
- Each result is an object with `status` (`"fulfilled"` or `"rejected"`) and `value` or `reason`.
- Unlike `Promise.all`, it never rejects; it waits for all to finish.

### 5. Async/Await

- `async` functions always return a promise.
- `await` pauses the function execution until the promise settles, then returns the fulfilled value or throws the rejection reason.
- `try/catch` around `await` handles rejections (like synchronous error handling).

---

## Interview Discussion

### Common Questions

1. **What is the difference between `Promise.all` and `Promise.allSettled`?**
   - `Promise.all` rejects immediately if any promise rejects; `allSettled` waits for all to settle and returns results regardless of rejections.

2. **What does `Promise.resolve` do?**
   - Creates a promise that is already resolved with the given value. Useful for starting promise chains.

3. **How does error handling differ between `.catch` and `try/catch` in async functions?**
   - `.catch` attaches a rejection handler to a promise chain. `try/catch` works with `await` inside `async` functions, providing a synchronous‑style error handling.

4. **What will be logged in this code?**
   - Immediately: `Promise { 'Turant' }`
   - After 3 seconds: `Error aa gya ji Masterji`

### Best Practices

- Always handle promise rejections – unhandled rejections can crash Node or cause silent failures.
- Use `Promise.allSettled` when you need results of all promises, even if some fail.
- Prefer `async/await` over raw `.then` chains for better readability, but always wrap in `try/catch`.
- Avoid mixing callback and promise styles.

### Follow‑Up Questions

- **What would happen if the `res` line were uncommented and `rej` commented?**  
  The promise would fulfill with `"Masterji"` after 3 seconds, and the async function would log `"Masterji"` (not the error message).

- **Why does `console.log(turant)` show a promise instead of the string?**  
  Because `turant` is a promise object, not the resolved value. To access the value, you need to use `.then()` or `await`.

- **What is the output of `Promise.allSettled` if uncommented?**  
  It would log an array of objects:  
  `[{ status: 'fulfilled', value: 'Chai' }, { status: 'fulfilled', value: 'Code' }, { status: 'rejected', reason: 'Error' }]`

---
