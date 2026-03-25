## JavaScript Set – The Collection of Unique Values

A `Set` is a built‑in object that stores **unique values** of any type. Unlike arrays, it automatically eliminates duplicates and provides efficient membership checks.

---

### 1. Creating a Set

```javascript
// Empty set
let set = new Set();

// From an iterable (e.g., array, string, another Set)
let fromArray = new Set([1, 2, 2, 3]); // Set {1, 2, 3}
let fromString = new Set("abac"); // Set {'a', 'b', 'c'}
let fromSet = new Set(fromArray); // Set {1, 2, 3}
```

---

### 2. Basic Methods

| Method          | Description                                                                               |
| --------------- | ----------------------------------------------------------------------------------------- |
| `add(value)`    | Appends `value` to the set (if not already present). Returns the set (chaining possible). |
| `delete(value)` | Removes `value`; returns `true` if it existed, else `false`.                              |
| `has(value)`    | Returns `true` if `value` is in the set.                                                  |
| `clear()`       | Removes all elements.                                                                     |
| `size`          | Property (not a method) with the number of elements.                                      |

```javascript
let s = new Set();
s.add(1).add(2).add(2); // chaining: only 1 and 2 are added
console.log(s.has(1)); // true
s.delete(2); // returns true
console.log(s.size); // 1
s.clear();
console.log(s.size); // 0
```

---

### 3. Iteration

A `Set` is iterable, and its default iterator returns the values in insertion order.

```javascript
let colors = new Set(["red", "green", "blue"]);
for (let color of colors) {
  console.log(color);
}
// red, green, blue

// Using forEach
colors.forEach((value, valueAgain, set) => {
  console.log(value); // same as valueAgain
});
```

You can also obtain iterators:

- `keys()` – same as `values()`
- `values()` – iterates over values
- `entries()` – returns `[value, value]` pairs (for compatibility with Map)

---

### 4. Real‑World Example: Counting Unique Characters and Words

The provided code snippet demonstrates `Set` for two purposes:

```javascript
let mm = `Hare Krishna Hare Krishna Krishna Krishna Hare Hare 
          Hare Rama Hare Rama Rama Rama Hare Hare 
          Hare Krishna Hare Krishna Krishna Krishna Hare Hare 
          Hare Rama Hare Rama Rama Rama Hare Hare`;

// Get unique characters (including spaces)
let uniqueChars = new Set(mm);
console.log(uniqueChars.size); // number of distinct characters

// Get unique words (splitting by spaces)
let uniqueWords = new Set(mm.split(/\s+/)); // splits on any whitespace
console.log(uniqueWords);
```

**Explanation:**

- `new Set(mm)` creates a set of all characters in the string. Duplicates are removed, so you get each distinct character (including spaces, newlines, etc.) exactly once.
- `mm.split(" ")` splits on single spaces – but if the text contains multiple spaces or newlines, using a regular expression `/\s+/` is more robust. The resulting array of words is then passed to `new Set` to get unique words.

---

### 5. Converting Set to Array

Often you need an array of unique values. Use spread syntax or `Array.from()`:

```javascript
let arrFromSet = [...uniqueWords];
let alsoArr = Array.from(uniqueWords);
```

---

### 6. Set vs Array – When to Use

| Feature             | Set                        | Array                        |
| ------------------- | -------------------------- | ---------------------------- |
| **Uniqueness**      | Guaranteed                 | Not enforced                 |
| **Order**           | Insertion order            | Indexed order                |
| **Access by index** | Not directly; must iterate | Yes (`arr[0]`)               |
| **Lookup**          | `has()` is O(1) average    | `indexOf()` is O(n)          |
| **Adding**          | `add()` (no duplicates)    | `push()` (allows duplicates) |
| **Iteration**       | `for...of`                 | `for...of` or `forEach`      |

**Use Set when:**

- You need to store unique values.
- You frequently check existence (`has()`).
- You want to automatically deduplicate data.

**Use Array when:**

- You need indexed access.
- You need to store duplicates.
- You need array methods like `map`, `filter`, etc.

---

### 7. Additional Set Features

- **Set can store any value type**, including objects, functions, and primitives. Two different objects are considered distinct even if they look identical.
- **`NaN`** is treated as equal to itself inside a Set (contrary to `===`).
- **Chaining** with `add` is possible because it returns the set.

```javascript
let set = new Set();
set.add(1).add(2).add(3);
```

---

### 8. Complete Code from Your Snippet (with Comments)

```javascript
let mm = `Hare Krishna Hare Krishna Krishna Krishna Hare Hare 
          Hare Rama Hare Rama Rama Rama Hare Hare 
          Hare Krishna Hare Krishna Krishna Krishna Hare Hare 
          Hare Rama Hare Rama Rama Rama Hare Hare`;

// Set of unique characters
let uniqueChars = new Set(mm);
console.log("Number of unique characters:", uniqueChars.size);

// Set of unique words (splitting on whitespace)
let uniqueWords = new Set(mm.split(/\s+/)); // split on any whitespace
console.log("Unique words:", uniqueWords);

// Convert to array if needed
let uniqueWordsArray = [...uniqueWords];
console.log(uniqueWordsArray);
```

**Output example:**

```
Number of unique characters: 17   (depends on actual string content)
Unique words: Set(5) { 'Hare', 'Krishna', 'Hare', 'Rama' }  // actually 'Hare', 'Krishna', 'Rama' only three unique words
```

---

### 9. Interview‑Ready Questions

- **What is the difference between a `Set` and an array?**  
  A `Set` contains only unique values, while an array can have duplicates. `Set` provides O(1) membership check with `has()`, whereas array lookup is O(n).

- **How do you convert a `Set` to an array?**  
  Using the spread operator `[...set]` or `Array.from(set)`.

- **Can a `Set` hold objects?**  
  Yes, it holds object references. Two distinct objects are considered different, even if they have the same properties.

- **What happens if you add the same value multiple times?**  
  The `add` method will not create duplicates; the second and subsequent calls have no effect.

- **How do you iterate over a `Set`?**  
  `for...of` loop, `forEach`, or using the iterators `values()`, `keys()`, `entries()`.

- **Why would you use a `Set` instead of an array for deduplication?**  
  `Set` is more concise and efficient: `[...new Set(array)]` is a one‑liner that handles deduplication.

- **What is the time complexity of `set.has()`?**  
  Typically O(1) on average (hash‑based).

---

### 10. Best Practices

- Use `Set` for tracking seen values, removing duplicates, or performing fast membership tests.
- When you need to preserve the order of insertion, `Set` already does that.
- Combine `Set` with `Array.from` or spread to convert to an array for further array operations.
- If you need to store key‑value pairs, use `Map` instead.

---

## Summary Table

| Concept             | Description                                             |
| ------------------- | ------------------------------------------------------- |
| **`new Set()`**     | Creates a set.                                          |
| **`add(value)`**    | Adds a value (if not already present); returns the set. |
| **`delete(value)`** | Removes a value; returns `true` if existed.             |
| **`has(value)`**    | Checks existence.                                       |
| **`clear()`**       | Empties the set.                                        |
| **`size`**          | Number of elements.                                     |
| **Iteration**       | `for...of`, `forEach`, `values()`, etc.                 |
| **Uniqueness**      | Automatically enforces unique values.                   |
| **Key types**       | Any value (objects, primitives).                        |

Mastering `Set` is essential for handling collections where uniqueness matters. It pairs beautifully with arrays, enabling clean and efficient code.
