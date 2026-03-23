## Overview

Let's understand the use of **explicit function binding** methods: `call`, `apply`, and `bind`. It shows how these methods allow you to manually set the `this` value inside a function and, in the case of `bind`, create a new function with a permanently bound `this`. The code also touches on a classic use of `apply` with `Math.max` to find the maximum number in an array, and contrasts it with the modern spread operator.

The script uses a cooking and delivery theme to illustrate these concepts.

---

## Step‑by‑Step Analysis

### 1. Base Function and Default Binding

```javascript
function cookDish(ingredient, style) {
  return `${this.name} prepares ${ingredient} in ${style} style !`;
}
console.log(cookDish());
```

- **Concept:** `cookDish` expects `this.name` to exist. Called directly (without a context), `this` is the global object (in non‑strict mode). Since the global object usually has no `name` property, `this.name` becomes `undefined`.
- **Output:** `"undefined prepares undefined in undefined style !"`

### 2. Objects to Serve as `this`

```javascript
const sharmaKitchen = { name: "Sharma jis Kitchen" };
const guptaKitchen = { name: "Gupta jis Kitchen" };
```

- Simple objects with a `name` property – will be used as explicit `this` values.

### 3. `call` – Invoke Immediately with Individual Arguments

```javascript
console.log(cookDish.call(sharmaKitchen, "Paneer and spices", "Muglai"));
```

- **Concept:** `call` invokes `cookDish` with `this` set to `sharmaKitchen`. Arguments are passed one by one.
- **Output:** `"Sharma jis Kitchen prepares Paneer and spices in Muglai style !"`

### 4. `apply` – Invoke Immediately with Array of Arguments

```javascript
const guptaOrder = ["Chole kulche", "Punjabi Dhaba"];
console.log(cookDish.apply(guptaKitchen, guptaOrder));
```

- **Concept:** `apply` is similar to `call`, but takes an array of arguments.
- **Output:** `"Gupta jis Kitchen prepares Chole kulche in Punjabi Dhaba style !"`

### 5. `Math.max.apply` and Spread Operator

```javascript
const bills = [100, 30, 45, 50];
Math.max.apply(null, bills);
Math.max(...bills);
```

- **Concept:** `Math.max` expects individual numbers. `apply` spreads the array into arguments. The first argument (`null`) is ignored because `Math.max` doesn’t use `this`. Modern JavaScript can achieve the same with the spread operator `...`.
- **No `console.log`** – just demonstrates alternative syntax.

### 6. Another Function for Delivery Reports

```javascript
function reportDelivery(location, status) {
  return `${this.name} at ${location}: ${status}`;
}
const deliveryBoy = { name: "Ranveer" };
```

- Another function that uses `this.name`. The object `deliveryBoy` will serve as context.

### 7. `call`, `apply`, and `bind` Compared

```javascript
console.log("Call: ", reportDelivery.call(deliveryBoy, "Lyari", "Ordered"));
console.log("Apply: ", reportDelivery.apply(deliveryBoy, ["Mars", "Pick up"]));
console.log("Bind: ", reportDelivery.bind(deliveryBoy, "Haridwar", "WHAT"));
```

- **`call`** – invokes immediately, arguments individually → `"Ranveer at Lyari: Ordered"`
- **`apply`** – invokes immediately, arguments as array → `"Ranveer at Mars: Pick up"`
- **`bind`** – returns a **new function** with `this` bound to `deliveryBoy` and the first two arguments pre‑set (`"Haridwar"`, `"WHAT"`). The function is **not invoked** yet, so the `console.log` prints the function itself, not its result. In most environments, you’ll see something like `[Function: bound reportDelivery]`.

### 8. Using a Bound Function

```javascript
const bindReport = reportDelivery.bind(deliveryBoy);
console.log(bindReport("Haridwar", "WHAT"));
```

- **Concept:** `bind` without pre‑set arguments creates a new function with `this` fixed to `deliveryBoy`. Then we call it with arguments `"Haridwar"` and `"WHAT"`.
- **Output:** `"Ranveer at Haridwar: WHAT"`

---

## Key Concepts Explained

### `call`

- Invokes the function immediately.
- First argument: value to use as `this`.
- Subsequent arguments: passed to the function individually.

### `apply`

- Same as `call`, but arguments are passed as an array (or array‑like object).
- Often used when arguments are already in an array, or to spread array elements (as in `Math.max.apply(null, arr)`).

### `bind`

- Returns a **new function** with `this` permanently bound to the provided value.
- Optionally, you can pre‑set arguments (partial application).
- The bound function can be called later, and its `this` cannot be changed.

### `Math.max.apply` and Spread

- `Math.max.apply(null, arr)` uses `apply` to pass array elements as arguments.
- Modern alternative: `Math.max(...arr)` using the spread operator.

### `this` in Explicit Binding

- When using `call`/`apply`/`bind`, the first argument becomes the `this` value inside the function, overriding default binding rules.

---

## Interview Discussion

### Common Questions

1. **What is the difference between `call`, `apply`, and `bind`?**
   - `call` and `apply` invoke the function immediately; `bind` returns a new function.
   - `call` takes arguments individually; `apply` takes an array of arguments.
   - `bind` allows pre‑setting arguments (partial application).

2. **When would you use `bind` instead of `call`/`apply`?**
   - When you need a function with a fixed `this` that you can call later (e.g., event handlers, callbacks).

3. **What happens if you pass `null` or `undefined` as the first argument to `call`/`apply`/`bind`?**
   - In non‑strict mode, `this` becomes the global object. In strict mode, it remains `null` or `undefined`.

4. **How can you find the maximum value in an array without using `Math.max`?**
   - `Math.max.apply(null, arr)` or `Math.max(...arr)` are the idiomatic ways.

5. **Can you use `bind` to pre‑set arguments? Give an example.**
   - `const greet = (greeting, name) => greeting + name;`  
     `const sayHello = greet.bind(null, "Hello ");`  
     `sayHello("Alice");` → `"Hello Alice"`

### Best Practices

- Use `bind` when you need to fix `this` for a function that will be called later (e.g., as a callback).
- Prefer the spread operator (`...`) over `apply` for spreading arrays, unless you need to support older environments.
- Be cautious with `call`/`apply` on methods that rely on `this`; ensure you pass the correct context.
- In strict mode, passing `null`/`undefined` does **not** default to the global object, so you may need to pass a dummy object if the function doesn’t use `this`.

### Follow‑Up Questions

- **What is the output of `Math.max.apply(null, [])`?**  
  `-Infinity`, because an empty array yields no arguments and `Math.max` returns `-Infinity`.

- **How would you implement `bind` manually?**  
  You could use a closure that stores the function, context, and pre‑set arguments, and returns a function that uses `apply` to combine pre‑set and runtime arguments.

- **What happens if you bind a method to an object and then call it with `call`?**  
  The `bind` has already permanently set `this`; `call` cannot override it.

---
