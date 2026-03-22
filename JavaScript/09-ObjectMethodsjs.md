## Overview

Let's explore JavaScript’s **object manipulation** capabilities. It demonstrates how to extract keys, values, and entries from an object, convert an array of key‑value pairs back into an object, freeze an object to make it immutable, seal an object to prevent additions/deletions while allowing modifications, and define properties with fine‑grained control using property descriptors. It also shows iteration with `for...of` over `Object.entries`.

---

## Step‑by‑Step Analysis

### 1. `Object.keys`, `Object.values`, `Object.entries`

```javascript
const artifact = {
  name: "Obsidian Crown",
  era: "Ancient",
  value: 50000,
  material: "volcanic glass",
};

const keys = Object.keys(artifact);
const values = Object.values(artifact);
const entries = Object.entries(artifact);

console.log(keys);
console.log(values);
console.log(entries);
```

- **Concept:** These static methods return arrays of the object’s **own enumerable** property names, values, and key‑value pairs (as arrays of `[key, value]`).
- **Output:**
  - `keys`: `['name', 'era', 'value', 'material']`
  - `values`: `['Obsidian Crown', 'Ancient', 50000, 'volcanic glass']`
  - `entries`: `[['name', 'Obsidian Crown'], ['era', 'Ancient'], ['value', 50000], ['material', 'volcanic glass']]`

### 2. Iterating with `for...of` and `Object.entries`

```javascript
for (const [key, value] of Object.entries(artifact)) {
  console.log(` ${key}: ${value}`);
}
```

- **Concept:** `Object.entries` returns an array of pairs, which can be destructured in a `for...of` loop to iterate over key‑value pairs.
- **Output:**
  ```
   name: Obsidian Crown
   era: Ancient
   value: 50000
   material: volcanic glass
  ```

### 3. `Object.fromEntries`

```javascript
const priceList = [
  ["Obsidian Crown", 50000],
  ["Ruby Pendant", 30000],
  ["Iron Shield", 5000],
];

const priceObject = Object.fromEntries(priceList);
```

- **Concept:** `Object.fromEntries` transforms an iterable of key‑value pairs into an object. It’s the inverse of `Object.entries`.
- the object would be `{ "Obsidian Crown": 50000, "Ruby Pendant": 30000, "Iron Shield": 5000 }`.

### 4. `Object.freeze`

```javascript
const displayCase = {
  artifact: "Obsidian",
  location: "Hall A, Case 3",
  locked: true,
};

Object.freeze(displayCase);
delete displayCase.locked; // fails silently (or throws in strict mode)
displayCase.newProp = "test"; // fails silently
console.log(displayCase);
```

- **Concept:** `Object.freeze` makes an object immutable: cannot add, delete, or change existing properties. In non‑strict mode, modifications are silently ignored; in strict mode, they throw a `TypeError`.
- **Output:** The object remains unchanged:
  ```javascript
  { artifact: 'Obsidian', location: 'Hall A, Case 3', locked: true }
  ```

### 5. `Object.seal`

```javascript
const catalogEntry = {
  id: "ART-001",
  description: "Ancient Crows",
  verified: true,
};

Object.seal(catalogEntry);
```

- **Concept:** `Object.seal` prevents adding or deleting properties, but existing properties can still be modified. Sealed objects have all properties marked as non‑configurable.
- **No `console.log` here, but subsequent operations would show that `catalogEntry` cannot have new properties, and existing ones can be changed.**

### 6. `Object.defineProperty` with Custom Descriptors

```javascript
const secureArtificats = { name: "Ruby Pendant" };

Object.defineProperty(secureArtificats, "catelogId", {
  value: "SEC-999",
  writable: false,
  enumerable: false,
  configurable: false,
});

console.log(secureArtificats.catelogId);
secureArtificats.catelogId = "HACKED";
console.log(secureArtificats.catelogId);
```

- **Concept:** `Object.defineProperty` allows fine‑grained control over property attributes:
  - `writable: false` – the value cannot be changed.
  - `enumerable: false` – the property won’t appear in `Object.keys`, `for...in`, etc.
  - `configurable: false` – the property cannot be deleted or have its attributes changed.
- **Output:**
  - First `console.log` → `"SEC-999"`
  - Second `console.log` → `"SEC-999"` (assignment fails silently in non‑strict mode)

### 7. Iterating over `secureArtificats` (Non‑enumerable property hidden)

```javascript
for (const [key, value] of Object.entries(secureArtificats)) {
  console.log(`${key} : ${value}`);
}
```

- **Concept:** Because `catelogId` is set to `enumerable: false`, it does **not** appear in `Object.entries` or `for...in` loops.
- **Output:**
  ```
  name : Ruby Pendant
  ```
  Only the enumerable property `name` is shown.

### 8. Getting Property Descriptor

```javascript
const desc = Object.getOwnPropertyDescriptor(secureArtificats, "name");
console.log(desc);
```

- **Concept:** `Object.getOwnPropertyDescriptor` returns the descriptor for a given own property.
- **Output:** An object like:
  ```javascript
  {
    value: 'Ruby Pendant',
    writable: true,
    enumerable: true,
    configurable: true
  }
  ```

---

## Key Concepts Explained

### `Object.keys`, `Object.values`, `Object.entries`

- Return arrays of own enumerable property names, values, and key‑value pairs.
- Used for iterating, transforming, or inspecting objects.

### `Object.fromEntries`

- Converts an iterable of key‑value pairs (like an array of `[key, value]` arrays) into an object.
- Useful for reconstructing objects after transformations.

### `Object.freeze`

- Makes an object completely immutable: cannot add, delete, or modify any property.
- The object’s properties become non‑configurable, non‑writable.
- Shallow – nested objects are not frozen unless explicitly done.

### `Object.seal`

- Prevents adding or deleting properties, but existing properties can be modified.
- All properties become non‑configurable but remain writable (if they were writable before).

### `Object.defineProperty`

- Defines a new property or modifies an existing one with custom descriptors:
  - `value` – the property value.
  - `writable` – can the value be changed?
  - `enumerable` – does it appear in enumeration?
  - `configurable` – can the property be deleted or its descriptors changed?
- Once a property is marked `configurable: false`, it cannot be made configurable again, and some attributes become locked.

### Property Descriptors

- An object describing the attributes of a property.
- `Object.getOwnPropertyDescriptor` retrieves the descriptor for an own property.
- `Object.defineProperty` uses these descriptors to define properties.

### Iteration Techniques

- `for...of` with `Object.entries` gives a clean way to iterate over key‑value pairs.
- The comment at the end lists common iteration constructs: `for`, `while`, `do while`, `for...in`, `for...of`, and array methods.

---

## Interview Discussion

### Common Questions

1. **What is the difference between `Object.freeze` and `Object.seal`?**
   - `freeze` prevents any changes to the object and its existing properties (add, delete, modify). `seal` prevents adding/deleting but allows modifying existing properties.

2. **How can you make an object truly immutable?**
   - Use `Object.freeze`. For nested objects, you need to recursively freeze them (e.g., `deepFreeze` utility).

3. **What does `Object.defineProperty` do and when would you use it?**
   - It defines a property with fine‑grained control. Used when you need to set non‑enumerable properties, read‑only properties, or when implementing getters/setters.

4. **How can you iterate over an object’s own properties?**
   - Use `Object.keys(obj).forEach(...)`, `Object.entries(obj).forEach(...)`, or `for...in` with `hasOwnProperty` check.

5. **What is the difference between `for...in` and `for...of`?**
   - `for...in` iterates over enumerable property **names** (including inherited). `for...of` iterates over values of iterable objects (arrays, strings, maps, sets, etc.). For objects, you need `Object.keys/values/entries` to make them iterable.

### Best Practices

- **Use `Object.freeze` for constants** – e.g., configuration objects that should not be mutated.
- **Prefer `Object.entries` + `for...of`** for cleaner iteration over key‑value pairs.
- **Avoid modifying built‑in prototypes** – use `Object.defineProperty` carefully if you must, but prefer composition.
- **Use `Object.hasOwn` (ES2022) over `hasOwnProperty`** for safer existence checks, especially for objects that may not have the method (e.g., `Object.create(null)`).

### Follow‑Up Questions

- **What happens if you try to delete a frozen property?**
  - It fails silently (non‑strict) or throws in strict mode.

- **Can you change a property that is non‑writable?**
  - No, the assignment is ignored (non‑strict) or throws (strict).

- **How do you make a property that is not enumerable?**
  - Use `Object.defineProperty` with `enumerable: false`.

- **What is the difference between `configurable: false` and `writable: false`?**
  - `writable: false` prevents changing the property’s value. `configurable: false` prevents deleting the property or changing its descriptor (including `writable` and `enumerable`). A property can be non‑writable but configurable, allowing you to later change its value if you reconfigure it.

---
