## Overview

This code demonstrates the fundamental differences between `var`, `let`, and `const` in JavaScript, focusing on scoping, hoisting, reassignment, and mutability. It also shows how variables declared with `var` can "leak" out of blocks, while `let` and `const` are block‑scoped. Additionally, it illustrates that `const` prevents reassignment of the variable itself, but does **not** make the contents of an object or array immutable.

The script contains several `console.log` statements that print variable values, and one line that would cause an error (commented out) to highlight the difference between reassigning a `const` variable and modifying its properties.

---

## Step‑by‑Step Analysis

### 1. `var` – Function‑scoped, can be redeclared

```javascript
var shipName = "The Amber";
console.log("Shipname: ", shipName);
```

- **Output:** `Shipname:  The Amber`
- **Concept:** `var` declares a variable that is function‑scoped (or globally scoped if outside a function). It is hoisted, meaning the declaration is moved to the top of its scope, but the initialisation stays. Here, the variable is declared and assigned, then logged.

### 2. `let` – Block‑scoped, can be reassigned

```javascript
let crewCount = 12;
console.log("crew count: ", crewCount);
crewCount = 14;
```

- **Output:** `crew count:  12`
- **Concept:** `let` is block‑scoped. The value can be changed (reassigned). The second `crewCount` assignment is not logged, so the console only shows the initial value.

### 3. `const` – Block‑scoped, cannot be reassigned

```javascript
const captainName = "Jack Sparrow";
console.log("Captain Name: ", captainName);
// captainName = "Dipesh";   // Would throw TypeError: Assignment to constant variable.
```

- **Output:** `Captain Name:  Jack Sparrow`
- **Concept:** `const` also has block scope, but it must be initialised at declaration and cannot be reassigned. The commented line demonstrates that reassignment is forbidden.

### 4. Block scope with `var` – leaks out

```javascript
if (true) {
  var leakyTreasure = "Gold coins";
}
```

- No immediate console log, but later:

```javascript
console.log(leakyTreasure);
```

- **Output:** `Gold coins`
- **Concept:** `var` is not block‑scoped; it is scoped to the containing function or global. Therefore, `leakyTreasure` is accessible outside the `if` block. This demonstrates the “leaky” behaviour of `var`.

### 5. `for` loops – scope of `var` vs `let`

```javascript
for (var i = 0; i < 10; i++) {
  //
}
for (let j = 0; i < 10; i++) {
  //
}
```

- The second loop uses `i` (which is still in scope because `var i` is global/function‑scoped) but tries to redeclare `j` with `let`. the loop shows that `var i` is accessible after the first loop, while `let j` would be scoped only to the second loop (if it were used). It highlights the difference in scoping: `var i` is accessible outside its loop, `let j` is not.

### 6. More `let` declarations – naming conventions

```javascript
let shipSpeed = 22;
let _privatelog = "secret";
let MONGODB_URI = "";
let name = "hitesh";
```

- These are just declarations;They show that variable names can include underscores, uppercase letters, and are case‑sensitive. No errors are thrown.

### 7. `const` with objects – mutability

```javascript
const treasureChest = {
  gold: 100,
  rubies: 50,
  maps: 2,
};

treasureChest.gold = 150;
// treasureChest = { gold: 50 };
```

- **Concept:** `const` prevents reassignment of the variable itself. However, the object’s properties **can** be changed. So `treasureChest.gold = 150` is allowed. The commented line would throw an error because it tries to assign a new object to `treasureChest`.

### 8. `const` with arrays – mutability

```javascript
const crewRoster = ["Alok", "Abhinav", "Tasnish"];
crewRoster.push("vraj");
crewRoster[0] = "Shubham";

crewRoster = ["Someone"]; // Error: Assignment to constant variable.
```

- **Concept:** Similarly, the array contents can be modified (`push`, index assignment) but the variable cannot be reassigned. The last line would throw a TypeError, so it’s commented out (or would cause a runtime error).

---

## Concepts Explained

### `var` vs `let` vs `const`

| Feature       | `var`                             | `let`                           | `const`                             |
| ------------- | --------------------------------- | ------------------------------- | ----------------------------------- |
| Scope         | Function / global                 | Block `{}`                      | Block `{}`                          |
| Hoisting      | Declared, initialised `undefined` | Declared, not initialised (TDZ) | Declared, not initialised (TDZ)     |
| Reassignment  | Yes                               | Yes                             | No (prevents variable reassignment) |
| Redeclaration | Allowed in same scope             | Not allowed in same scope       | Not allowed in same scope           |

### Temporal Dead Zone (TDZ)

`let` and `const` are hoisted but are not accessible until the declaration is evaluated; accessing them before throws a `ReferenceError`. `var` is hoisted and initialised with `undefined`, so it’s accessible (but `undefined`).

### Block Scope

- `var` ignores blocks (like `if`, `for`). The variable `leakyTreasure` is accessible outside the `if` block because it’s function‑scoped (or globally scoped here).
- `let` and `const` respect block scope; they are only accessible within the `{}` where they are defined.

### Const and Mutability

- `const` only prevents reassigning the variable. It does **not** make the value immutable. Objects and arrays declared with `const` can still have their properties/elements changed.

### Variable Naming

- Identifiers can include letters, digits, underscores, and dollar signs, but cannot start with a digit. Conventionally, constants use `UPPER_CASE`, private variables may start with an underscore, and environment variables often use `UPPER_CASE`.

---

## Interview Discussion

### Common Questions

1. **What is the difference between `var`, `let`, and `const`?**
   - Explain scope, hoisting, reassignment, and redeclaration.
2. **Why does `var` allow reassignment and redeclaration while `let` and `const` do not?**
   - Historical reasons; `let` and `const` were introduced in ES6 to fix scoping issues.
3. **Can you change the value of a `const` object?**
   - Yes, properties can be modified, but the variable cannot be reassigned.
4. **What is the Temporal Dead Zone?**
   - The period between entering scope and the actual declaration where `let`/`const` variables exist but cannot be accessed.
5. **Why is it recommended to use `let` and `const` over `var`?**
   - They provide block scoping, reduce bugs from accidental redeclaration, and prevent hoisting pitfalls.

### Best Practices

- Use `const` by default; use `let` only when you know the variable will be reassigned.
- Avoid `var` in modern code unless you have a specific need (e.g., writing code that must run in very old environments).
- Always declare variables at the top of their scope to avoid confusion with hoisting.
- Be mindful of block scope when using loops and conditionals.

### Follow‑Up

- **How does hoisting work with `let` and `const`?**  
  They are hoisted but remain uninitialised; accessing them before declaration throws `ReferenceError`.
- **What happens if you try to redeclare a `let` variable in the same scope?**  
  SyntaxError: Identifier already declared.

---
