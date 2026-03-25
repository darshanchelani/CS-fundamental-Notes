## JavaScript Map – The Ultimate Key‑Value Collection

`Map` is a built‑in object introduced in ES6 that holds key‑value pairs. Unlike plain objects, a `Map` **preserves insertion order** and allows **keys of any type** – not just strings or symbols. This makes it ideal for situations where you need to associate data with objects, numbers, or other non‑string keys.

---

### 1. Creating a Map and Basic Methods

```javascript
const m = new Map(); // empty map
m.set(1, "one"); // stores value under key 1
console.log(m.get(1)); // "one"
console.log(m.get(2)); // undefined (key doesn't exist)
console.log(m.has(1)); // true
console.log(m.size); // 1
m.delete(1); // removes the entry
m.clear(); // removes all entries
```

**Method summary:**

- `set(key, value)` – adds or updates the entry.
- `get(key)` – returns the value (or `undefined`).
- `has(key)` – boolean existence check.
- `delete(key)` – removes the entry; returns `true` if it existed.
- `clear()` – empties the map.
- `size` – property (not a method) with the number of entries.

---

### 2. Map Keys Can Be Anything

Unlike plain objects (where keys are converted to strings), a `Map` uses **strict equality (`===`)** to compare keys. This allows using objects, functions, or other primitives as keys.

```javascript
const affiliates = new Map();
const first = { name: "vidya4sure" };
const second = { name: "devwithjay" };

affiliates.set(first, 50_000);
affiliates.set(second, 20_000);

console.log(affiliates.get(first)); // 50000
console.log(affiliates.get(second)); // 20000
```

Here, two separate objects (`first` and `second`) are distinct keys, even though they look similar. `Map` respects object identity.

---

### 3. Iteration Over a Map

A `Map` is iterable, and its default iterator returns `[key, value]` pairs – the same as `entries()`.

```javascript
const freq = new Map();
freq.set("the", 3);
freq.set("cat", 2);

// Using for...of (destructures key and value)
for (const [key, value] of freq) {
  console.log(key, value);
}
// Output:
// the 3
// cat 2

// You can also iterate over keys, values, or entries
console.log(freq.keys()); // MapIterator {'the', 'cat'}
console.log(freq.values()); // MapIterator {3, 2}
console.log(freq.entries()); // MapIterator {['the',3], ['cat',2]}
```

All iterators are **ordered** – the same order as insertion.

---

### 4. Practical Example: Word Frequency

```javascript
const text = "the cat sat on the mat the cat";
const freq = new Map();

for (const word of text.split(" ")) {
  const wordFreq = freq.get(word) || 0;
  freq.set(word, wordFreq + 1);
}

console.log(freq);
// Map(6) { 'the' => 3, 'cat' => 2, 'sat' => 1, 'on' => 1, 'mat' => 1 }
```

- `text.split(" ")` creates an array of words.
- For each word, we get its current frequency (or 0 if not present), increment it, and store back.

---

### 5. Converting Between Map and Object

#### Object → Map

Use `Object.entries(obj)` to get an array of `[key, value]` pairs, then pass it to the `Map` constructor.

```javascript
let obj = { name: "Ashu", age: 22 };
let map = new Map(Object.entries(obj));
console.log(map); // Map(2) { 'name' => 'Ashu', 'age' => 22 }
```

#### Map → Object

Use `Object.fromEntries(map)` (ES2019) – it accepts an iterable of `[key, value]` pairs.

```javascript
let obj1 = Object.fromEntries(map);
console.log(obj1); // { name: 'Ashu', age: 22 }
```

**Important:** `Object.fromEntries` converts only string keys. If the map contains non‑string keys (e.g., objects), they become `"[object Object]"` keys – not recommended.

---

### 6. Map vs Plain Object – When to Use What

| Feature          | Map                                    | Plain Object                             |
| ---------------- | -------------------------------------- | ---------------------------------------- |
| **Key types**    | Any (objects, functions, primitives)   | Strings or Symbols                       |
| **Key order**    | Insertion order                        | Integer keys first, then insertion order |
| **Iteration**    | Directly iterable (`for...of`)         | Need `Object.keys()` etc.                |
| **Size**         | `map.size` property                    | `Object.keys(obj).length`                |
| **Performance**  | Better for frequent additions/removals | Good for static objects                  |
| **JSON support** | Not directly; must convert             | `JSON.stringify` works                   |

**Use Map when:**

- You need keys that are not strings/symbols.
- You frequently add/remove entries and need efficient iteration.
- You need to preserve insertion order.

**Use plain objects when:**

- You have simple string‑keyed data.
- You need to send data over JSON.
- You are using object literals for configuration.

---

### 7. Additional Map Methods

- `forEach(callback, thisArg)` – iterates over entries in insertion order.

```javascript
freq.forEach((value, key) => {
  console.log(key, value);
});
```

- `Map` can be created from an iterable of `[key, value]` pairs:

```javascript
const newMap = new Map([
  ["a", 1],
  ["b", 2],
]);
```

- `Map` keys are compared using `SameValueZero` algorithm (like `===`, except that `NaN` is considered equal to itself).

---

### 8. Complete Code from Your Snippet (Annotated)

```javascript
// 1. Basic Map
const m = new Map();
m.set(1, "one");
console.log(m.get(1)); // one

// 2. Word frequency
const text = "the cat sat on the mat the cat";
const freq = new Map();

for (const word of text.split(" ")) {
  const wordFreq = freq.get(word) || 0;
  freq.set(word, wordFreq + 1);
}
console.log(freq);
// Output: Map(6) { 'the' => 3, 'cat' => 2, 'sat' => 1, 'on' => 1, 'mat' => 1 }

// 3. Using objects as keys
const affiliates = new Map();
const first = { name: "vidya4sure" };
const second = { name: "devwithjay" };
affiliates.set(first, 50_000);
affiliates.set(second, 20_000);
console.log(affiliates.get(first)); // 50000

// 4. Object to Map conversion
let obj = { name: "Ashu", age: 22 };
let mapFromObj = new Map(Object.entries(obj));
console.log(mapFromObj); // Map(2) { 'name' => 'Ashu', 'age' => 22 }

// 5. Map to Object conversion
let objFromMap = Object.fromEntries(mapFromObj);
console.log(objFromMap); // { name: 'Ashu', age: 22 }
```

---

### 9. Interview‑Ready Questions

- **What is the main difference between a `Map` and a plain object?**  
  `Map` allows any key type, preserves insertion order, and has a `size` property. Plain objects only allow string/symbol keys and have more complex inheritance.

- **How do you iterate over a `Map`?**  
  `for...of` over the map itself yields `[key, value]` pairs. You can also use `forEach`, or iterate over `keys()`, `values()`, or `entries()`.

- **Can you use an object as a key in a `Map`?**  
  Yes. The key is the object reference, not its content. Two different objects with identical properties are distinct keys.

- **How do you convert an object to a `Map` and vice versa?**  
  `new Map(Object.entries(obj))` and `Object.fromEntries(map)`.

- **What does `map.set()` return?**  
  It returns the map itself, allowing method chaining: `map.set('a',1).set('b',2)`.

- **What is the default iteration of a `Map`?**  
  `map[Symbol.iterator]` is `map.entries`, so `for (const [k,v] of map)` iterates over key‑value pairs.

---

### 10. Best Practices

- Use `Map` when you need to associate metadata with objects, or when keys are dynamic.
- Prefer `Map` over objects for **caches** because of better performance with frequent updates.
- When you need to send a `Map` over JSON, convert it to an object (or array of entries) first.
- Use `Map` for word frequency, counting occurrences, or any scenario where keys are not limited to strings.

---

## Summary Table

| Concept               | Description                                                                           |
| --------------------- | ------------------------------------------------------------------------------------- |
| **`new Map()`**       | Creates an empty map.                                                                 |
| **`set(key, value)`** | Adds or updates an entry. Returns the map.                                            |
| **`get(key)`**        | Returns value (or `undefined`).                                                       |
| **`has(key)`**        | Boolean check.                                                                        |
| **`delete(key)`**     | Removes entry; returns `true` if existed.                                             |
| **`clear()`**         | Empties the map.                                                                      |
| **`size`**            | Property with entry count.                                                            |
| **Iteration**         | `for...of` gives `[key, value]`; `keys()`, `values()`, `entries()` provide iterators. |
| **Keys**              | Any value (objects, functions, primitives) compared with `SameValueZero`.             |
| **Conversion**        | `new Map(Object.entries(obj))` and `Object.fromEntries(map)`.                         |

Mastering `Map` will make your code more expressive and efficient, especially when dealing with complex keys or frequent insertions and deletions.
