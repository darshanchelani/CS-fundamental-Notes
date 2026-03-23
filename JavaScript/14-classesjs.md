## Overview

Let's explore **JavaScript classes** introduced in ES6. It demonstrates how to define a class, create instances with `new`, and examines the behaviour of instance properties and methods. It also highlights the difference between methods defined on the prototype (shared) and methods defined as arrow functions inside the constructor (instance‑specific). The code logs the result of `hasOwnProperty` on an instance, the `typeof` a class, and compares two instance methods to show they are distinct.

---

## Step‑by‑Step Analysis

### 1. Class `Cricketer` – Prototype Method

```javascript
class Cricketer {
  constructor(name, role) {
    this.name = name;
    this.role = role;
    this.matchesPlayed = 0;
    this.stamina = 100;
  }

  introduce() {
    return `${this.name} the ${this.role} | matchesPlayed: ${this.matchesPlayed} | stamina: ${this.stamina} `;
  }
}

const player1 = new Cricketer("Virat", "Batsman");
const player2 = new Cricketer("Bumrah", "Bowler");
```

- **Concept:** The `class` syntax is syntactic sugar over the constructor + prototype pattern.
- `constructor` is called when `new` is used, setting own properties on the new instance.
- `introduce` is defined on `Cricketer.prototype` – shared by all instances.
- `player1` and `player2` each have their own `name`, `role`, `matchesPlayed`, `stamina`.

### 2. `hasOwnProperty` Check

```javascript
console.log(player1.hasOwnProperty("name"));
```

- **Output:** `true`
- **Concept:** `hasOwnProperty` checks if the property exists directly on the object (not inherited). Since the constructor assigns `this.name`, it’s an own property. This demonstrates that instance properties are stored on the instance itself.

### 3. `typeof` a Class

```javascript
console.log(typeof Cricketer);
```

- **Output:** `"function"`
- **Concept:** Classes in JavaScript are actually functions (constructors) under the hood. `typeof` returns `"function"` for a class.

### 4. Class `Debutant` – Instance‑Specific Arrow Function

```javascript
class Debutant {
  constructor(name) {
    this.name = name;
    this.walkOut = () => `${this.name} walks out to bat for the first time`;
  }
}

const debutant1 = new Debutant("Shubman");
const somethingFromLastClass = debutant1.walkOut;

const debutant2 = new Debutant("Yashasvi");
console.log(debutant1.walkOut === debutant2.walkOut);
```

- **Concept:** Inside the constructor, `this.walkOut = () => ...` assigns an **arrow function** to each instance. Arrow functions capture the `this` of the surrounding context (the constructor), so each instance gets its own function object.
- Because each call to `new Debutant` creates a new arrow function, `debutant1.walkOut` and `debutant2.walkOut` are **different** function objects.
- **Output:** `false`

### 5. Commented Code

```javascript
// console.log(somethingFromLastClass());
```

- If uncommented, it would call the arrow function `debutant1.walkOut`, which retains the `this` of the `debutant1` instance, logging `"Shubman walks out to bat for the first time"`.

---

## Why Each `console.log` Executes and Its Output

| Code                                                   | Output       | Concept                                                                                |
| ------------------------------------------------------ | ------------ | -------------------------------------------------------------------------------------- |
| `console.log(player1.hasOwnProperty("name"))`          | `true`       | Instance properties are own properties.                                                |
| `console.log(typeof Cricketer)`                        | `"function"` | Classes are functions.                                                                 |
| `console.log(debutant1.walkOut === debutant2.walkOut)` | `false`      | Arrow functions defined inside the constructor create separate functions per instance. |

---

## Key Concepts Explained

### 1. Classes as Syntactic Sugar

- JavaScript classes provide a cleaner syntax for creating objects and dealing with inheritance. However, they are still based on prototypes.
- Methods defined inside a class (without using arrow functions) are added to the **prototype**, not to each instance. This is memory‑efficient.

### 2. Instance Properties vs. Prototype Methods

- Properties set in the `constructor` (using `this`) become **own properties** of each instance.
- Methods defined directly in the class body (like `introduce`) become **prototype methods** – they are shared across all instances.

### 3. Arrow Functions in Constructors

- Assigning an arrow function to `this` in the constructor creates a **new function for each instance**. This can be useful when you need to bind the instance to the function (e.g., for callbacks), but it uses more memory.

### 4. `hasOwnProperty`

- `hasOwnProperty` is a method inherited from `Object.prototype` that checks if a property exists directly on the object (not inherited). It’s useful to distinguish own properties from those in the prototype chain.

### 5. `typeof` on Classes

- In JavaScript, classes are a type of function. `typeof` returns `"function"` for classes.

---

## Interview Discussion

### Common Questions

1. **What is the difference between a method defined in a class body and a method defined as an arrow function in the constructor?**
   - Class body methods are on the prototype, shared by all instances. Arrow functions in the constructor are created per instance, each with its own closure, but they capture the instance’s `this` automatically.

2. **Why does `typeof` a class return `"function"`?**
   - Because classes are syntactic sugar over constructor functions; they are still functions internally.

3. **What does `hasOwnProperty` do and when would you use it?**
   - It checks if a property is directly on the object, not inherited. Used to avoid iterating over inherited properties in `for...in` loops or when you need to ensure a property is not from the prototype.

4. **How does the `new` operator work with a class?**
   - It creates a new object, sets its prototype to the class’s `prototype`, calls the constructor with `this` bound to the new object, and returns the object.

5. **Why is `player1.introduce === player2.introduce` `true`?**
   - Because `introduce` is on the prototype, and both instances share the same reference to that method.

### Best Practices

- **Prefer class methods (not arrow functions in constructors)** for shared behaviour – they are memory efficient.
- Use arrow functions in constructors only when you need to bind the method to the instance (e.g., to use as a callback that preserves `this`).
- Use `hasOwnProperty` carefully – it can be shadowed; in modern code, `Object.hasOwn(obj, prop)` is preferred.
- Always use `new` when instantiating a class; forgetting it leads to errors (the constructor runs in global scope).

### Follow‑Up Questions

- **What happens if you call a class without `new`?**  
  It throws a `TypeError` (classes have a `[[Construct]]` internal method but cannot be called as normal functions).

- **Can you add methods to a class after its definition?**  
  Yes, because a class is just a constructor function; you can add to its prototype: `Cricketer.prototype.newMethod = ...`.

- **What is the difference between `player1.hasOwnProperty('name')` and `'name' in player1`?**  
  `in` checks the prototype chain as well; `hasOwnProperty` only checks the object itself.

- **Why do the two arrow functions in `Debutant` produce different function objects?**  
  Because the arrow function is created anew each time the constructor runs, so each instance gets its own closure.

---
