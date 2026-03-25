## JavaScript Methods, `this`, and Object References

In JavaScript, functions that are properties of an object are called **methods**. When a method is called, the object it belongs to becomes the **context** for `this`. Understanding how to define methods and how `this` behaves is crucial for object‑oriented programming. Additionally, objects are stored by reference, which affects how they are passed around and garbage‑collected.

---

### 1. Defining Methods in Objects

There are several ways to attach a function to an object:

#### a) Assign an existing function

```javascript
function viralDance() {
  console.log("Ichu Inchu Song");
}

const dogesh = {
  name: "Husky",
  dance: viralDance, // property references the function
};

dogesh.dance(); // "Ichu Inchu Song"
```

#### b) Assign a function expression

```javascript
const dogesh = {
  name: "Husky",
  dance: function () {
    console.log("Ichu Inchu Song");
  },
};
```

#### c) Method shorthand (ES6) – most concise

```javascript
const dogesh = {
  name: "Husky",
  dance() {
    console.log("Ichu Inchu Song");
  },
};
```

All three styles produce a method that can be called with `dogesh.dance()`.

---

### 2. The `this` Keyword in Methods

Inside a method, `this` refers to the object that the method was called on. This allows the method to access other properties of the same object.

```javascript
let user = {
  name: "Piyush Garg",
  age: 26,
  college: "Chitkara University",
  passout: 2021,
  gf: "Mai Ka Ladli",

  intro() {
    console.log(`Hi, my name is ${this.name}!`);
    console.log(`I am ${this.age} years old.`);
    console.log(
      `I studied at ${this.college} and passed out in ${this.passout}.`,
    );
    console.log(`My girlfriend's name is ${this.gf}.`);
  },
};

user.intro();
```

**Output:**

```
Hi, my name is Piyush Garg!
I am 26 years old.
I studied at Chitkara University and passed out in 2021.
My girlfriend's name is Mai Ka Ladli.
```

Here, `this` inside `intro()` refers to the `user` object.

---

### 3. Object References and Garbage Collection

Objects in JavaScript are stored **by reference**. When you assign an object to another variable, you are copying the reference, not the object itself. The object remains in memory as long as there is at least one reference pointing to it.

```javascript
let piyush = user; // piyush now references the same object as user
user = null; // user no longer references the object, but piyush still does

piyush.intro(); // Works perfectly – the object is still alive
```

**Why?**  
Even though we set `user = null`, the object is still reachable through `piyush`. The garbage collector will not free the object because there is still a live reference.

---

### 4. Method Shorthand – Best Practice

The method shorthand is the most modern and readable way to define methods:

```javascript
const dogesh = {
  name: "Husky",
  dance() {
    console.log("Ichu Inchu Song");
  },
};
```

It is equivalent to the older `dance: function() { ... }`, but cleaner.

---

### 5. Summary of Key Concepts

| Concept                | Explanation                                                                                                    |
| ---------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Method**             | A function that is a property of an object.                                                                    |
| **Method definition**  | Use function expression, named function, or method shorthand (`method() {}`).                                  |
| **`this` in methods**  | Refers to the object that owns the method (the one before the dot).                                            |
| **Object references**  | Objects are stored by reference; assigning an object to another variable copies the reference, not the object. |
| **Garbage Collection** | An object stays alive as long as it is reachable from a root (global, stack) via at least one reference.       |

---

## Complete Code Example with Explanations

```javascript
// 1. Different ways to define a method
function viralDance() {
  console.log("Ichu Inchu Song");
}

const dogesh = {
  name: "Husky",
  dance: viralDance, // method referencing an external function
};

const dogesh2 = {
  name: "Husky",
  dance: function () {
    // method using function expression
    console.log("Ichu Inchu Song");
  },
};

const dogesh3 = {
  name: "Husky",
  dance() {
    // method shorthand (ES6)
    console.log("Ichu Inchu Song");
  },
};

dogesh.dance(); // "Ichu Inchu Song"
dogesh2.dance(); // "Ichu Inchu Song"
dogesh3.dance(); // "Ichu Inchu Song"

// 2. Using `this` inside a method
let user = {
  name: "Piyush Garg",
  age: 26,
  college: "Chitkara University",
  passout: 2021,
  gf: "Mai Ka Ladli",

  intro() {
    console.log(`Hi, my name is ${this.name}!`);
    console.log(`I am ${this.age} years old.`);
    console.log(
      `I studied at ${this.college} and passed out in ${this.passout}.`,
    );
    console.log(`My girlfriend's name is ${this.gf}.`);
  },
};

user.intro();

// 3. Object references and garbage collection
let piyush = user; // piyush now references the same object
user = null; // user no longer points to the object

// The object is still reachable via piyush, so it is not garbage collected.
piyush.intro(); // Works fine – all data is still accessible
```

---

## Interview‑Ready Questions

1. **What is a method in JavaScript?**  
   A method is a function that is a property of an object.

2. **What is the difference between a regular function and a method?**  
   A method is called in the context of an object (`obj.method()`), which sets `this` to that object.

3. **Why does `this` inside a method refer to the object?**  
   Because the method is called as a property lookup (`obj.method()`), JavaScript binds `this` to the object before the dot.

4. **What happens if you assign a method to a variable and call it?**  
   You lose the implicit binding; `this` becomes the global object (or `undefined` in strict mode).

5. **How does object reference work in the code `let piyush = user; user = null;`?**  
   `piyush` and `user` are two references to the same object. Setting `user = null` only removes that reference; the object remains because `piyush` still points to it.

6. **Can a method be defined with arrow functions?**  
   Yes, but arrow functions do not have their own `this`; they inherit `this` from the outer scope, which is usually not desirable for methods.

---
