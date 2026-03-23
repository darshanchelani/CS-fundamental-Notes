## Overview

Let's understand two distinct object creation patterns in JavaScript:

1. **Constructor function with prototype** – `TataCar` is a constructor function; instances are created using the `new` keyword. Methods are shared via the `prototype` property.
2. **Factory function** – `createAutoRickshaw` is a factory function; it returns a plain object literal with its own methods. No `new` is used, and each instance has its own copy of the method.

The code logs properties and method outputs for instances of both patterns, illustrating how they work.

---

## Step‑by‑Step Analysis

### 1. Constructor Function with Prototype

```javascript
function TataCar(chassisNumber, modelName) {
  this.chassisNumber = chassisNumber;
  this.modelName = modelName;
  this.fuelLevel = 100;
}

TataCar.prototype.status = function () {
  return `Tata ${this.modelName} #${this.chassisNumber} | Fuel: ${this.fuelLevel}`;
};

const car1 = new TataCar("MH-101", "Nexon");
const car2 = new TataCar("DL-202", "Harrier");

console.log(car1.modelName);
console.log(car2.modelName);
console.log(car1.status());
console.log(car2.status());
```

- **Constructor function:** `TataCar` is a regular function intended to be called with `new`. When `new` is used:
  1. A new empty object is created.
  2. The object’s prototype is set to `TataCar.prototype`.
  3. The constructor is executed with `this` bound to the new object.
  4. The new object is returned (unless the constructor explicitly returns another object).

- **Adding to prototype:** `TataCar.prototype.status = function() {...}` adds a `status` method to the prototype. All instances share this same method (memory efficient).

- **Instance properties:** Each car has its own `chassisNumber`, `modelName`, and `fuelLevel` (set in the constructor).

**Console output:**

- `car1.modelName` → `"Nexon"`
- `car2.modelName` → `"Harrier"`
- `car1.status()` → `"Tata Nexon #MH-101 | Fuel: 100"`
- `car2.status()` → `"Tata Harrier #DL-202 | Fuel: 100"`

**Why these outputs:** The `status` method uses `this` to refer to the current instance, accessing its own properties.

### 2. Factory Function

```javascript
function createAutoRickshaw(id, route) {
  return {
    id,
    route,
    run() {
      return `Auto ${this.id} running on ${this.route}`;
    },
  };
}

const auto1 = createAutoRickshaw("UP-1", "Lucknow-kanpu");
const auto2 = createAutoRickshaw("UP-2", "Agra-Mathura");

console.log(auto1.run());
console.log(auto2.run());
```

- **Factory function:** `createAutoRickshaw` returns a new object literal each time it is called.
- **Method definition:** The `run` method is defined inside the returned object literal. Each object gets its **own copy** of the method (no shared prototype).
- **Property shorthand:** `id` and `route` are property value shorthands (ES6) equivalent to `id: id, route: route`.

**Console output:**

- `auto1.run()` → `"Auto UP-1 running on Lucknow-kanpu"`
- `auto2.run()` → `"Auto UP-2 running on Agra-Mathura"`

**Why these outputs:** Each object has its own `id` and `route`, and the `run` method uses `this` to refer to the object itself.

---

## Key Concepts Explained

### Constructor Function + Prototype

- **`new` operator** – creates an instance and sets up the prototype chain.
- **Prototype** – a mechanism for sharing methods across instances. Methods are stored once on the constructor’s `prototype` and are inherited by all instances.
- **`instanceof`** – `car1 instanceof TataCar` returns `true`.
- **Memory efficiency** – methods are shared, not duplicated.
- **Dynamic updates** – adding to the prototype after instance creation is reflected in existing instances.

### Factory Function

- **No `new`** – just a regular function that returns an object.
- **No prototype sharing** – each instance has its own copy of methods (though they could be moved to a shared object if desired).
- **Flexibility** – can return different types of objects, include private variables via closure, or conditionally construct objects.
- **`instanceof`** – `auto1 instanceof createAutoRickshaw` returns `false` (it's a plain object).

### Differences in a Nutshell

| Feature              | Constructor + Prototype             | Factory Function                    |
| -------------------- | ----------------------------------- | ----------------------------------- |
| Creation             | `new Constructor()`                 | `factory()`                         |
| Method location      | On `Constructor.prototype` (shared) | On each instance (own copy)         |
| Memory usage         | Lower (shared methods)              | Higher (each instance gets its own) |
| `instanceof`         | Works                               | Does not work (returns false)       |
| Ability to use `new` | Required for correct `this`         | Not needed                          |
| Private data         | Can use closures (less common)      | Easily achieved via closure         |

---

---

## Interview Discussion

### Common Questions

1. **What is the difference between a constructor function and a factory function?**
   - Constructor functions are invoked with `new`, set properties on `this`, and typically add methods to `prototype`. Factory functions return a new object directly, without `new`.

2. **Why would you use a prototype instead of defining methods inside the constructor?**
   - To share methods across instances, saving memory and allowing dynamic addition of methods.

3. **Can you use `instanceof` with factory‑created objects?**
   - No, because they are plain objects, not instances of the factory function.

4. **What are the advantages of factory functions?**
   - Flexibility: can return different object shapes, hide private data using closures, avoid the need for `new` (and the risk of forgetting it), and work well with composition.

5. **What is the `new` keyword doing internally?**
   - Creates a new object, sets its prototype, binds `this`, and returns it.

### Best Practices

- Use **constructor + prototype** when you need many instances of a “class” with shared methods, especially if you’re building a library or framework.
- Use **factory functions** for simpler object creation, when you need private variables, or when you want to avoid the `new` boilerplate.
- In modern JavaScript, `class` syntax is syntactic sugar over the constructor + prototype pattern, offering a cleaner way to achieve the same.

### Follow‑Up Questions

- **How would you implement a shared method in a factory function?**  
  You can define the method on a separate object and use `Object.create` or `Object.assign` to mix it in, or return an object with a method that references a shared function via closure.

- **What happens if you call a constructor without `new`?**  
  In non‑strict mode, `this` will be the global object, polluting it. In strict mode, `this` is `undefined`, causing an error. Using `new` is essential.

- **How can you detect if a function was called with `new`?**  
  Use `new.target` (ES6). Inside a function, `new.target` is `undefined` if called without `new`.

- **Why does `car1.status()` work even though `status` is not defined on `car1` itself?**  
  Because JavaScript looks up the prototype chain: `car1` → `TataCar.prototype` → finds `status`.

---
