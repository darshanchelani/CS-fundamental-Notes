## Overview

Let's explore JavaScript **strings**—how they are created, their immutability, common methods (`length`, `charAt`, bracket notation, `at`, `toLowerCase`, `indexOf`, `slice`, `split`, `padStart`), template literals, and the `void` operator. It also shows how arrays are returned by `split` and how variables can be reassigned to `null`. The final part demonstrates that objects can be reassigned and that `null` is an intentional absence of value.

---

## Step‑by‑Step Analysis

### 1. Creating Strings

```javascript
const codeName = "Shadow Fox";
const backupName = String("Night Own");
const templateName = `Agent ${codeName}`;
```

- **Concept:** Strings can be created with quotes (`''`, `""`) or the `String` constructor (but called as a function, not `new`, it converts the argument to a string). Template literals (backticks) allow embedding expressions via `${}`.
- **No console logs yet.**

### 2. String Immutability

```javascript
let intercepted = "HELLO";
intercepted[0] = "J"; // silent fail
console.log(intercepted);
```

- **Output:** `"HELLO"`
- **Concept:** Strings are **immutable**—they cannot be changed in place. Attempting to assign to an index does nothing (silently fails in non‑strict mode). To change a string, you must create a new one.

### 3. String Properties and Methods

```javascript
const secretCode = "OMEGA-7";
console.log(secretCode.length); // 7
console.log(secretCode.charAt(99)); // "" (empty string)
console.log(secretCode[99]); // undefined
console.log(secretCode.at(-1)); // "7" (last character)
```

- **Output:**
  - `7`
  - `""` (empty string)
  - `undefined`
  - `"7"`
- **Concepts:**
  - `.length` returns the number of characters.
  - `charAt(index)` returns the character at that index, or an empty string if out of range.
  - Bracket notation (`[index]`) also returns the character, but out‑of‑range gives `undefined`.
  - `.at(index)` (ES2022) supports negative indices, counting from the end.

### 4. Case Conversion

```javascript
const rawTransmission = "ThE EaGLE has LandeD";
console.log(rawTransmission.toLowerCase()); // "the eagle has landed"
```

- **Output:** `"the eagle has landed"`
- **Concept:** `toLowerCase()` returns a new string with all uppercase letters converted to lowercase. Original string unchanged.

### 5. Searching with `indexOf`

```javascript
const message = "The drop point is at Dock 7. Repeat: Dock 7";
console.log(message.indexOf("Dock")); // 27 (first occurrence index)
```

- **Output:** `27`
- **Concept:** `indexOf` returns the first index where the substring is found, or `-1` if not found.

### 6. Extracting Substrings with `slice`

```javascript
message.slice(0, 12);
```

- **No console.log**, but this line returns a new string (starting at index 0, ending before 12) but doesn’t log it. It’s just there to show the method exists. The result is `"The drop po"`.

### 7. Splitting Strings

```javascript
const orders = "    move-north|hold-position|extract-vip";
let orderList = orders.split("|");
console.log("Split", orderList);
```

- **Output:** `Split [ '    move-north', 'hold-position', 'extract-vip' ]`
- **Concept:** `split` divides a string into an array of substrings based on a separator (`"|"`). Leading/trailing spaces are preserved.

```javascript
const myDataValue = "SOS".split("");
console.log(typeof myDataValue); // "object"
console.log(Array.isArray(myDataValue)); // true
```

- **Output:**
  - `"object"`
  - `true`
- **Concept:** `split("")` splits the string into an array of characters. `typeof` an array returns `"object"`, but `Array.isArray` correctly identifies it as an array.

### 8. Padding Strings

```javascript
const missionNumber = "42";
console.log(missionNumber.padStart(6, "0")); // "000042"
```

- **Output:** `"000042"`
- **Concept:** `padStart` adds characters to the beginning until the string reaches the given length.

### 9. Template Literals (Multiline)

```javascript
const spellCard = `

  ++==========================
  | Spell: ${myDataValue} |

  `;
```

- **Concept:** Template literals preserve whitespace and line breaks. Here, `myDataValue` is an array; when interpolated, it calls `toString()` on the array, resulting in `"S,O,S"` (commas). No console output.

### 10. The `void` Operator

```javascript
console.log(void "hitesh"); // undefined
```

- **Output:** `undefined`
- **Concept:** `void` evaluates its operand and returns `undefined`. It’s often used to obtain `undefined` safely (since `undefined` can be overwritten). Here, `void "hitesh"` yields `undefined`.

### 11. Object Reassignment and `null`

```javascript
let generalStore = { name: "Kirana", goods: 2 };
console.log(generalStore); // { name: 'Kirana', goods: 2 }
generalStore = null;
console.log(generalStore); // null
```

- **Output:**
  - `{ name: 'Kirana', goods: 2 }`
  - `null`
- **Concept:** Variables can be reassigned to any value, including `null`. `null` represents the intentional absence of any object value. The object previously referenced is now eligible for garbage collection.

---

---

## Interview Discussion

### Common Questions

1. **Are strings mutable in JavaScript?**  
   No, strings are immutable. Any method that appears to modify a string returns a new string; the original is unchanged.

2. **What is the difference between `charAt` and bracket notation?**  
   `charAt` returns an empty string for out‑of‑range indices, while bracket notation returns `undefined`. Also, `charAt` works in older JavaScript environments, but bracket notation is more modern.

3. **What does `at()` do that other methods don’t?**  
   `at` supports negative indexing, allowing you to access characters from the end of the string.

4. **How do you split a string into an array of characters?**  
   Use `str.split("")`. However, be cautious with multi‑byte characters (e.g., emojis); a better approach might be `Array.from(str)` or spread `[...str]`.

5. **What is the purpose of `void`?**  
   It forces an expression to evaluate to `undefined`. It’s rarely needed in modern JavaScript but is sometimes used to get the `undefined` value safely or in `javascript:void(0)` to prevent navigation in anchor tags.

6. **What is the difference between `null` and `undefined`?**  
   `undefined` means a variable has been declared but not assigned a value. `null` is an assignment value representing “no value” or “empty”. They are distinct types.

### Best Practices

- Use `const` for strings that won’t change; use `let` if you need to reassign.
- When manipulating strings, remember they are immutable—always capture the result.
- Prefer `at()` for negative indexing over manual calculation.
- Use `Array.from(str)` to split into characters when dealing with Unicode code points.
- Use `void 0` if you must obtain `undefined`, but it’s rarely necessary because `undefined` is a global property (though mutable in older non‑strict code).
- Use `null` to explicitly indicate “no object” when you want to clear a reference.

### Follow‑Up Questions

- **How does `split` handle empty separators?**  
  `split("")` splits into characters, but with Unicode surrogate pairs it may not work correctly. `Array.from` is safer.

- **What is the difference between `slice`, `substring`, and `substr`?**  
  `slice` works with negative indices; `substring` treats negative as 0; `substr` is deprecated (second argument is length). Prefer `slice`.

- **What are tagged template literals?**  
  They allow you to parse template literals with a function, giving access to raw strings and substitutions.

- **What happens if you use `void` with an expression that has side effects?**  
  The expression is still evaluated (side effects occur), but the result is ignored and `undefined` is returned.

---
