## Overview

Let's explore **Symbols** in JavaScript—a primitive type introduced in ES6. Symbols are unique, immutable values often used as property keys to avoid name collisions. The code also demonstrates **well‑known symbols** like `Symbol.iterator` (to make objects iterable) and `Symbol.toPrimitive` (to control type conversion). Through these examples, the code shows how Symbols provide privacy, uniqueness, and customization.

---

## Step‑by‑Step Analysis

### 1. Creating Symbols and Their Properties

```javascript
const aadhaar_of_mayur = Symbol("aadhaar");
const aadhaar_of_piyush = Symbol("aadhaar");

console.log(typeof aadhaar_of_mayur);
console.log(aadhaar_of_mayur === aadhaar_of_piyush);
console.log(aadhaar_of_mayur.toString());
console.log(aadhaar_of_mayur.description);
```

- **`Symbol("aadhaar")`** – creates a new Symbol with the optional description `"aadhaar"`. Each Symbol is guaranteed to be unique, even with the same description.
- **`typeof`** – returns `"symbol"` for Symbols.
- **`===`** – compares two Symbols; they are different objects, so `false`.
- **`toString()`** – returns `"Symbol(aadhaar)"`; includes the description.
- **`.description`** – a property that holds the description string (ES2019). For the first two, it’s `"aadhaar"`.

```javascript
const nonIndian = Symbol();
console.log(nonIndian.description);
```

- **`Symbol()`** without a description – `description` returns `undefined`.

### 2. Using Symbols as Object Keys

```javascript
const biometricHash = Symbol("biometricHash");
const bloodGroup = Symbol("bloodGroup");

const citizenRecord = {
  name: "Ved Pandey",
  age: 21,
  [biometricHash]: "a7yknfky788dn",
  [bloodGroup]: "O+",
};
```

- Symbols can be used as computed property keys (inside `[ ]`). They are **non‑enumerable** by default, meaning they won’t appear in `for...in` loops or `Object.keys()`.
- `Object.keys(citizenRecord)` returns only the string‑keyed enumerable properties (`["name", "age"]`).
- `Object.getOwnPropertySymbols(citizenRecord)` returns an array of the symbol keys (`[ Symbol(biometricHash), Symbol(bloodGroup) ]`).

### 3. Custom Iterable with `Symbol.iterator`

```javascript
const rtiQueryBook = {
  queries: ["Infra budget", "Ration Card", "Education budget", "Startup laws"],
  [Symbol.iterator]() {
    let index = 0;
    const queries = this.queries;
    return {
      next() {
        if (index < queries.length) {
          return { value: queries[index++], done: false };
        }
        return { value: undefined, done: true };
      },
    };
  },
};

for (const query of rtiQueryBook) {
  console.log(`Filing RTI: ${query}`);
}
```

- **`Symbol.iterator`** is a well‑known symbol used to define the iterator for an object. When an object has an `[Symbol.iterator]` method that returns an iterator (an object with a `next` method), it becomes **iterable** and can be used with `for...of`, spread, etc.
- Here, the iterator yields each query string.
- The loop logs each query with a prefix.

### 4. Custom Type Conversion with `Symbol.toPrimitive`

```javascript
const governmentScheme = {
  name: "PM Kisan Yojna",
  people: 54,
  [Symbol.toPrimitive](hint) {
    if (hint === "string") return this.name;
    if (hint === "number") return 88;
  },
};

console.log(+governmentScheme);
console.log(`${governmentScheme}`);
```

- **`Symbol.toPrimitive`** is a well‑known symbol that lets you control how an object is converted to a primitive value. It receives a `hint` (`"string"`, `"number"`, or `"default"`).
- `+governmentScheme` triggers the `"number"` hint → returns `88`.
- Template literal `${governmentScheme}` triggers the `"string"` hint → returns `"PM Kisan Yojna"`.

---

## Why Each `console.log` Executes and Its Output

| Code                                                       | Output                                          | Concept                                                  |
| ---------------------------------------------------------- | ----------------------------------------------- | -------------------------------------------------------- |
| `console.log(typeof aadhaar_of_mayur)`                     | `"symbol"`                                      | `typeof` on a Symbol.                                    |
| `console.log(aadhaar_of_mayur === aadhaar_of_piyush)`      | `false`                                         | Symbols are unique; even same description ≠ same Symbol. |
| `console.log(aadhaar_of_mayur.toString())`                 | `"Symbol(aadhaar)"`                             | `toString()` returns a string representation.            |
| `console.log(aadhaar_of_mayur.description)`                | `"aadhaar"`                                     | The description property.                                |
| `console.log(nonIndian.description)`                       | `undefined`                                     | Symbol without description → description is `undefined`. |
| `console.log(Object.keys(citizenRecord))`                  | `["name", "age"]`                               | Symbol‑keyed properties are non‑enumerable.              |
| `console.log(Object.getOwnPropertySymbols(citizenRecord))` | `[ Symbol(biometricHash), Symbol(bloodGroup) ]` | Retrieves only symbol keys.                              |
| `for...of` loop logs                                       | `"Filing RTI: Infra budget"`, etc.              | Iterable protocol via `Symbol.iterator`.                 |
| `console.log(+governmentScheme)`                           | `88`                                            | `Symbol.toPrimitive` with `"number"` hint.               |
| `console.log(`${governmentScheme}`)`                       | `"PM Kisan Yojna"`                              | `Symbol.toPrimitive` with `"string"` hint.               |

---

## Interview Discussion

### Common Questions

1. **What are Symbols and why would you use them?**
   - Symbols are unique, immutable primitives used as property keys to avoid name collisions. They are especially useful for adding “hidden” properties that won’t appear in normal enumeration.

2. **How do you retrieve symbol properties from an object?**
   - Use `Object.getOwnPropertySymbols(obj)`. Note that they are not enumerated by `Object.keys`, `for...in`, or `JSON.stringify`.

3. **What is the difference between `Symbol.for` and `Symbol`?**
   - `Symbol.for(key)` creates/retrieves a Symbol from a global registry; the same key always returns the same Symbol. `Symbol()` always creates a new, unique Symbol.

4. **What are well‑known symbols?**
   - Built‑in symbols like `Symbol.iterator`, `Symbol.toPrimitive`, `Symbol.toStringTag`, etc. They allow you to customize language behaviour (e.g., iteration, type conversion).

5. **How do you make a custom object iterable?**
   - Implement the `[Symbol.iterator]` method that returns an iterator object with a `next()` method. Then the object can be used with `for...of`, spread, etc.

### Best Practices

- **Use Symbols for “private” or metadata properties** – they help avoid accidental conflicts with string keys.
- **Prefer `Symbol.for` only when you need cross‑realm sharing** – otherwise, use `Symbol()` to guarantee uniqueness.
- **Implement `Symbol.iterator`** when you want to make your own data structures work with built‑in iteration constructs.
- **Use `Symbol.toPrimitive`** to control type coercion if your object represents a value that should behave like a primitive in certain contexts.

### Follow‑Up Questions

- **What would happen if you try to use `JSON.stringify` on an object with Symbol keys?**  
  The Symbol‑keyed properties are omitted because they are not enumerable and JSON does not support Symbol keys.

- **Can you use a Symbol as a method name?**  
  Yes, you can assign a function to a Symbol property and call it: `obj[symbol]()`. This is common for well‑known symbols like `[Symbol.iterator]`.

- **What is the output of `Object.getOwnPropertySymbols([])`?**  
  Arrays have a built‑in `Symbol.iterator` property, so it will appear in the returned array (though it’s not enumerable, `getOwnPropertySymbols` still shows it).

- **How does `Symbol.toPrimitive` differ from `valueOf` and `toString`?**  
  `Symbol.toPrimitive` gives you one method to handle all coercion hints; it’s more explicit and can distinguish between string, number, and default hints.

---
