## JavaScript Objects – A Comprehensive Guide

Objects are one of the most fundamental building blocks in JavaScript. They store data in key‑value pairs and serve as the backbone for almost every non‑primitive structure. In this guide, we’ll explore everything from basic object creation to advanced topics like shallow vs deep cloning, property descriptors, and object references. Each concept is accompanied by code examples drawn from your provided snippet.

---

### 1. Object Creation

Objects can be created in two primary ways:

#### a) Object Constructor Syntax

```javascript
let gemini = new Object(); // rarely used
```

#### b) Object Literal (preferred)

```javascript
let claude = {}; // empty object
```

#### c) Initialising with Properties

```javascript
let gpt = {
  company: "openai", // key: value
  version: 5.3,
  releaseYear: 2025,
};
```

**Key Points:**

- **Keys** are strings (or Symbols). If a key is a valid identifier, quotes can be omitted.
- **Values** can be any JavaScript type (primitives, functions, other objects).

---

### 2. Accessing and Modifying Properties

#### Dot Notation

```javascript
console.log(gpt.company); // "openai"
```

- Use when property name is a valid identifier (no spaces, starts with letter/underscore/$).

#### Bracket Notation

```javascript
console.log(gpt["company"]); // "openai"
console.log(sonnet["released on"]); // multi‑word keys
console.log(sonnet[1]); // numeric keys
```

- Useful for dynamic keys, keys with spaces, or numeric property names.

#### Adding a New Property

```javascript
gpt.type = "Large Language Model"; // dot
gpt.isMultiModal = true; // any type
```

#### Modifying an Existing Property

```javascript
gpt.type = "LLM";
```

#### Deleting a Property

```javascript
delete gpt.type; // removes the property
console.log(gpt.type); // undefined
```

---

### 3. Trailing Commas (Optional)

JavaScript allows a trailing comma after the last property – it’s ignored but can make version control diffs cleaner.

```javascript
let sonnet = {
  company: "anthropic",
  version: 4.6,
  "released on": 2026,
  1: "claude hi hai", // trailing comma allowed
};
```

---

### 4. Using Expressions as Property Names

With bracket notation, you can use any expression (e.g., a variable) as the key.

```javascript
const input = "company";
console.log(sonnet[input]); // "anthropic"
```

---

### 5. Property Shorthand

When creating an object from variables, if the property name matches the variable name, you can use shorthand.

```javascript
function getLaptop(name, price) {
  return {
    brand: "Apple",
    name, // same as name: name
    price, // same as price: price
  };
}
let myMac = getLaptop("M4 Air", 99_900);
console.log(myMac); // { brand: 'Apple', name: 'M4 Air', price: 99900 }
```

---

### 6. Checking Property Existence

Two common ways:

- **`in` operator** – checks own or inherited properties.
- **`undefined` comparison** – works only for own properties that might legitimately hold `undefined` (rare).

```javascript
console.log(myMac.ram === undefined); // true (property doesn't exist)
console.log("ram" in myMac); // false (property doesn't exist)
```

> ⚠️ Using `undefined` can be misleading if the property exists but has the value `undefined`. Prefer `in` or `hasOwnProperty`.

---

### 7. Looping Over Objects: `for...in`

`for...in` iterates over enumerable properties (including inherited ones, unless filtered).

```javascript
for (let key in myMac) {
  console.log(key); // "brand", "name", "price"
  console.log(myMac[key]); // "Apple", "M4 Air", 99900
}
```

**Best Practice:** To avoid inherited properties, combine with `hasOwnProperty`:

```javascript
for (let key in myMac) {
  if (myMac.hasOwnProperty(key)) {
    // own properties only
  }
}
```

---

### 8. Property Order

JavaScript objects have a property order rule:

- **Integer indices** (like `"1"`, `"2"`) appear first, sorted in ascending order.
- **Other properties** (strings, symbols) appear in the order they were created.

```javascript
let codes = {
  "+7": "Russia",
  "+32": "Belgium",
  "+91": "India",
  "+1": "Canada",
  "+52": "Mexico",
};

for (let code in codes) {
  console.log(code); // Output order: +1, +7, +32, +52, +91 (numeric keys first)
}
```

> Note: The `+` prefix makes these keys numeric after conversion (e.g., `+1` becomes `1`). They are sorted as numbers.

---

### 9. Reference vs Value Copy

#### Primitives are copied by value

```javascript
let like = "Radhika Das";
let love = like; // love gets a copy of the value
like = "Taylor Swift";
console.log(love); // "Radhika Das" – unchanged
```

#### Objects are copied by reference

```javascript
let artist = { name: "Radhika Das", country: "UK" };
let kirtaniya = artist; // both reference the same object
artist.country = "England";
console.log(kirtaniya); // { name: "Radhika Das", country: "England" }
```

#### Reference equality

```javascript
let a = {};
let b = {};
console.log(a === b); // false – different objects
```

---

### 10. Const and Object Mutability

Declaring an object with `const` prevents reassigning the variable, but does **not** make the object itself immutable.

```javascript
const ev = { name: "Mahindra be6" };
ev.name = "BYD Seal"; // allowed – property mutation
// ev = {};                // TypeError: Assignment to constant variable
```

---

### 11. Cloning Objects

Because objects are stored by reference, a simple assignment does not create a copy. To create a new object with the same properties, you need to clone.

#### Shallow Clone (own enumerable properties)

- **Manual loop**

```javascript
const original = { k1: "v1", k2: "v2" };
let clone = {};
for (let key in original) {
  clone[key] = original[key];
}
```

- **`Object.assign()`**

```javascript
let clone = Object.assign({}, original);
```

> `Object.assign` copies only enumerable own properties. It also does a shallow copy – nested objects are still shared.

#### Shallow Clone with Spread Operator (ES6)

```javascript
let clone = { ...original };
```

#### Deep Clone – Nested Objects

When an object contains nested objects, a shallow clone will only copy the top‑level references, leaving inner objects shared. To avoid this, use **deep cloning**.

- **`structuredClone()`** (modern, built‑in)

```javascript
const nestedObj = {
  model: "gpt",
  version: "5.3",
  capabilities: {
    reasoning: true,
    codeGeneration: true,
    // ...
  },
};
const nestedClone = structuredClone(nestedObj);
nestedObj.version = "5.2";
console.log(nestedClone.version); // still "5.3" – independent
```

`structuredClone` works for most data types (objects, arrays, primitives, Dates, etc.) but not for functions, DOM nodes, or symbols.

- **JSON trick** (limited, loses methods, undefined, etc.)

```javascript
const deepClone = JSON.parse(JSON.stringify(nestedObj));
```

---

### 12. Summary of Key Concepts

| Concept            | Explanation                                                             |
| ------------------ | ----------------------------------------------------------------------- |
| Creation           | `{}` literal or `new Object()`.                                         |
| Access             | Dot (`.`) for valid identifiers; bracket (`[]`) for dynamic/weird keys. |
| Add/Modify/Delete  | Direct assignment or `delete` operator.                                 |
| Property Shorthand | `{ name, price }` when variable names match.                            |
| Existence Check    | `"key" in obj` or `obj.hasOwnProperty("key")`.                          |
| Looping            | `for...in` (including inherited) – use `hasOwnProperty` filter.         |
| Order              | Integer‑like keys first, then creation order for others.                |
| Reference vs Value | Primitives are copied by value; objects are copied by reference.        |
| `const` & Objects  | `const` prevents reassignment, not property mutation.                   |
| Cloning            | Shallow: `{...obj}`, `Object.assign`; Deep: `structuredClone`, JSON.    |

---

## Interview‑Ready Questions (Based on This Code)

1. **What will be logged?**

   ```javascript
   let a = { x: 1 };
   let b = a;
   b.x = 2;
   console.log(a.x);
   ```

   **Answer:** `2` – objects are referenced.

2. **How do you check if an object has a property called `"name"`?**
   - `"name" in obj` (includes inherited)
   - `obj.hasOwnProperty("name")` (own only)

3. **Why does `for...in` loop sometimes iterate properties in a different order than they were added?**
   - Integer keys are sorted; others keep insertion order.

4. **What is the difference between shallow and deep cloning?**
   - Shallow copies top‑level properties; nested objects remain shared. Deep creates a completely independent copy.

5. **Why does `const obj = {}; obj.x = 1` work, but `obj = {}` fails?**
   - `const` prevents reassigning the variable, but the object itself can be mutated.

---
