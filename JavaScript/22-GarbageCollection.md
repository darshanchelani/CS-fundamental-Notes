## Garbage Collection & Memory Management in JavaScript

JavaScript automatically manages memory using a **garbage collector**. The core principle is that memory is reclaimed when objects are no longer **reachable** – that is, when there is no way to access them from the root (the global object, current function scope, etc.). Understanding how reachability works is crucial to avoiding memory leaks.

---

### 1. Reachability

An object is **reachable** if it can be accessed directly or indirectly from the **root**. The root includes:

- The global object (`window` in browsers, `global` in Node.js)
- Variables currently in the call stack (local variables in active functions)
- Any object referenced by the root or by another reachable object.

When an object is no longer reachable, it becomes eligible for garbage collection.

#### Example 1: Simple variable

```javascript
let temp = {
  email: "gibberish@xyz.com",
  valid: "5",
};

// The object is reachable via the variable `temp`

temp = null; // Now the object is unreachable – garbage collector will free it.
```

**Explanation:** After `temp = null`, there is no reference to the object, so it becomes unreachable and will be collected.

---

### 2. Garbage Collection Algorithm: Mark-and-Sweep

Modern JavaScript engines (like V8) use a **mark-and-sweep** algorithm:

1. **Mark** – start from the roots and traverse all reachable objects, marking them.
2. **Sweep** – delete any object that was not marked.

This algorithm correctly handles **circular references**, because cycles are still reachable if any object in the cycle is reachable from a root.

#### Example 2: Circular References

```javascript
function coStar(actor, actress) {
  actor.coStar = actress;
  actress.coStar = actor;
  return {
    leading: actor,
    supporting: actress,
  };
}

const movie = {
  title: "Ghosted",
  release: 2023,
  production: "Apple TV",
};

movie.cast = coStar(
  { name: "Chris Evans", salary: 10_000_000 },
  { name: "Ana de Armas", salary: 2_000_000 },
);

console.log(movie);
```

**Explanation:** The `coStar` function creates a circular reference: `actor` and `actress` each point to each other. The returned object contains references to both. The whole structure is reachable from `movie.cast`. Hence none of these objects will be garbage collected.

---

### 3. Breaking Reachability

If we later set `movie.cast = null`, the entire structure becomes unreachable (unless there are other references elsewhere). Then the garbage collector can free all involved objects, including the circular references.

```javascript
movie.cast = null;
```

Now the objects created inside `coStar` are no longer reachable from the global root. The mark-and-sweep algorithm will not mark them, so they will be collected.

---

### 4. Memory Leaks

A **memory leak** occurs when objects that are no longer needed remain reachable, preventing garbage collection. Common causes:

- **Accidental global variables** – `function foo() { x = "leak"; }` (missing `var/let/const`).
- **Forgotten timers or event listeners** that hold references to objects.
- **Closures** that inadvertently retain large objects.
- **Detached DOM elements** (in browsers) that are still referenced.

#### Example of a memory leak

```javascript
let largeArray = new Array(1000000).fill("data");
let button = document.getElementById("myButton");
button.addEventListener("click", () => {
  console.log(largeArray.length); // closure keeps reference to largeArray
});
```

Even if you no longer need `largeArray`, the event listener keeps it alive. Removing the listener or nullifying the reference can fix it.

---

### 5. Weak References

JavaScript provides `WeakMap` and `WeakSet` to hold references that **do not prevent garbage collection**. Keys in a `WeakMap` must be objects, and if the object is no longer referenced elsewhere, it can be collected, and the entry is automatically removed.

#### Use case: caching without leaking memory

```javascript
let cache = new WeakMap();

function process(obj) {
  if (!cache.has(obj)) {
    let result = heavyComputation(obj);
    cache.set(obj, result);
  }
  return cache.get(obj);
}
```

When the object `obj` becomes unreachable, the cached result is also freed.

---

### 6. Summary of Key Concepts

| Concept                 | Explanation                                                               |
| ----------------------- | ------------------------------------------------------------------------- |
| **Reachability**        | An object is reachable if it can be accessed from a root (global, stack). |
| **Garbage Collection**  | Automatic memory reclamation of unreachable objects.                      |
| **Mark-and-Sweep**      | Algorithm that marks reachable objects and sweeps (frees) the rest.       |
| **Circular References** | Handled correctly – cycles are collected if no root reference exists.     |
| **Memory Leak**         | Unintentional retention of objects due to lingering references.           |
| **WeakMap/WeakSet**     | Allow references that do not block garbage collection.                    |

---

## Interview Discussion

- **How does JavaScript manage memory?**  
  It uses garbage collection with a mark-and-sweep algorithm. The engine tracks reachable objects from roots and frees unreachable ones.

- **What is a circular reference? Can it cause memory leaks?**  
  Circular references themselves do not cause leaks as long as the entire cycle becomes unreachable. In older JavaScript engines (IE6‑7), they sometimes leaked, but modern engines handle them.

- **How would you prevent memory leaks in event listeners?**  
  Remove listeners when no longer needed, use `removeEventListener`, or use `WeakRef` / `AbortController` to clean up.

- **What is the difference between `WeakMap` and `Map`?**  
  `Map` holds strong references to keys, preventing garbage collection. `WeakMap` holds weak references; keys can be collected, and entries are automatically removed.

- **What happens when you set an object reference to `null`?**  
  The variable no longer points to the object, making it unreachable (if no other references exist) and eligible for collection.

---

## Complete Code Example with Explanations

```javascript
// =============================================
// Reachability and Garbage Collection
// =============================================

// 1. Simple object – reachable via variable 'temp'
let temp = {
  email: "gibberish@xyz.com",
  valid: "5",
};
// Object is reachable.

// After setting to null, the object becomes unreachable.
temp = null;
// Garbage collector will free the memory.

// 2. Circular reference example
const movie = {
  title: "Ghosted",
  release: 2023,
  production: "Apple TV",
};

function coStar(actor, actress) {
  // Create circular references between actor and actress
  actor.coStar = actress;
  actress.coStar = actor;
  return {
    leading: actor,
    supporting: actress,
  };
}

// The objects created inside coStar are now reachable via movie.cast.
movie.cast = coStar(
  { name: "Chris Evans", salary: 10_000_000 },
  { name: "Ana de Armas", salary: 2_000_000 },
);

console.log(movie);
// At this point, all objects are reachable.

// 3. Break the reference
movie.cast = null;
// Now the entire circular structure is unreachable and will be collected.

// 4. WeakMap example (no memory leak)
let wm = new WeakMap();
let obj = { name: "temporary" };
wm.set(obj, "some data");
obj = null; // Now the entry in wm is also eligible for collection.
```

---
