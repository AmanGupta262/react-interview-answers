# JavaScript Interview Answers (Frontend, 4+ Years)

## Part 1: Execution Context & Hoisting

### 1. How does JavaScript execute code?

**Short answer:** JavaScript is synchronous and single-threaded: one call stack, one line at a time. Code runs inside execution contexts, each created in two phases: memory creation, then code execution.
**Real-life example:** A single chef working through one order ticket at a time; if a dish needs a sub-recipe, the chef pauses the ticket, finishes the sub-recipe, then resumes.
- The engine first creates a global execution context (GEC) holding two parts: Memory (variable environment) and Thread of Execution.
- Phase 1, memory creation: `var` variables get `undefined`, function declarations get the whole function body, `let`/`const` are registered but left uninitialised (temporal dead zone).
- Phase 2, code execution: lines run top to bottom and variables get their real values.
- Every function call creates a new execution context that repeats both phases; when the function returns, that context is popped and destroyed.
- The contexts are managed by the call stack (LIFO); too many nested calls without returning gives `RangeError: Maximum call stack size exceeded`.
- Timers, network and DOM events are handled by the host (browser or Node), not by the single JS thread.
```js
var n = 2;
function square(num) {
  var ans = num * num;
  return ans;
}
var sq2 = square(n);
var sq4 = square(4);
console.log(sq2, sq4);
// Stack over time: [GEC] -> [GEC, square(2)] -> [GEC] -> [GEC, square(4)] -> [GEC]
```
**Output:** `4 16`
**Interview tip:** Mention that `this` and the scope chain are also set up when an execution context is created, and be ready to draw the stack for a nested call.

### 2. What is the event loop, and in what order do microtasks and macrotasks run?

**Short answer:** The event loop pushes queued callbacks onto the call stack only when it is empty. After every macrotask (script, `setTimeout`, I/O, UI event) the whole microtask queue (promise callbacks, `queueMicrotask`) is drained before the next macrotask.
**Real-life example:** A bank counter: the teller serves the current customer fully, then clears all quick "urgent" slips (microtasks) before calling the next person in the regular queue (macrotasks).
- Synchronous code always finishes first because the stack must be empty before anything from a queue runs.
- `then`/`catch`/`finally` callbacks and `await` continuations are microtasks.
- `setTimeout(fn, 0)` is a macrotask with a minimum delay, so it runs after all pending microtasks.
- A microtask that keeps queueing more microtasks can starve rendering and timers.
```js
console.log("start");
setTimeout(() => console.log("timeout"), 0);
Promise.resolve().then(() => console.log("promise"));
console.log("end");
```
**Output:** `start`, `end`, `promise`, `timeout`
**Interview tip:** Browsers render between macrotasks (not between microtasks), and `setTimeout(fn, 0)` is clamped to about 4ms after 5 nested calls.

### 3. What is hoisting?

**Short answer:** Hoisting is the effect of the memory creation phase: declarations are registered before any code runs, so `var` variables read as `undefined` and function declarations can be called before they appear in the code.
- Nothing physically moves; the engine simply allocates memory for declarations first.
- Function declarations are hoisted with their full body, so they work before the line they are written on.
- `var` is hoisted and initialised to `undefined`; only the declaration is hoisted, never the assignment.
```js
getName();
console.log(x);
console.log(getName);

var x = 10;
function getName() {
  console.log("name");
}
```
**Output:** `name`, `undefined`, then the function itself (`[Function: getName]` in Node, its source in a browser console)
**Interview tip:** Hoisting is per scope: `var` is hoisted to the top of its function, `let`/`const` to the top of their block.

### 4. Do function expressions, arrow functions and classes hoist?

**Short answer:** The variable name is hoisted but the function value is not. With `var` the name is `undefined` so calling it throws `TypeError`; with `let`/`const`/`class` it is in the temporal dead zone so touching it throws `ReferenceError`.
- Only function declarations are hoisted with their body; `var f = () => {}` hoists only `var f` as `undefined`.
- Correction to a common claim: an arrow function assigned with `const` does not get `undefined`, it stays uninitialised until its line runs.
- Classes behave like `let`: hoisted but in the TDZ.
```js
try { early(); } catch (e) { console.log(e.name + ": " + e.message); }
var early = () => console.log("hi");

try { late(); } catch (e) { console.log(e.name + ": " + e.message); }
const late = () => console.log("hi");
```
**Output:** `TypeError: early is not a function`, `ReferenceError: Cannot access 'late' before initialization`
**Interview tip:** The two error types are the giveaway: `TypeError` means the binding exists with `undefined`, `ReferenceError` means TDZ or no binding at all.

### 5. What is the temporal dead zone (TDZ)?

**Short answer:** The TDZ is the time between entering a scope (when `let`/`const`/`class` bindings are hoisted) and the line that initialises them; any access in that window throws a `ReferenceError`.
- `let` and `const` are hoisted, otherwise the inner `let` in a block would not shadow an outer variable of the same name.
- That is why the TDZ error says "Cannot access 'x' before initialization" rather than "x is not defined".
- Even `typeof` throws inside the TDZ, while `typeof` on a never-declared name safely returns `"undefined"`.
- Default parameter expressions also have a TDZ (`function f(a = b, b) {}` throws).
```js
let v = "outer";
{
  try { console.log(v); } catch (e) { console.log(e.message); }
  let v = "inner";
}
try { typeof t; } catch (e) { console.log(e.message); }
let t = 1;
console.log(typeof neverDeclared);
```
**Output:** `Cannot access 'v' before initialization`, `Cannot access 't' before initialization`, `undefined`
**Interview tip:** The TDZ is temporal (about when code runs), not spatial (where the code sits), so a function that reads a `let` declared below it works fine if it is called after initialisation.

### 6. What is the difference between var, let and const?

**Short answer:** `var` is function-scoped, hoisted as `undefined`, redeclarable and attaches to the global object; `let` and `const` are block-scoped, live in the TDZ until initialised and cannot be redeclared; `const` also cannot be reassigned.

| | var | let | const |
|---|---|---|---|
| Scope | function (or global) | block | block |
| Hoisting | yes, initialised to `undefined` | yes, TDZ | yes, TDZ |
| Redeclare in same scope | allowed | SyntaxError | SyntaxError |
| Reassign | allowed | allowed | TypeError |
| Initialiser required | no | no | yes (`SyntaxError: Missing initializer in const declaration`) |
| Becomes `window` property (global scope) | yes | no | no |

- `const` protects the binding, not the value: object contents can still be mutated unless frozen with `Object.freeze`.
- Redeclaration and missing-initialiser errors are early SyntaxErrors, so the whole script fails before any line runs.
- `let a = 10; var a = 100;` and `var a = 10; let a = 100;` are both SyntaxErrors.
```js
var a = 1;
var a = 2;
console.log(a);

const c = { n: 1 };
c.n = 2;
console.log(c.n);
try { c = {}; } catch (e) { console.log(e.name + ": " + e.message); }
```
**Output:** `2`, `2`, `TypeError: Assignment to constant variable.`
**Interview tip:** Default to `const`, use `let` when you must reassign, and avoid `var`; the per-iteration binding of `let` in loops is the practical reason it matters.

## Part 2: Scope & Closures

### 7. What are scope, lexical environment and the scope chain?

**Short answer:** Scope decides where a variable is accessible. A lexical environment is a function's local memory plus a reference to its parent's lexical environment, and linking these gives the scope chain used for variable lookup.
- Lexical means "where the function is written in the source", not where it is called from.
- Lookup starts in the current environment and walks outwards through parents until the global scope; failing that you get `ReferenceError`.
- Scope is fixed at write time (static), which is what makes closures possible.
- Kinds of scope: global, function, block (`{}` with `let`/`const`/`class`), and module.
```js
const g = "global";
function outer() {
  const o = "outer";
  function inner() {
    const i = "inner";
    console.log(i, o, g);
  }
  inner();
}
outer();
```
**Output:** `inner outer global`
**Interview tip:** Lookups resolve only inward-to-outward, never the reverse, so an outer scope can never see an inner variable.

### 8. How does block scope work, and what is shadowing (including illegal shadowing)?

**Short answer:** A block is code in curly braces; `let`, `const` and `class` declared inside stay inside, while `var` ignores blocks. Shadowing is declaring a same-named variable in an inner scope; it is legal unless a `var` would clash with an outer `let`/`const` in the same function.
- A block-scoped `let b` in `{}` is a separate variable from the outer `b`, so changing it leaves the outer one untouched.
- `var a` inside a block is not shadowing at all: it is the same function/global `a`, so it overwrites the outer value.
- Illegal shadowing: a `var` inside a block conflicts with an enclosing `let`/`const` in the same function because `var` hoists up into that scope.
- Shadowing `let` with `var` inside a different function is legal because the `var` stops at the function boundary.
```js
var a = 100;
let b = 100;
const c = 100;
{
  var a = 10;
  let b = 20;
  const c = 30;
  console.log(a, b, c);
}
console.log(a, b, c);
```
**Output:** `10 20 30`, then `10 100 100`
```js
try { new Function("let a = 10; { var a = 100; }"); }
catch (e) { console.log(e.name + ": " + e.message); }
new Function("var a = 10; { let a = 100; }");
console.log("var outside, let inside is fine");
```
**Output:** `SyntaxError: Identifier 'a' has already been declared`, `var outside, let inside is fine`
**Interview tip:** Block scope for functions in sloppy mode is quirky (Annex B hoisting); use function expressions or strict mode in blocks.

### 9. What is a closure?

**Short answer:** A closure is a function bundled together with a reference to the lexical environment where it was created, so it can keep using outer variables even after the outer function has returned.
**Real-life example:** A backpack: when the inner function leaves its home function, it carries the variables it needs with it.
- Closures capture variables by reference, not by value, so later changes to the variable are visible inside the closure.
- Technically every function is a closure, but the word matters when the inner function outlives its creator.
- Engines keep alive only the outer variables the inner function actually references, so unused outer variables can be freed.
```js
function makeGreeter(greeting) {
  return function (name) {
    return greeting + ", " + name;
  };
}
const hi = makeGreeter("Hi");
console.log(hi("Asha"));
```
**Output:** `Hi, Asha`
**Interview tip:** Say "function plus lexical scope" and then immediately give a use case (private state, memoize, once); the definition alone sounds memorised.

### 10. Why does a var loop with setTimeout print the same number, and how do you fix it?

**Short answer:** `var` is function-scoped, so every callback closes over the same single `i`; by the time the timers fire the loop has finished and `i` is `6`. `let` creates a new binding per iteration, so each closure sees its own value.
- With `var` there is one shared `i`; the loop ends when `i` becomes 6, and all five callbacks read that 6.
- With `let`, each iteration gets a fresh copy of `i` (copied from the previous iteration), so each callback captures a different binding.
- Pre-ES6 fix: wrap the call in an IIFE (or a helper function) that receives `i` as a parameter, giving each callback its own variable `j`.
- Another fix: pass an argument to the timer, `setTimeout(console.log, i * 10, i)`.
```js
for (var i = 1; i <= 5; i++) {
  setTimeout(function () { console.log(i); }, i * 10);
}
```
**Output:** `6` printed five times
```js
for (let i = 1; i <= 5; i++) {
  setTimeout(function () { console.log(i); }, i * 10);
}
```
**Output:** `1 2 3 4 5` (one per line)
```js
for (var i = 1; i <= 5; i++) {
  (function (j) {
    setTimeout(function () { console.log(j); }, j * 10);
  })(i);
}
```
**Output:** `1 2 3 4 5` (one per line)
**Interview tip:** If asked to fix it without `let`, reach for the IIFE or the extra `setTimeout` argument; mention that the helper-function version only needs `var` because the parameter creates the new scope.

### 11. What are practical uses of closures?

**Short answer:** Data privacy (module and factory patterns), stateful functions like counters, `once`, memoization, partial application/currying, and event handlers or callbacks that remember their setup data.
- Encapsulation: variables inside the outer function are unreachable except through the returned functions.
- Function factories: `makeMultiplier(2)` returns a function preconfigured with its multiplier.
- Memoization and `once` keep a cache or flag in the closure between calls.
- Debounce, throttle and React hooks (stale closure bugs) all rely on closures.
- Downside: a long-lived closure keeps everything it references alive, which can leak large objects or DOM nodes if never released.
```js
function createCounter() {
  let count = 0;
  return {
    increment() { return ++count; },
    getCount() { return count; },
  };
}
const counter = createCounter();
counter.increment();
counter.increment();
console.log(counter.getCount(), counter.count);

function once(fn) {
  let done = false, result;
  return (...args) => {
    if (!done) { done = true; result = fn(...args); }
    return result;
  };
}
const init = once(() => { console.log("init ran"); return 42; });
console.log(init(), init());
```
**Output:** `2 undefined`, `init ran`, `42 42`
**Interview tip:** Bring up stale closures in React (`useEffect` reading old state) to show you have hit the downside in real code.

### 12. What is the difference between a function declaration, function expression, anonymous function and named function expression?

**Short answer:** A declaration is a statement that is hoisted with its body; an expression is a function used as a value and is not hoisted; an anonymous function has no name; a named function expression has a name visible only inside itself.
- Declaration (statement): `function x() {}`, hoisted, callable before its line.
- Expression: `const x = function () {}`, behaves like the variable it is assigned to.
- Anonymous function: a bare `function () {}` as a statement is a SyntaxError ("Function statements require a function name"); it is valid only where a value is expected (assignment, argument, return, IIFE).
- Named function expression: `const x = function xyz() {}`; `xyz` exists only inside the function body (useful for recursion and stack traces), not outside.
- Parameters are the names in the definition; arguments are the values passed in the call.
- First-class functions: functions are values that can be assigned, passed as arguments and returned, which is what makes callbacks and closures work.
```js
function decl() { return "decl"; }
const expr = function () { return "expr"; };
const nfe = function inner() { return typeof inner; };
console.log(decl(), expr(), nfe());
try { inner(); } catch (e) { console.log(e.name + ": " + e.message); }
try { new Function("function () {}"); } catch (e) { console.log(e.name + ": " + e.message); }
```
**Output:** `decl expr function`, `ReferenceError: inner is not defined`, `SyntaxError: Function statements require a function name`
**Interview tip:** Mention that higher-order functions (map, filter, debounce) exist only because functions are first-class.

### 13. What is a factory function, and how does it differ from a constructor function?

**Short answer:** A factory function is an ordinary function that returns a new object each call and needs no `new`; unlike a constructor it does not rely on `this` or prototypes, and variables inside it become truly private through closures.
- No `new` keyword, so no risk of forgetting it and getting `this` bound to the global object.
- Private state: variables in the factory are visible only to the methods defined inside it.
- Trade-off: each object gets its own copy of every method, while constructor/class methods live once on the prototype and save memory.
- Factories cannot be used with `instanceof` because the result is a plain object.
```js
function createPerson(name, age) {
  return {
    name,
    age,
    greet() { return `Hello, my name is ${this.name} and I am ${this.age} years old.`; },
  };
}
console.log(createPerson("Alice", 30).greet());

function createCounter() {
  let count = 0;
  return {
    increment() { count++; return count; },
    decrement() { count--; return count; },
    getCount() { return count; },
  };
}
const counter = createCounter();
counter.increment();
console.log(counter.increment(), counter.getCount(), counter.count);
console.log(counter.decrement());
```
**Output:** `Hello, my name is Alice and I am 30 years old.`, `2 2 undefined`, `1`
**Interview tip:** Hard-private `#fields` in classes now give privacy without per-instance methods, so choose factories for composition, classes for many instances.

## Part 3: Types and Coercion

### 14. What is the difference between primitives and objects in JavaScript?
**Short answer:** Primitives are immutable values copied by value; objects (including arrays and functions) are mutable and copied by reference.
**Real-life example:** A primitive is a photocopy of a page; an object is a shared Google Doc link.
- There are 7 primitives: string, number, bigint, boolean, undefined, null, symbol.
- Assigning or passing a primitive copies the value, so changes never leak back.
- Assigning or passing an object copies the reference, so two variables point at the same data.
- `===` on objects compares references, not contents.
- Primitives have no properties; `"abc".length` works because JS temporarily wraps it in a String object.
```js
let a = 5; let b = a; b++;
console.log(a, b);
const o1 = { n: 1 }; const o2 = o1; o2.n = 99;
console.log(o1.n, { n: 1 } === { n: 1 });
```
**Output:** `5 6` then `99 false`
**Interview tip:** Mention that `const` freezes the binding, not the object, because interviewers often follow up with that.

### 15. What is the difference between == and ===, and when would you use ==?
**Short answer:** `===` compares without conversion; `==` converts types first using the abstract equality rules.
- `===` returns false immediately if types differ (and treats `NaN !== NaN`).
- `==` coerces: `"5" == 5` is true, `0 == false` is true, `null == undefined` is true.
- `null == undefined` is the only `==` use many style guides allow (`x == null` checks both).
- Objects are compared by reference with both operators; `==` only converts an object when the other side is a primitive.
- `Object.is` differs from `===` for `NaN` (equal) and `+0/-0` (not equal).
```js
console.log("5" == 5, 0 == false, null == undefined, null == 0, [] == false);
console.log("5" === 5, NaN === NaN, Object.is(NaN, NaN), Object.is(0, -0));
```
**Output:** `true true true false true` then `false false true false`
**Interview tip:** Say you default to `===` and explain the single `== null` exception.

### 16. What are the quirks of typeof?
**Short answer:** `typeof null` is `"object"` (a legacy bug), arrays are `"object"`, and functions are `"function"`; undeclared variables return `"undefined"` instead of throwing.
- Use `Array.isArray` for arrays and `x === null` for null.
- `typeof NaN` is `"number"`.
- `typeof` on a let/const in its temporal dead zone throws a ReferenceError.
- For precise tags use `Object.prototype.toString.call(x)`.
```js
console.log(typeof null, typeof [], typeof NaN, typeof function () {}, typeof undeclaredVar);
console.log(typeof class {}, typeof Symbol(), typeof 10n);
console.log(Object.prototype.toString.call([]), Object.prototype.toString.call(null));
```
**Output:** `object object number function undefined` then `function symbol bigint` then `[object Array] [object Null]`
**Interview tip:** Name the null bug and its safe replacement in one breath.

### 17. What is the difference between null and undefined?
**Short answer:** `undefined` means a value was never assigned; `null` is an explicit "no value" set by the developer.
- Uninitialised variables, missing properties, and functions without return give `undefined`.
- `null` is assigned deliberately, for example to clear a reference.
- `typeof` gives `"undefined"` vs `"object"`; both are falsy and `null == undefined`.
- JSON.stringify drops `undefined` properties but keeps `null`.
- Default parameters trigger only for `undefined`, not `null`.
```js
function f(x = "default") { return x; }
console.log(f(undefined), f(null));
console.log(JSON.stringify({ a: undefined, b: null }));
```
**Output:** `default null` then `{"b":null}`
**Interview tip:** The default-parameter and JSON differences show real-world awareness.

### 18. What is NaN and how do you check for it reliably?
**Short answer:** NaN is the number-type result of an invalid numeric operation and is the only value not equal to itself; use `Number.isNaN`.
- Global `isNaN` coerces its argument first, so `isNaN("abc")` is true.
- `Number.isNaN` does not coerce and is true only for the real NaN.
- `Object.is(x, NaN)` and `x !== x` also work.
- `[NaN].includes(NaN)` is true but `[NaN].indexOf(NaN)` is -1.
```js
console.log(isNaN("abc"), Number.isNaN("abc"), Number.isNaN(NaN));
console.log([NaN].includes(NaN), [NaN].indexOf(NaN));
console.log(0 / 0, parseInt("x"), Math.sqrt(-1));
```
**Output:** `true false true` then `true -1` then `NaN NaN NaN`
**Interview tip:** Contrast global isNaN with Number.isNaN; that is the actual question being tested.

### 19. What are truthy and falsy values?
**Short answer:** Falsy values are `false, 0, -0, 0n, "", null, undefined, NaN`; everything else is truthy, including `[]`, `{}`, and `"0"`.
- Conditions, `!`, `&&`, `||`, and ternaries coerce to boolean.
- `[]` and `{}` are truthy even though `[] == false` is true (the two coercion paths differ).
- `||` returns the first truthy operand; `&&` returns the first falsy one.
- Use `??` when 0 or "" are valid values.
```js
const falsy = [false, 0, -0, 0n, "", null, undefined, NaN];
console.log(falsy.every((v) => !v));
console.log(!!"0", !![], !!{}, !!"false");
console.log(0 || "fallback", 0 ?? "fallback");
```
**Output:** `true` then `true true true true` then `fallback 0`
**Interview tip:** Bring up the `||` vs `??` bug with a 0 quantity or empty string.

### 20. How does implicit type coercion work? Explain a few classic puzzles.
**Short answer:** With `+`, if either operand is a string it concatenates; other arithmetic operators convert to numbers; objects go through `valueOf`/`toString` first.
- `+` with a string operand concatenates; `-`, `*`, `/` always coerce to number.
- Unary `+` is the shortest string-to-number conversion.
- `[]` becomes `""` and `{}` becomes `"[object Object]"` when converted to a primitive.
- `null` becomes 0, `undefined` becomes NaN, `true` becomes 1.
```js
console.log(1 + "2", "3" - 1, "3" * "4", +"", +"abc");
console.log([] + [], [] + {}, [1, 2] + [3]);
console.log(true + true, null + 1, undefined + 1);
```
**Output:** `12 2 12 0 NaN` then ` [object Object] 1,23` then `2 1 NaN`
**Interview tip:** Walk through the steps (ToPrimitive, then string or number) instead of memorising results.

### 21. Why does 0.1 + 0.2 !== 0.3 and how do you handle money or precision?
**Short answer:** Numbers are IEEE 754 doubles, which cannot represent most decimal fractions exactly, so tiny rounding errors appear.
- Compare with a tolerance such as `Math.abs(a - b) < Number.EPSILON`.
- For money, store integer minor units (paise or cents) and convert only for display.
- Use `toFixed` only for display; it returns a string.
- Use `BigInt` for integers beyond `Number.MAX_SAFE_INTEGER` (2^53 - 1).
```js
console.log(0.1 + 0.2, 0.1 + 0.2 === 0.3);
console.log(Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON);
console.log((1999 + 1) / 100, 9007199254740993n + 1n, Number.MAX_SAFE_INTEGER + 2);
```
**Output:** `0.30000000000000004 false` then `true` then `20 9007199254740994n 9007199254740992`
**Interview tip:** Say "integers in minor units" for money; it is the answer interviewers want.

## Part 4: Language Essentials

### 22. What is the difference between a shallow copy and a deep copy?

**Short answer:** A shallow copy duplicates only the top level, so nested objects and arrays are still shared by reference; a deep copy recursively clones everything so the copy shares nothing with the original.
- Shallow copy tools: spread (`{...obj}`, `[...arr]`), `Object.assign`, `Array.from`, `slice`.
- Deep copy tools: `structuredClone(obj)` (modern, handles Date, Map, Set, cycles), `JSON.parse(JSON.stringify(obj))`, or a recursive function.
- The JSON trick drops `undefined`, functions and symbols, turns `Date` into a string, loses `Map`/`Set`/`RegExp`, converts `NaN`/`Infinity` to `null` and throws on circular references.
- `structuredClone` cannot clone functions or DOM nodes (it throws `DataCloneError`).
```js
const obj = { name: "Alice", address: { city: "Wonderland" } };
const shallow = Object.assign({}, obj);
shallow.address.city = "New City";
console.log(obj.address.city);

const arr = [1, 2, { nested: "value" }];
const shallowArr = [...arr];
shallowArr[2].nested = "new value";
console.log(arr[2].nested);

const original = { name: "Alice", address: { city: "Wonderland" }, at: new Date(0) };
const viaJson = JSON.parse(JSON.stringify(original));
const viaClone = structuredClone(original);
viaClone.address.city = "New City";
console.log(original.address.city, typeof viaJson.at, viaClone.at instanceof Date);
```
**Output:** `New City`, `new value`, `Wonderland string true`
**Interview tip:** React state updates rely on shallow copies of the changed path only (immutable updates); deep cloning everything defeats memoization.

### 23. How do you implement a deep clone function?

**Short answer:** Recursively copy arrays and plain objects, return primitives as-is, special-case `Date` and `RegExp`, and keep a `WeakMap` of already-cloned objects so circular references do not recurse forever.
- The version often written first (`typeof obj !== "object"` check, then `for...in`) breaks on cycles, copies inherited enumerable keys, and turns a `Date` into `{}`.
- The `WeakMap` maps each original to its clone, so a second visit returns the clone already being built.
- Production code should usually call `structuredClone` instead of hand-rolling this.
```js
function deepClone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== "object") return value;
  if (value instanceof Date) return new Date(value);
  if (value instanceof RegExp) return new RegExp(value.source, value.flags);
  if (seen.has(value)) return seen.get(value);
  const copy = Array.isArray(value) ? [] : {};
  seen.set(value, copy);
  for (const key of Object.keys(value)) copy[key] = deepClone(value[key], seen);
  return copy;
}

const src = { list: [1, { a: 2 }], when: new Date(0) };
src.self = src;
const dup = deepClone(src);
console.log(dup !== src, dup.list[1] !== src.list[1], dup.self === dup, dup.when instanceof Date);
```
**Output:** `true true true true`
**Interview tip:** Name what your version does not handle (Map, Set, typed arrays, getters, prototypes) before the interviewer asks.

## Part 5: Functions and Objects

### 24. What are the differences between arrow functions and regular functions?
**Short answer:** Arrow functions have no own `this`, `arguments`, `super`, or `prototype`, and cannot be used with `new`.
| Feature | Regular function | Arrow function |
|---|---|---|
| `this` | Set by call site | Inherited lexically |
| `arguments` | Available | Not available (use rest) |
| `new` | Allowed | TypeError |
| `prototype` | Yes | No |
| Good as method | Yes | Usually no |
- Do not use arrows for object methods or prototype methods that need `this`.
- Arrows are ideal for callbacks that must keep the outer `this`.
```js
const obj = {
  name: "obj",
  regular() { return this.name; },
  arrow: () => typeof this,
};
console.log(obj.regular(), obj.arrow());
const arrow = () => {};
console.log(arrow.prototype, typeof function () {}.prototype);
try { new arrow(); } catch (e) { console.log(e.constructor.name); }
```
**Output:** `obj object` then `undefined object` then `TypeError`
**Interview tip:** State "lexical this" and give the callback-in-class-method use case.

### 25. When do you use call, apply, and bind in real code?
**Short answer:** All three set `this` explicitly; call and apply invoke immediately (args list vs array), bind returns a new function with `this` (and optionally args) fixed.
- `call(thisArg, a, b)` and `apply(thisArg, [a, b])` run now.
- `bind(thisArg, a)` returns a function for later (event handlers, partial application).
- Borrowing methods: `Array.prototype.slice.call(arguments)`.
- `Math.max.apply(null, arr)` was the pre-spread idiom.
- A bound function's `this` cannot be changed by a later call or bind.
```js
function greet(greeting, punct) { return `${greeting}, ${this.name}${punct}`; }
const user = { name: "Asha" };
console.log(greet.call(user, "Hi", "!"), greet.apply(user, ["Hello", "?"]));
const hi = greet.bind(user, "Hey");
console.log(hi("."), hi.call({ name: "Other" }, "."));
```
**Output:** `Hi, Asha! Hello, Asha?` then `Hey, Asha. Hey, Asha.`
**Interview tip:** Mention partial application with bind and that re-binding a bound function does nothing.

### 26. What is currying and why is it useful?
**Short answer:** Currying turns `f(a, b, c)` into `f(a)(b)(c)`, so each call takes one argument and returns a function until all are supplied.
- It enables partial application and reusable specialised functions.
- It differs from partial application: currying always returns unary functions, partial fixes some arguments at once.
- Useful in functional pipelines and configuration-style helpers.
```js
const add = (a) => (b) => (c) => a + b + c;
const add5 = add(5);
console.log(add(1)(2)(3), add5(10)(20));
const log = (level) => (msg) => `[${level}] ${msg}`;
console.log(log("warn")("disk low"));
```
**Output:** `6 35` then `[warn] disk low`
**Interview tip:** Follow up by offering to write a generic `curry(fn)`; see the coding section.

### 27. What are higher-order functions?
**Short answer:** Functions that take functions as arguments or return functions; `map`, `filter`, `reduce`, `setTimeout`, and `debounce` are all examples.
- They enable abstraction over behaviour, not just data.
- Functions are first-class values in JS, which makes this possible.
- Prefer them over manual loops for readability.
```js
const times = (n) => (x) => x * n;
const nums = [1, 2, 3, 4];
console.log(nums.map(times(3)).filter((x) => x > 5).reduce((a, b) => a + b, 0));
```
**Output:** `27`
**Interview tip:** Give one example that takes a function and one that returns a function.

### 28. What is a pure function and why does it matter?
**Short answer:** A pure function returns the same output for the same input and has no side effects (no mutation, I/O, or hidden state).
- Pure functions are trivially testable, cacheable (memoizable), and safe to run in parallel.
- React reducers and render logic rely on purity.
- Mutating arguments or reading `Date.now()` / `Math.random()` makes a function impure.
```js
const impure = (arr) => { arr.push(1); return arr; };
const pure = (arr) => [...arr, 1];
const src = [0];
pure(src); console.log(src);
impure(src); console.log(src);
```
**Output:** `[ 0 ]` then `[ 0, 1 ]`
**Interview tip:** Tie purity to memoization and React state updates.

### 29. Explain the prototype chain and prototypal inheritance.
**Short answer:** Every object has an internal `[[Prototype]]` link; property lookup walks that chain until it finds the key or reaches `null`.
- `Object.getPrototypeOf(obj)` reads the link; `__proto__` is the legacy accessor.
- `fn.prototype` is the object assigned as `[[Prototype]]` of instances made with `new fn`.
- Methods on the prototype are shared across instances, saving memory.
- `hasOwnProperty` / `Object.hasOwn` distinguish own from inherited keys.
- Writing to an inherited property creates an own property (shadowing).
```js
function Animal(name) { this.name = name; }
Animal.prototype.speak = function () { return this.name + " makes a sound"; };
const dog = new Animal("Rex");
console.log(dog.speak(), Object.hasOwn(dog, "speak"), "speak" in dog);
console.log(Object.getPrototypeOf(dog) === Animal.prototype);
console.log(Object.getPrototypeOf(Animal.prototype) === Object.prototype);
```
**Output:** `Rex makes a sound false true` then `true` then `true`
**Interview tip:** Draw the chain `dog -> Animal.prototype -> Object.prototype -> null`.

### 30. How do ES6 classes relate to prototypes? Explain static, private fields, and extends.
**Short answer:** Classes are syntactic sugar over prototypes: methods live on `Class.prototype`, `extends` wires the prototype chain, and `super` calls the parent.
- A class body is always in strict mode and class declarations are not hoisted like functions.
- `static` members live on the constructor itself, not on instances.
- `#field` is truly private (enforced by the engine), unlike the `_name` convention.
- Calling a class without `new` throws a TypeError.
- In a derived constructor you must call `super()` before touching `this`.
```js
class Account {
  #balance = 0;
  static count = 0;
  constructor(owner) { this.owner = owner; Account.count++; }
  deposit(n) { this.#balance += n; return this; }
  get balance() { return this.#balance; }
}
class Savings extends Account {
  addInterest() { return this.deposit(this.balance * 0.1); }
}
const s = new Savings("Ravi").deposit(100).addInterest();
console.log(s.balance, Account.count, typeof Account, s instanceof Account);
console.log(Object.hasOwn(s, "balance"), Object.hasOwn(Account.prototype, "deposit"));
try { Account("x"); } catch (e) { console.log(e.constructor.name); }
```
**Output:** `110 1 function true` then `false true` then `TypeError`
**Interview tip:** Say "sugar over prototypes, with real private fields" and show `typeof Class === "function"`.

### 31. What does Object.create do and when would you use it?
**Short answer:** `Object.create(proto, descriptors?)` creates a new object whose `[[Prototype]]` is `proto`, without running a constructor.
- `Object.create(null)` makes a dictionary with no inherited keys (no `toString`, no `hasOwnProperty`).
- It is the building block of classical inheritance without classes.
- Property descriptors can be passed as the second argument.
```js
const base = { hello() { return "hi from " + this.name; } };
const child = Object.create(base);
child.name = "child";
console.log(child.hello(), Object.getPrototypeOf(child) === base);
const dict = Object.create(null);
dict.key = 1;
console.log("toString" in dict, "toString" in {});
```
**Output:** `hi from child true` then `false true`
**Interview tip:** Mention `Object.create(null)` as a safe hash map for untrusted keys.

### 32. What is structuredClone and what can it not copy?
**Short answer:** `structuredClone(value)` is the built-in deep clone that handles Date, Map, Set, RegExp, typed arrays, and circular references.
- It throws `DataCloneError` for functions, DOM nodes, and class private state.
- Class instances lose their prototype and become plain objects.
- Symbol keys and getters/setters are not preserved as such.
- Available in modern browsers and Node 17+.
```js
const src = { when: new Date(0), tags: new Set([1, 2]), m: new Map([["k", 1]]) };
src.self = src;
const c = structuredClone(src);
console.log(c.when instanceof Date, c.tags.has(2), c.self === c, c !== src);
class P { constructor() { this.x = 1; } hi() {} }
console.log(structuredClone(new P()) instanceof P);
try { structuredClone({ f() {} }); } catch (e) { console.log(e.name); }
```
**Output:** `true true true true` then `false` then `DataCloneError`
**Interview tip:** Pair it with the JSON-copy limitations from the previous question.

### 33. What is the difference between Object.freeze, Object.seal, and Object.preventExtensions?
**Short answer:** freeze blocks add, delete, and modify; seal blocks add and delete but allows modifying existing values; preventExtensions blocks only adding.
| Method | Add | Delete | Modify |
|---|---|---|---|
| preventExtensions | No | Yes | Yes |
| seal | No | No | Yes |
| freeze | No | No | No |
- All three are shallow; nested objects stay mutable.
- Failures are silent in sloppy mode and throw TypeError in strict mode.
- Check with `Object.isFrozen`, `isSealed`, `isExtensible`.
```js
"use strict";
const o = Object.freeze({ a: 1, inner: { b: 1 } });
o.inner.b = 2;
try { o.a = 2; } catch (e) { console.log(e.constructor.name); }
const s = Object.seal({ a: 1 });
s.a = 5;
try { s.z = 1; } catch (e) { console.log(e.constructor.name); }
console.log(o.inner.b, s.a, Object.isFrozen(o));
```
**Output:** `TypeError` then `TypeError` then `2 5 true`
**Interview tip:** Stress "shallow" and mention a recursive deepFreeze if asked.

### 34. What are getters and setters in JavaScript objects?
**Short answer:** Accessor properties run a function on read (`get`) or write (`set`) while looking like plain properties.
- Use them for computed values, validation, or read-only views.
- Define them in object literals, classes, or with `Object.defineProperty`.
- A getter without a setter is read-only (silently ignored in sloppy mode, TypeError in strict).
- They live on the prototype for classes, so they do not appear in `Object.keys` of an instance.
```js
const temp = {
  _c: 25,
  get f() { return this._c * 9 / 5 + 32; },
  set f(v) { this._c = (v - 32) * 5 / 9; },
};
temp.f = 212;
console.log(temp._c, temp.f);
const ro = { get id() { return 7; } };
ro.id = 9;
console.log(ro.id);
```
**Output:** `100 212` then `7`
**Interview tip:** Mention validation in setters and computed properties as the real use cases.

### 35. What is a Symbol and what is it used for?
**Short answer:** A Symbol is a unique, immutable primitive used as a collision-free property key or to hook into language behaviour via well-known symbols.
- `Symbol("x") !== Symbol("x")` even with the same description; `Symbol.for` shares one via the global registry.
- Symbol keys are skipped by `for...in`, `Object.keys`, and `JSON.stringify`.
- Well-known symbols: `Symbol.iterator`, `Symbol.asyncIterator`, `Symbol.toPrimitive`, `Symbol.hasInstance`.
- They give pseudo-private keys but are discoverable via `Object.getOwnPropertySymbols`.
```js
const id = Symbol("id");
const user = { name: "A", [id]: 123 };
console.log(Object.keys(user), JSON.stringify(user), user[id]);
console.log(Symbol("x") === Symbol("x"), Symbol.for("x") === Symbol.for("x"));
console.log(Object.getOwnPropertySymbols(user).length);
```
**Output:** `[ 'name' ] {"name":"A"} 123` then `false true` then `1`
**Interview tip:** Name `Symbol.iterator` as the practical one; it ties into iterators and generators.

### 36. What are WeakMap and WeakSet and when are they useful?
**Short answer:** They hold weak references to object keys (or members), so entries disappear when nothing else references the object, which prevents memory leaks.
- Keys must be objects (or non-registered symbols); primitives are rejected.
- They are not iterable and have no `size` or `clear`.
- Use cases: private data per object, caching metadata for DOM nodes, tracking visited objects.
- Normal Map would keep the key alive and leak memory.
```js
const priv = new WeakMap();
class User {
  constructor(name) { priv.set(this, { name }); }
  get name() { return priv.get(this).name; }
}
const u = new User("Meera");
console.log(u.name, priv.has(u));
try { priv.set("str", 1); } catch (e) { console.log(e.constructor.name); }
console.log(typeof priv[Symbol.iterator], priv.size);
```
**Output:** `Meera true` then `TypeError` then `undefined undefined`
**Interview tip:** Say "no leak because GC can drop the key" and note you cannot iterate them.

### 37. What is the difference between Map and a plain Object?
**Short answer:** Map allows any key type, preserves insertion order, has `size`, and is built for frequent add/remove; objects only allow string/symbol keys and inherit prototype keys.
| Feature | Map | Object |
|---|---|---|
| Key types | Any value | String or Symbol |
| Order | Insertion | Mostly insertion (integer keys first) |
| Size | `map.size` | `Object.keys(o).length` |
| Iteration | Directly iterable | Needs `Object.keys/entries` |
| Inherited keys | None | Yes (`toString` etc.) |
- Use Map for dynamic dictionaries and non-string keys; use objects for fixed records and JSON.
- Maps do not serialise to JSON directly.
```js
const m = new Map();
const keyObj = {};
m.set(keyObj, "obj").set(1, "num").set("1", "str");
console.log(m.size, m.get(keyObj), m.get(1), m.get("1"));
const o = {}; o[keyObj] = "a"; o[{}] = "b";
console.log(Object.keys(o), { b: 1, 2: 1, 1: 1 });
```
**Output:** `3 obj num str` then `[ '[object Object]' ] { '1': 1, '2': 1, b: 1 }`
**Interview tip:** Mention the object-key stringification and integer-key ordering traps.

### 38. What is a Set and how do you use it in practice?
**Short answer:** A Set stores unique values of any type with O(1) average `has`/`add`/`delete` and keeps insertion order.
- Deduplicate arrays: `[...new Set(arr)]`.
- Uniqueness uses SameValueZero, so `NaN` equals `NaN` and objects compare by reference.
- Modern engines also offer `union`, `intersection`, and `difference` methods.
- Prefer `set.has(x)` over `array.includes(x)` in hot loops.
```js
console.log([...new Set([1, 2, 2, 3, NaN, NaN, "1"])]);
const a = new Set([1, 2, 3]), b = new Set([2, 3, 4]);
console.log([...a].filter((x) => b.has(x)), new Set([{}, {}]).size);
```
**Output:** `[ 1, 2, 3, NaN, '1' ]` then `[ 2, 3 ] 2`
**Interview tip:** Mention O(1) lookup against array includes at O(n).

## Part 6: this

### 39. How is the value of `this` determined?

**Short answer:** `this` is decided by how a function is called, not where it is defined (except arrow functions): method call gives the object before the dot, `new` gives the new object, `call`/`apply`/`bind` give the chosen object, and a plain call gives `undefined` (strict) or the global object (sloppy).

| Call style | `this` |
|---|---|
| Global scope (script) | global object: `window` in browsers, `global` in Node (module top level in Node is `module.exports`) |
| Plain function call `f()` | `undefined` in strict mode, global object in sloppy mode |
| Method call `obj.f()` | `obj` |
| `new F()` | the newly created object |
| `f.call(x)` / `f.apply(x)` / `f.bind(x)()` | `x` |
| Arrow function | `this` of the enclosing lexical scope, fixed at creation |
| DOM event handler (regular function) | the element the listener is attached to (`event.currentTarget`) |

```js
const user = { name: "Asha", hi() { return this.name; } };
console.log(user.hi());
const detached = user.hi;
console.log(detached.call({ name: "Ben" }));
function Person(n) { this.n = n; }
console.log(new Person("Cy").n);
```
**Output:** `Asha`, `Ben`, `Cy`
**Interview tip:** Precedence is `new` over explicit binding (`bind`/`call`) over implicit (method) over default; arrow functions ignore all of them.

### 40. What is this substitution, and how does strict mode change `this`?

**Short answer:** In sloppy mode, if `this` would be `undefined` or `null` the engine substitutes the global object (and boxes primitives); in strict mode no substitution happens, so a plain function call sees `this === undefined`.
- `this` substitution is why an unbound method in sloppy mode silently reads or writes globals instead of crashing.
- Gotcha in the original notes: `('use strict');` with parentheses is just an expression, not a directive, so it does not enable strict mode. The directive must be a bare string literal at the top of a script or function.
- `window.x()` still gives `window` even in strict mode because it is a method call on `window`.
- ES modules and class bodies are always strict.
```js
'use strict';
function strictFn() { return this; }
console.log(strictFn());
console.log(strictFn.call(globalThis) === globalThis);

// new Function bodies are sloppy, so substitution applies
const sloppyFn = new Function("return this === globalThis");
console.log(sloppyFn());
```
**Output:** `undefined`, `true`, `true`
**Interview tip:** Call out that `class` methods are strict, which is why a detached class method throws instead of touching globals.

### 41. How do arrow functions treat `this`, and when should you not use them?

**Short answer:** Arrow functions have no `this` of their own; they capture `this` from the enclosing lexical scope when defined and it can never be changed afterwards, not even by `call`, `apply` or `bind`.
- Good for callbacks inside methods (`setTimeout`, `map`) where you want the outer `this`.
- Bad as object methods: `obj = { x: () => this }` captures the outer scope (module or `window`), not `obj`.
- Bad as constructors (no `new`, no `prototype`) and as DOM handlers that need `this` to be the element.
- They also have no own `arguments` and no `super` binding.
```js
const obj = {
  a: 20,
  regular: function () { return this.a; },
  arrow: () => this === obj,
  nested: function () {
    const y = () => this.a;
    return y();
  },
};
console.log(obj.regular(), obj.arrow(), obj.nested());

const timer = {
  start() {
    setTimeout(function () { console.log("function:", this === timer); }, 0);
    setTimeout(() => console.log("arrow:", this === timer), 0);
  },
};
timer.start();
```
**Output:** `20 false 20`, `function: false`, `arrow: true`
**Interview tip:** An arrow function inherits `this` from where it is written, so reading "enclosing lexical context" literally (the surrounding function, or module/global if none) answers every output puzzle.

### 42. Why does a method lose `this` when passed as a callback, and how do you fix it?

**Short answer:** Passing `obj.method` passes only the function; the object before the dot is not carried along, so when it is invoked as a plain function `this` is `undefined` (strict) or global (sloppy). Fix it with `bind`, an arrow wrapper, or class-field arrow functions.
- Extracting a method (`const { inc } = counter`) or handing it to `setTimeout`/`addEventListener` has the same effect.
- `bind` returns a new function with `this` fixed, while an arrow wrapper `() => counter.inc()` re-establishes the method call.
- In React class components this is why handlers were bound in the constructor.
- For DOM listeners written as regular functions, `this` is the element itself, so a method passed as a listener gets the element as `this`, not your object.
```js
class Counter {
  constructor() { this.n = 0; }
  inc() { this.n++; return this.n; }
}
const c = new Counter();
const { inc } = c;
try { inc(); } catch (e) { console.log(e.name + ": " + e.message); }
console.log(c.inc.bind(c)(), (() => c.inc())());
```
**Output:** `TypeError: Cannot read properties of undefined (reading 'n')`, `1 2`
**Interview tip:** `bind` creates a new function every call, so binding inside a render or in `addEventListener` makes `removeEventListener` impossible unless you keep the reference.

## Part 7: Promises & async/await

### 43. What are the drawbacks of callbacks, and how do promises solve them?

**Short answer:** Callbacks lead to callback hell (deeply nested code) and inversion of control (you hand your logic to code you do not control, which may call it twice, late or never). Promises give a standard, chainable object you control.
**Real-life example:** A callback is giving a stranger your phone number and hoping they call back once; a promise is a tracking number you check yourself.
- Callback hell: sequential async steps nest into a pyramid, making error handling and reading hard.
- Inversion of control: with a callback you trust a third-party function to invoke yours exactly once with the right arguments.
- A promise is an object representing the eventual completion (or failure) of an async operation; it settles once and handlers are always called asynchronously.
- Promise chains flatten the pyramid because each `then` returns a new promise.
```js
const wait = (v, ms) => new Promise((resolve) => setTimeout(() => resolve(v), ms));

wait("order", 10)
  .then((o) => wait(o + " -> pay", 10))
  .then((p) => wait(p + " -> ship", 10))
  .then(console.log);
```
**Output:** `order -> pay -> ship`
**Interview tip:** Promises fix the trust problem because resolution is one-shot and the consumer decides what to do with the result.

### 44. What are the states of a promise, and how do chaining and `catch` work?

**Short answer:** A promise is `pending`, then settles once to `fulfilled` or `rejected` and never changes again. `then` returns a new promise, a thrown error skips ahead to the next `catch`, and `catch` only handles errors from earlier in the chain.
- Calling `resolve` or `reject` more than once is ignored after the first.
- Returning a value from a handler fulfils the next promise with it; returning a promise makes the chain wait for it; throwing rejects the next promise.
- `catch(fn)` is `then(undefined, fn)`; handlers placed before it are covered, handlers after it are not.
- If `catch` returns normally the chain recovers; if it throws, the next `catch` is needed.
- `finally` runs either way and passes the original value or error through.
```js
const p = new Promise((resolve, reject) => {
  resolve("done");
  reject(new Error("ignored"));
});

p.then((v) => { console.log(v); return v.toUpperCase(); })
  .then((v) => { console.log(v); throw new Error("boom"); })
  .then(() => console.log("skipped"))
  .catch((e) => { console.log("caught", e.message); return "recovered"; })
  .then((v) => console.log(v))
  .finally(() => console.log("finally"));
```
**Output:** `done`, `DONE`, `caught boom`, `recovered`, `finally`
**Interview tip:** An unhandled rejection crashes modern Node (since v15) and fires `unhandledrejection` in browsers, so always end chains with a `catch`.

### 45. How does Promise.all work?

**Short answer:** `Promise.all` takes an iterable of promises and resolves with an array of results (in input order) once all fulfil; it rejects immediately with the first rejection reason.
- Results keep input order regardless of which promise finishes first.
- On the first rejection the combined promise rejects, but the other promises are not cancelled; they keep running and their results are ignored.
- Non-promise values are treated as already resolved, and an empty iterable resolves immediately with `[]`.
```js
const ok = (v, ms) => new Promise((resolve) => setTimeout(() => resolve(v), ms));
const fail = (v, ms) => new Promise((_, reject) => setTimeout(() => reject(v), ms));

Promise.all([ok("P1 Success", 30), ok("P2 Success", 10), ok("P3 Success", 20)])
  .then(console.log);

Promise.all([ok("P1 Success", 30), fail("P2 Fail", 10), ok("P3 Success", 20)])
  .catch(console.log);
```
**Output:** `P2 Fail`, then `[ 'P1 Success', 'P2 Success', 'P3 Success' ]`
**Interview tip:** Use it for independent parallel requests where one failure invalidates the whole result.

### 46. How does Promise.allSettled work?

**Short answer:** It waits for every promise to settle and never rejects; it resolves with an array of `{ status: "fulfilled", value }` or `{ status: "rejected", reason }` objects in input order.
- Correction to the usual note: a rejected entry holds `reason`, not `value`.
- Ideal when you want partial results, for example loading several widgets where some may fail.
- You inspect `status` per entry to separate successes from failures.
```js
const ok = (v, ms) => new Promise((resolve) => setTimeout(() => resolve(v), ms));
const fail = (v, ms) => new Promise((_, reject) => setTimeout(() => reject(v), ms));

Promise.allSettled([ok("P1 Success", 10), ok("P2 Success", 20), fail("P3 Fail", 30)])
  .then((r) => console.log(JSON.stringify(r)));
```
**Output:** `[{"status":"fulfilled","value":"P1 Success"},{"status":"fulfilled","value":"P2 Success"},{"status":"rejected","reason":"P3 Fail"}]`
**Interview tip:** Say "all or nothing" for `all` versus "best effort" for `allSettled`.

### 47. How does Promise.race work?

**Short answer:** `Promise.race` settles with the first promise to settle, whether fulfilled or rejected, and ignores the rest.
- The classic use is a timeout: race the real request against a timer that rejects.
- An empty iterable gives a promise that stays pending forever.
- The losing promises are not cancelled; use `AbortController` to cancel real work.
```js
const ok = (v, ms) => new Promise((resolve) => setTimeout(() => resolve(v), ms));
const fail = (v, ms) => new Promise((_, reject) => setTimeout(() => reject(v), ms));

Promise.race([ok("P1 Success", 30), ok("P2 Success", 20), fail("P3 Fail", 10)])
  .then(console.log, (e) => console.log("rejected:", e));
```
**Output:** `rejected: P3 Fail`
**Interview tip:** Note that race can reject, which is the difference from `any`.

### 48. How does Promise.any work, and what is AggregateError?

**Short answer:** `Promise.any` resolves with the first fulfilled promise and ignores rejections; only if every promise rejects does it reject with an `AggregateError` whose `errors` array holds all reasons.
- Good for fallbacks, such as the fastest healthy mirror or CDN.
- `error.errors` keeps the reasons in input order.
- Added in ES2021, so older environments need a polyfill.
```js
const ok = (v, ms) => new Promise((resolve) => setTimeout(() => resolve(v), ms));
const fail = (v, ms) => new Promise((_, reject) => setTimeout(() => reject(v), ms));

Promise.any([fail("P1 Fail", 10), ok("P2 Success", 20), fail("P3 Fail", 5)]).then(console.log);

Promise.any([fail("P1 Fail", 10), fail("P2 Fail", 20), fail("P3 Fail", 5)])
  .catch((e) => console.log(e.name, e.message, e.errors));
```
**Output:** `P2 Success`, then `AggregateError All promises were rejected [ 'P1 Fail', 'P2 Fail', 'P3 Fail' ]`
**Interview tip:** Be ready to give the one-line contrast: `any` ignores rejections, `race` does not.

### 49. When would you use all, allSettled, race or any?

**Short answer:** Use `all` when every result is required, `allSettled` when you want every outcome regardless of failures, `race` for the first outcome of any kind (timeouts), and `any` for the first success.

| API | Resolves when | Rejects when | Result |
|---|---|---|---|
| `Promise.all` | all fulfil | first rejection | array of values |
| `Promise.allSettled` | all settle | never | array of `{status, value/reason}` |
| `Promise.race` | first settles as fulfilled | first settles as rejected | first outcome |
| `Promise.any` | first fulfils | all reject (`AggregateError`) | first value |

- None of them cancel the work of the promises they ignore.
- All four accept any iterable and treat non-promises as resolved values.
**Interview tip:** Mention `Promise.withResolvers` and `AbortSignal.timeout` as modern companions for timeouts and manual resolution.

### 50. How do async functions and `await` work?

**Short answer:** An `async` function always returns a promise: a returned value is wrapped in a resolved promise and a thrown error becomes a rejection. `await` pauses only that function until the promise settles, letting the rest of the program keep running.
- `await` is allowed inside async functions (and at the top level of ES modules).
- Correction: returning a promise from an async function does not hand back that same promise object, it returns a new promise that follows it.
- Code after `await` runs as a microtask continuation, so it is always asynchronous.
- `await value` on a non-promise still yields to the microtask queue.
```js
const f = async () => 1;
const base = Promise.resolve(2);
const g = async () => base;
console.log(f() instanceof Promise, g() === base);

f().then(console.log);
g().then(console.log);

(async () => {
  console.log("Hello World");
  await null;
  console.log("after await");
})();
console.log("sync end");
```
**Output:** `true false`, `Hello World`, `sync end`, `1`, `after await`, `2` (the `2` takes extra ticks because the async function must adopt the inner promise)
**Interview tip:** `async/await` is syntax over promises; explain that each `await` is a `.then` continuation on the microtask queue.

### 51. How do you handle errors in async/await?

**Short answer:** Wrap awaits in `try/catch/finally`, or attach `.catch` to the returned promise; a rejected awaited promise throws at the `await` line.
- `try/catch` catches both rejections and synchronous throws inside the block.
- Forgetting `await` inside `try` lets the rejection escape the `catch`.
- A common helper returns `[error, data]` tuples so callers avoid nested try blocks.
```js
const bad = () => Promise.reject(new Error("nope"));

async function run() {
  try {
    await bad();
  } catch (e) {
    console.log("caught", e.message);
  } finally {
    console.log("cleanup");
  }
}
run();
```
**Output:** `caught nope`, `cleanup`
**Interview tip:** `return await promise` inside `try` is needed if you want the `catch` to handle that rejection; plain `return promise` skips it.

## Part 8: Async JavaScript

### 52. Predict the order: sync code, process.nextTick/setTimeout, promises, and queueMicrotask. What is the difference between microtasks and macrotasks?
**Short answer:** After each macrotask (script, timer, I/O callback) the engine drains the entire microtask queue (promise callbacks, queueMicrotask) before running the next macrotask.
- Macrotasks: setTimeout, setInterval, I/O, UI events, MessageChannel.
- Microtasks: Promise `.then/.catch/.finally`, `await` continuations, `queueMicrotask`, MutationObserver.
- A microtask that queues another microtask runs in the same drain, so it can starve timers.
- In Node, `process.nextTick` runs before promise microtasks.
```js
console.log("1 sync");
setTimeout(() => console.log("5 timeout"), 0);
Promise.resolve().then(() => {
  console.log("3 micro A");
  queueMicrotask(() => console.log("4 micro nested"));
});
queueMicrotask(() => console.log("3b micro B"));
console.log("2 sync end");
```
**Output:** `1 sync`, `2 sync end`, `3 micro A`, `3b micro B`, `4 micro nested`, `5 timeout` (one per line)
**Interview tip:** Say "drain all microtasks, then one macrotask" and note nested microtasks run before timers.

### 53. What is AbortController and how do you cancel async work?
**Short answer:** AbortController gives you a `signal` you pass to cancellable APIs (fetch, event listeners, timers/promises); calling `abort()` makes them reject with `AbortError`.
- One controller can cancel many operations through the same signal.
- Typical use: cancel the previous request when a search input changes or a React component unmounts.
- `AbortSignal.timeout(ms)` creates a signal that aborts automatically.
- Custom code can listen to `signal.addEventListener("abort", ...)`.
```js
function cancellable(ms, signal) {
  return new Promise((resolve, reject) => {
    const t = setTimeout(() => resolve("done"), ms);
    signal.addEventListener("abort", () => { clearTimeout(t); reject(signal.reason); });
  });
}
const ctrl = new AbortController();
cancellable(100, ctrl.signal).then(console.log, (e) => console.log(e.name));
setTimeout(() => ctrl.abort(), 10);
```
**Output:** `AbortError`
**Interview tip:** Use the "cancel stale search requests" example; it shows practical value.

### 54. What are generators and where are they useful?
**Short answer:** A generator function (`function*`) can pause at `yield` and resume later, producing values lazily and enabling two-way communication via `next(value)`.
- Calling it returns an iterator; the body runs only as you call `next()`.
- Great for lazy sequences, infinite streams, and custom iteration.
- `next(x)` sends `x` back as the result of the paused `yield` expression.
- Redux-Saga and old `co`-style async code built on them; async/await replaced that.
```js
function* idGen() { let id = 1; while (true) { const reset = yield id++; if (reset) id = 1; } }
const g = idGen();
console.log(g.next().value, g.next().value, g.next(true).value, g.next().value);
function* take(n, it) { let i = 0; for (const v of it) { if (i++ >= n) return; yield v; } }
console.log([...take(3, idGen())]);
```
**Output:** `1 2 1 2` then `[ 1, 2, 3 ]`
**Interview tip:** Highlight laziness (infinite sequences) and that `yield` can receive values.

### 55. What are iterators and the iterable protocol? What are async iterators?
**Short answer:** An iterable has a `[Symbol.iterator]()` method returning an iterator, an object with `next()` that returns `{ value, done }`; async iterables use `Symbol.asyncIterator` and are consumed with `for await...of`.
- `for...of`, spread, destructuring, `Array.from`, `Map`, and `Set` consume iterables.
- Plain objects are not iterable unless you add `[Symbol.iterator]`.
- Async iterators suit paginated APIs and streams.
```js
const range = {
  from: 1, to: 4,
  [Symbol.iterator]() {
    let c = this.from, t = this.to;
    return { next: () => (c <= t ? { value: c++, done: false } : { value: undefined, done: true }) };
  },
};
console.log([...range], Math.max(...range));
async function* pages() { for (let p = 1; p <= 3; p++) yield p; }
(async () => { const out = []; for await (const p of pages()) out.push(p); console.log(out); })();
```
**Output:** `[ 1, 2, 3, 4 ] 4` then `[ 1, 2, 3 ]`
**Interview tip:** Recite the `{ value, done }` contract and mention `for await` for streams.

### 56. How do you run async calls in sequence vs parallel, and why does await inside forEach not work?
**Short answer:** `await` in a `for...of` loop runs sequentially; start all promises first then `Promise.all` to run in parallel; `forEach` ignores returned promises so it does not wait.
- Sequential is right when each step depends on the previous result or order matters.
- Parallel is faster for independent calls; limit concurrency for large lists.
- `forEach(async ...)` fires all callbacks and returns immediately.
- Use `for...of` with await, or `await Promise.all(arr.map(async ...))`.
```js
const wait = (ms, v) => new Promise((r) => setTimeout(() => r(v), ms));
(async () => {
  let t = Date.now();
  for (const n of [1, 2, 3]) await wait(30, n);
  const seq = Date.now() - t;
  t = Date.now();
  await Promise.all([1, 2, 3].map((n) => wait(30, n)));
  const par = Date.now() - t;
  console.log(seq >= 90, par < 80);
  [1, 2].forEach(async (n) => { await wait(10); console.log("forEach item", n); });
  console.log("after forEach");
})();
```
**Output:** `true true`, then `after forEach`, then `forEach item 1`, `forEach item 2`
**Interview tip:** Mention a concurrency limiter (p-limit style) for hundreds of requests.

## Part 9: ES6 and Beyond

### 57. How does destructuring work, including defaults, renaming, and nesting?
**Short answer:** Destructuring unpacks array elements by position or object properties by name into variables, with optional defaults and renames.
- Defaults apply only when the value is `undefined`.
- Rename with `{ a: alias }`; nest with `{ a: { b } }`.
- Swap variables with `[a, b] = [b, a]`.
- Function parameters can be destructured with defaults, a common config pattern.
- Destructuring `null` or `undefined` throws a TypeError.
```js
const { name: n, age = 30, address: { city } = {} } = { name: "Ria", address: { city: "Pune" } };
console.log(n, age, city);
const [first, , third = "def", ...rest] = [1, 2, undefined, 4, 5];
console.log(first, third, rest);
let a = 1, b = 2; [a, b] = [b, a];
console.log(a, b);
```
**Output:** `Ria 30 Pune` then `1 def [ 4, 5 ]` then `2 1`
**Interview tip:** Mention that defaults skip `null` and that parameter destructuring is how config objects are written.

### 58. What is the difference between spread and rest operators?
**Short answer:** Both use `...`; rest collects remaining items into an array in a parameter or destructuring position, spread expands an iterable or object into individual items.
- Rest: `function f(...args)`, `const [h, ...t] = arr`.
- Spread: `f(...arr)`, `[...a, ...b]`, `{ ...obj, k: 1 }`.
- Object spread is shallow and later keys override earlier ones.
- Rest must be the last element.
```js
const sum = (...nums) => nums.reduce((a, b) => a + b, 0);
console.log(sum(...[1, 2, 3], 4));
const base = { a: 1, b: 2 };
console.log({ ...base, b: 3, c: 4 }, [..."hi"]);
const { a, ...others } = base;
console.log(a, others);
```
**Output:** `10` then `{ a: 1, b: 3, c: 4 } [ 'h', 'i' ]` then `1 { b: 2 }`
**Interview tip:** Say "rest gathers, spread scatters" and point out the shallow copy limit.

### 59. What is optional chaining and when should you avoid it?
**Short answer:** `a?.b`, `a?.[k]`, and `a?.()` short-circuit to `undefined` when `a` is null or undefined instead of throwing.
- It only guards `null` and `undefined`, not other falsy values such as 0 or "".
- The whole rest of the chain is skipped on short-circuit.
- Avoid chaining everything; it hides bugs where the data should always exist.
- Combine with `??` for defaults: `user?.profile?.name ?? "Guest"`.
```js
const user = { profile: { greet() { return "hi"; } }, count: 0 };
console.log(user?.profile?.greet?.(), user.settings?.theme, user.missing?.fn());
console.log(user.count?.toFixed(1), user?.["pro" + "file"] !== undefined);
```
**Output:** `hi undefined undefined` then `0.0 true`
**Interview tip:** Warn that over-using `?.` can mask real data bugs.

### 60. What is the nullish coalescing operator and how is it different from ||?
**Short answer:** `a ?? b` returns `b` only when `a` is null or undefined, while `a || b` returns `b` for any falsy `a`.
- Use `??` when 0, false, or "" are legitimate values.
- `??=` assigns only when the left side is nullish; `||=` when falsy.
- Mixing `??` with `||` or `&&` without parentheses is a SyntaxError.
```js
const settings = { volume: 0, title: "", dark: false };
console.log(settings.volume || 50, settings.volume ?? 50);
console.log(JSON.stringify(settings.title || "Untitled"), JSON.stringify(settings.title ?? "Untitled"));
const o = { a: null, b: 0 };
o.a ??= "set"; o.b ||= 9;
console.log(o);
```
**Output:** `50 0` then `"Untitled" ""` then `{ a: 'set', b: 9 }`
**Interview tip:** Use the volume-set-to-0 example; it is the classic bug.

### 61. What is the difference between ES Modules and CommonJS?
**Short answer:** ESM (`import`/`export`) is static, asynchronous, and exports live bindings; CommonJS (`require`/`module.exports`) is dynamic, synchronous, and exports copies of values.
| Feature | ESM | CommonJS |
|---|---|---|
| Syntax | `import` / `export` | `require` / `module.exports` |
| Loading | Static analysis, async | Runtime, sync |
| Bindings | Live read-only | Value copy |
| Tree shaking | Yes | Hard |
| Top-level await | Yes | No |
| `this` at top | `undefined` | `module.exports` |
- ESM imports are hoisted and parsed before execution; `import()` is the dynamic form.
- Browsers support ESM natively with `<script type="module">`, which is deferred by default.
- Mixed codebases need care: `require` of ESM is awkward, and `__dirname` does not exist in ESM.
```js
// CommonJS: exports are copies
let count = 1;
const mod = { get count() { return count; }, inc: () => count++ };
const copied = mod.count;
mod.inc();
console.log(copied, mod.count);
// ESM would give live bindings: importing `count` afterwards sees 2
```
**Output:** `1 2`
**Interview tip:** The key phrases are "static vs dynamic" and "live bindings vs copies", plus tree shaking.

### 62. What are tagged template literals?
**Short answer:** A tag is a function that receives the string pieces and interpolated values of a template literal, letting you process or escape them.
- The tag gets `(strings, ...values)`; `strings.raw` holds unprocessed text.
- Used by styled-components, GraphQL (`gql`), and safe-HTML or i18n helpers.
- It can return anything, not just a string.
```js
function highlight(strings, ...values) {
  return strings.reduce((out, s, i) => out + s + (i < values.length ? `[${values[i]}]` : ""), "");
}
const name = "Zoya", n = 3;
console.log(highlight`Hello ${name}, you have ${n} items`);
const safe = (s, ...v) => s.reduce((o, str, i) => o + str + (v[i] ?? "").toString().replace(/</g, "&lt;"), "");
console.log(safe`<p>${"<script>"}</p>`);
```
**Output:** `Hello [Zoya], you have [3] items` then `<p>&lt;script></p>`
**Interview tip:** Name styled-components and gql as familiar real-world users.

### 63. What are Proxy and Reflect, and where are they used?
**Short answer:** A Proxy wraps an object and intercepts operations (get, set, has, deleteProperty, and more) through traps; Reflect provides the matching default operations.
- Use cases: validation, default values, logging, reactivity (Vue 3), read-only views.
- Traps should usually call the matching `Reflect` method to preserve default behaviour.
- A `set` trap must return true in strict mode or it throws.
- Proxies add overhead and can break identity checks, so use them sparingly.
```js
const validator = {
  set(target, key, value, receiver) {
    if (key === "age" && !Number.isInteger(value)) throw new TypeError("age must be integer");
    return Reflect.set(target, key, value, receiver);
  },
  get(target, key, receiver) { return key in target ? Reflect.get(target, key, receiver) : `no ${String(key)}`; },
};
const p = new Proxy({}, validator);
p.age = 30;
console.log(p.age, p.nope);
try { p.age = "x"; } catch (e) { console.log(e.message); }
```
**Output:** `30 no nope` then `age must be integer`
**Interview tip:** Mention that Vue 3 reactivity is built on Proxy.

## Part 10: DOM and Events

### 64. Explain event bubbling and capturing.
**Short answer:** An event travels in three phases: capturing from window down to the target, the target phase, then bubbling back up; listeners run on bubble by default.
- Pass `{ capture: true }` (or `true`) as the third argument to listen during capture.
- `event.target` is the element that triggered the event; `event.currentTarget` is the element the listener is attached to.
- Most events bubble; `focus`, `blur`, and `mouseenter` do not (use `focusin`/`focusout`).
- `{ once: true }` removes the listener after the first run.
```js
// Plain simulation of the phase order (the DOM itself is not available in Node)
const path = ["window", "div", "button"];
const log = [];
path.forEach((n) => log.push(n + " capture"));
log.push("button target");
[...path].reverse().slice(1).forEach((n) => log.push(n + " bubble"));
console.log(log.join(" > "));
```
**Output:** `window capture > div capture > button capture > button target > div bubble > window bubble`
**Interview tip:** Always name the three phases and the target vs currentTarget difference.

### 65. What is event delegation and why is it useful?
**Short answer:** Attach one listener on a parent and use `event.target` to handle events from many (even future) children, relying on bubbling.
- Saves memory: one listener instead of hundreds.
- Works for dynamically added elements.
- Use `event.target.closest(selector)` to handle clicks on nested children.
- It does not work for events that do not bubble.
```js
// Pseudo-DOM so the logic can be run in Node: parent receives bubbled event
function dispatch(parentHandler, targetEl) { parentHandler({ target: targetEl }); }
const items = [{ tag: "LI", id: 1 }, { tag: "LI", id: 2 }, { tag: "SPAN", id: 3 }];
const handler = (e) => { if (e.target.tag === "LI") console.log("clicked item", e.target.id); };
items.forEach((el) => dispatch(handler, el));
// Browser version:
// list.addEventListener("click", (e) => { const li = e.target.closest("li"); if (li && list.contains(li)) {...} });
```
**Output:** `clicked item 1` then `clicked item 2`
**Interview tip:** Mention `closest()` for nested children and dynamic elements as the benefits.

### 66. What is the difference between preventDefault, stopPropagation, and stopImmediatePropagation?
**Short answer:** preventDefault cancels the browser's default action; stopPropagation stops the event travelling to other elements; stopImmediatePropagation also stops other listeners on the same element.
| Method | Stops default action | Stops other elements | Stops same-element listeners |
|---|---|---|---|
| preventDefault | Yes | No | No |
| stopPropagation | No | Yes | No |
| stopImmediatePropagation | No | Yes | Yes |
- `return false` in a jQuery handler does both preventDefault and stopPropagation; in plain DOM handlers it only prevents default for `onclick` style handlers.
- Use `event.defaultPrevented` to check and `cancelable` to know if cancelling is possible.
- Passive listeners (`{ passive: true }`) cannot call preventDefault; browsers use this for smooth scrolling.
```js
const t = new EventTarget();
t.addEventListener("ping", (e) => { console.log("first"); e.stopImmediatePropagation(); });
t.addEventListener("ping", () => console.log("second never runs"));
const ev = new Event("ping", { cancelable: true });
ev.preventDefault();
t.dispatchEvent(ev);
console.log(ev.defaultPrevented);
```
**Output:** `first` then `true`
**Interview tip:** Give the link-click (default) vs nested-button-in-card (propagation) examples.

### 67. What are the differences between localStorage, sessionStorage, and cookies?
**Short answer:** localStorage persists until cleared, sessionStorage lasts for the tab session, and cookies are small, can expire, and are sent to the server with each request.
| Feature | localStorage | sessionStorage | Cookies |
|---|---|---|---|
| Lifetime | Until cleared | Until tab closes | Set by `Expires`/`Max-Age` |
| Size | About 5 MB | About 5 MB | About 4 KB |
| Sent to server | No | No | Yes, every request |
| Scope | Origin, shared across tabs | Origin and single tab | Domain and path |
| JS access | Always | Always | Blocked by `HttpOnly` |
- Storage APIs are synchronous and store strings only (use JSON.stringify).
- Never store tokens in localStorage if XSS is a concern; prefer `HttpOnly; Secure; SameSite` cookies.
- Cookie flags: `HttpOnly` (hide from JS), `Secure` (HTTPS only), `SameSite` (CSRF control).
- Other options: IndexedDB for large structured data.
```js
// Browser-only APIs; here is a Node-safe demonstration of the string-only rule
const store = new Map();
const setItem = (k, v) => store.set(k, String(v));
setItem("obj", { a: 1 }); setItem("n", 5);
console.log(store.get("obj"), typeof store.get("n"));
console.log(JSON.stringify({ a: 1 }), JSON.parse('{"a":1}').a);
```
**Output:** `[object Object] string` then `{"a":1} 1`
**Interview tip:** Lead with the XSS/HttpOnly trade-off; it separates seniors from juniors.

### 68. What is the difference between debounce and throttle?
**Short answer:** Debounce waits until activity stops for N ms and then fires once; throttle fires at most once every N ms during continuous activity.
**Real-life example:** Debounce is an elevator that waits until people stop entering; throttle is a bus leaving every 10 minutes regardless.
- Debounce: search-as-you-type, window-resize end, autosave.
- Throttle: scroll position tracking, mousemove, button spam, infinite-scroll checks.
- Debounce can delay forever if events never pause; throttle guarantees regular updates.
- Options: leading edge (fire first) and trailing edge (fire last).
```js
const calls = [0, 10, 20, 30, 200];
const debounced = []; const throttled = [];
let lastFire = -Infinity;
calls.forEach((t, i) => {
  const next = calls[i + 1];
  if (next === undefined || next - t >= 100) debounced.push(t);
  if (t - lastFire >= 100) { throttled.push(t); lastFire = t; }
});
console.log("debounce fires at", debounced, "throttle fires at", throttled);
```
**Output:** `debounce fires at [ 30, 200 ] throttle fires at [ 0, 200 ]`
**Interview tip:** Give search box for debounce and scroll for throttle, then offer to implement both.

## Part 11: Polyfills

### 69. How do you resolve an array of promise-returning tasks sequentially?

**Short answer:** Fold the task list with `reduce`, starting from `Promise.resolve([])`, and chain each task inside the previous promise's `then` so the next one starts only after the last finishes.
- Tasks must be functions returning promises (not promises already running), otherwise they would all start immediately.
- Each step spreads the previous results and appends the new one.
- Modern equivalent: a `for...of` loop with `await`.
```js
function resolvePromisesSequentially(tasks) {
  return tasks.reduce(
    (promise, task) => promise.then((result) => task().then((current) => [...result, current])),
    Promise.resolve([])
  );
}

const delay = (v, ms) => () => new Promise((resolve) => setTimeout(() => resolve(v), ms));
resolvePromisesSequentially([delay("Task 1", 30), delay("Task 2", 10), delay("Task 3", 5)])
  .then(console.log);

async function sequential(tasks) {
  const results = [];
  for (const task of tasks) results.push(await task());
  return results;
}
```
**Output:** `[ 'Task 1', 'Task 2', 'Task 3' ]`
**Interview tip:** Mention a concurrency-limited pool as the follow-up (run N tasks at a time).

### 70. How do you implement infinite currying, like add(1)(2)(3)()?

**Short answer:** Return a function that either accumulates the next argument and returns itself again, or, when called with no argument, returns the accumulated total.
- The note's version tests `if (b)`, which wrongly stops on `0`; check `b === undefined` instead.
- Each call returns a new closure holding the running sum.
- A variant without the final `()` overrides `valueOf`/`toString` so that coercion yields the total.
```js
function add(a) {
  return function (b) {
    if (b === undefined) return a;
    return add(a + b);
  };
}
console.log(add(1)(2)(3)(4)(5)());
console.log(add(5)(0)(3)());
```
**Output:** `15`, `8`
**Interview tip:** If asked for `add(1)(2)(3) == 6` without the trailing call, mention overriding `valueOf` on the returned function.

### 71. How do you implement curry?

**Short answer:** Collect arguments across calls until their count reaches the original function's `length`, then call it with all of them.
- `func.length` is the number of declared parameters (it ignores rest parameters and defaults).
- It supports mixed calls such as `curried(1, 2)(3)` and `curried(1)(2, 3)`.
- Currying enables partial application and reusable specialised functions.
```js
function curry(func) {
  return function curried(...args) {
    if (args.length >= func.length) return func.apply(this, args);
    return (...next) => curried.apply(this, [...args, ...next]);
  };
}

const sum = (a, b, c) => a + b + c;
const curriedSum = curry(sum);
console.log(curriedSum(1)(2)(3), curriedSum(1, 2)(3), curriedSum(1)(2, 3));
```
**Output:** `6 6 6`
**Interview tip:** Contrast currying (one argument at a time) with partial application (fix some arguments now).

### 72. How do you implement Function.prototype.call?

**Short answer:** Temporarily attach the function to the target object under a unique key, call it as a method so `this` is that object, then remove it and return the result.
- The note's `context.fn` can overwrite a real `fn` property and does not box primitives; use a `Symbol` key and `Object(context)`.
- `window` does not exist in Node; use `globalThis`.
- Wrapping in `try/finally` guarantees cleanup even if the function throws.
- This mimics sloppy-mode semantics (null or undefined becomes global); the native `call` follows the callee's strictness.
```js
Function.prototype.myCall = function (context, ...args) {
  if (typeof this !== "function") throw new TypeError("myCall must be called on a function");
  context = context === null || context === undefined ? globalThis : Object(context);
  const key = Symbol("fn");
  context[key] = this;
  try {
    return context[key](...args);
  } finally {
    delete context[key];
  }
};

function greet(greeting, punct) { return greeting + ", " + this.name + punct; }
console.log(greet.myCall({ name: "Asha" }, "Hi", "!"));
console.log(greet.call({ name: "Asha" }, "Hi", "!"));
```
**Output:** `Hi, Asha!`, `Hi, Asha!`
**Interview tip:** The key trick is "make the function a method of the object", because `this` is set by the receiver of the call.

### 73. How do you implement Function.prototype.apply?

**Short answer:** Same as `call`, but the arguments arrive as one array (or array-like), so unpack them when invoking.
- A missing or `null` argument list means no arguments.
- Real `apply` accepts any array-like, so convert with `Array.from` rather than rejecting everything that is not an array.
- Non-object, non-nullish argument lists throw `TypeError`.
```js
Function.prototype.myApply = function (context, args) {
  if (typeof this !== "function") throw new TypeError("myApply must be called on a function");
  if (args !== null && args !== undefined && typeof args !== "object") {
    throw new TypeError("Arguments must be an array-like object");
  }
  context = context === null || context === undefined ? globalThis : Object(context);
  const key = Symbol("fn");
  context[key] = this;
  try {
    return context[key](...Array.from(args || []));
  } finally {
    delete context[key];
  }
};

function greet(greeting, punct) { return greeting + ", " + this.name + punct; }
console.log(greet.myApply({ name: "Ben" }, ["Yo", "?"]));
console.log(Math.max.myApply(null, [1, 5, 3]));
```
**Output:** `Yo, Ben?`, `5`
**Interview tip:** Mnemonic: `apply` takes an array, `call` takes a comma-separated list.

### 74. How do you implement Function.prototype.bind?

**Short answer:** Return a new function that calls the original with a fixed `this` and pre-filled arguments, appending any later arguments; it must return the result and also work with `new`.
- The note's version never returns the result of the call, and `context.fn` pollutes the target object.
- `target.apply(context, ...)` is enough for `this` binding; no temporary property is needed.
- With `new`, the bound `this` is ignored and the freshly created object is used, so the prototype chain must still link to the original.
- Partial application falls out for free by concatenating bound and new arguments.
```js
Function.prototype.myBind = function (context, ...boundArgs) {
  const target = this;
  if (typeof target !== "function") throw new TypeError("myBind must be called on a function");
  function bound(...newArgs) {
    const useNew = this instanceof bound;
    return target.apply(useNew ? this : context, [...boundArgs, ...newArgs]);
  }
  if (target.prototype) bound.prototype = Object.create(target.prototype);
  return bound;
};

function greet(g, p) { return g + ", " + this.name + p; }
console.log(greet.myBind({ name: "Asha" }, "Hello")("!"));

function Point(x, y) { this.x = x; this.y = y; }
const PointX1 = Point.myBind(null, 1);
const pt = new PointX1(2);
console.log(pt.x, pt.y, pt instanceof Point);
```
**Output:** `Hello, Asha!`, `1 2 true`
**Interview tip:** A function's `this` can be bound only once; calling `bind` on an already bound function cannot change it.

### 75. How do you implement Array.prototype.map?

**Short answer:** Loop over the array, call the callback with `(element, index, array)` and a `thisArg`, store each return value in a new array of the same length, and skip holes.
- `map` never mutates the original array and always returns a new array of equal length.
- Validate that the callback is a function, throwing `TypeError` otherwise.
- Sparse arrays: skipped holes stay holes in the output, hence the `i in this` check.
```js
Array.prototype.myMap = function (callback, thisArg) {
  if (typeof callback !== "function") throw new TypeError(callback + " is not a function");
  const result = new Array(this.length);
  for (let i = 0; i < this.length; i++) {
    if (i in this) result[i] = callback.call(thisArg, this[i], i, this);
  }
  return result;
};

console.log([1, 2, 3].myMap((n, i) => n * 2 + i));
console.log(["1", "2", "3"].myMap(Number));
```
**Output:** `[ 2, 5, 8 ]`, `[ 1, 2, 3 ]`
**Interview tip:** The classic trap is `["1","2","3"].map(parseInt)` giving `[1, NaN, NaN]` because `map` passes the index as the radix.

### 76. How do you implement Array.prototype.filter?

**Short answer:** Loop, call the callback for each element and push the element into a new array only when the callback returns a truthy value.
- Returns a new array that may be shorter; it does not mutate the original.
- The callback receives `(element, index, array)` and a `thisArg`.
- Holes are skipped.
```js
Array.prototype.myFilter = function (callback, thisArg) {
  if (typeof callback !== "function") throw new TypeError(callback + " is not a function");
  const result = [];
  for (let i = 0; i < this.length; i++) {
    if (i in this && callback.call(thisArg, this[i], i, this)) result.push(this[i]);
  }
  return result;
};

console.log([1, 2, 3, 4, 5].myFilter((n) => n % 2));
console.log(["a", "", "b", 0].myFilter(Boolean));
```
**Output:** `[ 1, 3, 5 ]`, `[ 'a', 'b' ]`
**Interview tip:** Mention that `filter` plus `map` walks the array twice and `reduce` can fuse them into one pass.

### 77. How do you implement Array.prototype.reduce?

**Short answer:** Keep an accumulator; if no initial value is given use the first present element as the start, then fold the callback `(acc, element, index, array)` over the rest.
- With no initial value and an empty array, native `reduce` throws `TypeError`; the polyfill must too.
- Check `arguments`-style presence (not truthiness) so an initial value of `0` or `undefined` is respected.
- Skip holes and start the loop after the element used as the seed.
```js
Array.prototype.myReduce = function (callback, ...initial) {
  if (typeof callback !== "function") throw new TypeError(callback + " is not a function");
  let i = 0;
  let acc;
  if (initial.length) {
    acc = initial[0];
  } else {
    while (i < this.length && !(i in this)) i++;
    if (i >= this.length) throw new TypeError("Reduce of empty array with no initial value");
    acc = this[i++];
  }
  for (; i < this.length; i++) {
    if (i in this) acc = callback(acc, this[i], i, this);
  }
  return acc;
};

console.log([1, 2, 3, 4].myReduce((a, b) => a + b));
console.log([1, 2, 3].myReduce((a, b) => a + b, 10));
try { [].myReduce((a, b) => a + b); } catch (e) { console.log(e.message); }
```
**Output:** `10`, `16`, `Reduce of empty array with no initial value`
**Interview tip:** Once you have `reduce`, you can derive `map`, `filter`, `flat` and `Promise` sequencing from it.

### 78. How do you implement a Promise from scratch?

**Short answer:** Keep a state (`pending`/`fulfilled`/`rejected`), a value and a list of handlers; `then` returns a new promise and every handler runs asynchronously (microtask) after the promise settles.
- Bugs in the original note: `reject` called `this.onReject(value)` with an undefined variable (should be `this.value`); `then` returned `this`, so chaining and multiple handlers do not work; handlers were stored in a single slot and run synchronously.
- `then` must return a new promise whose fate depends on the handler's return value or throw.
- If a handler returns a thenable, the outer promise adopts its result (the `adopt` function below).
- Handlers added after settling must still run, so `then` checks the current state.
```js
class MyPromise {
  #state = "pending";
  #value;
  #handlers = [];

  constructor(executor) {
    const settle = (state, value) => {
      if (this.#state !== "pending") return;
      this.#state = state;
      this.#value = value;
      this.#handlers.forEach((h) => this.#run(h));
      this.#handlers = [];
    };
    const adopt = (v) => {
      if (v === this) return settle("rejected", new TypeError("Chaining cycle detected"));
      if (v !== null && (typeof v === "object" || typeof v === "function")) {
        let then;
        try { then = v.then; } catch (e) { return settle("rejected", e); }
        if (typeof then === "function") {
          let called = false;
          try {
            then.call(
              v,
              (x) => { if (!called) { called = true; adopt(x); } },
              (r) => { if (!called) { called = true; settle("rejected", r); } }
            );
          } catch (e) {
            if (!called) { called = true; settle("rejected", e); }
          }
          return;
        }
      }
      settle("fulfilled", v);
    };
    let locked = false;
    const resolve = (v) => { if (!locked) { locked = true; adopt(v); } };
    const reject = (r) => { if (!locked) { locked = true; settle("rejected", r); } };
    try { executor(resolve, reject); } catch (e) { reject(e); }
  }

  #run({ onFulfilled, onRejected, resolve, reject }) {
    queueMicrotask(() => {
      const fulfilled = this.#state === "fulfilled";
      const cb = fulfilled ? onFulfilled : onRejected;
      if (typeof cb !== "function") return (fulfilled ? resolve : reject)(this.#value);
      try { resolve(cb(this.#value)); } catch (e) { reject(e); }
    });
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const handler = { onFulfilled, onRejected, resolve, reject };
      if (this.#state === "pending") this.#handlers.push(handler);
      else this.#run(handler);
    });
  }

  catch(onRejected) { return this.then(undefined, onRejected); }

  finally(fn) {
    return this.then(
      (v) => MyPromise.resolve(fn()).then(() => v),
      (r) => MyPromise.resolve(fn()).then(() => { throw r; })
    );
  }

  static resolve(v) { return v instanceof MyPromise ? v : new MyPromise((res) => res(v)); }
  static reject(r) { return new MyPromise((_, rej) => rej(r)); }
}

console.log("sync start");
new MyPromise((res) => setTimeout(() => res(1), 10))
  .then((v) => v + 1)
  .then((v) => { throw new Error("x" + v); })
  .catch((e) => e.message)
  .then(console.log);
MyPromise.resolve("now").then(console.log);
console.log("sync end");
```
**Output:** `sync start`, `sync end`, `now`, `x2`
**Interview tip:** In interviews, a smaller version (state, value, callback queue, chainable `then`) is acceptable; the key points are asynchronous handlers and `then` returning a new promise.

### 79. How do you implement Promise.all?

**Short answer:** Return a new promise that stores each result at its original index, counts remaining items, resolves when the count hits zero, and rejects on the first rejection.
- Wrap each item with `Promise.resolve` so plain values work.
- Use the index, not push order, so results keep input order.
- Handle an empty input by resolving immediately with `[]`.
```js
Promise.myAll = function (iterable) {
  return new Promise((resolve, reject) => {
    const items = Array.from(iterable);
    const results = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) return resolve(results);
    items.forEach((item, i) => {
      Promise.resolve(item).then((value) => {
        results[i] = value;
        if (--remaining === 0) resolve(results);
      }, reject);
    });
  });
};

const later = (v, ms) => new Promise((r) => setTimeout(() => r(v), ms));
Promise.myAll([later("a", 20), 2, later("c", 5)]).then(console.log);
Promise.myAll([later("a", 20), Promise.reject("err")]).catch(console.log);
```
**Output:** `err`, then `[ 'a', 2, 'c' ]`
**Interview tip:** Most bugs are using `push` (wrong order) or comparing `results.length` instead of a counter (breaks with holes).

### 80. How do you implement Promise.allSettled?

**Short answer:** Same as `all`, but convert every outcome into a `{status, value}` or `{status, reason}` object and resolve after all have settled; it never rejects.
- Attach both a fulfil and a reject handler to each item.
- Store results by index.
- Resolve once the counter reaches zero.
```js
Promise.myAllSettled = function (iterable) {
  return new Promise((resolve) => {
    const items = Array.from(iterable);
    const results = new Array(items.length);
    let remaining = items.length;
    if (remaining === 0) return resolve(results);
    const done = (i, outcome) => {
      results[i] = outcome;
      if (--remaining === 0) resolve(results);
    };
    items.forEach((item, i) => {
      Promise.resolve(item).then(
        (value) => done(i, { status: "fulfilled", value }),
        (reason) => done(i, { status: "rejected", reason })
      );
    });
  });
};

Promise.myAllSettled([Promise.resolve(1), Promise.reject("no"), 3])
  .then((r) => console.log(JSON.stringify(r)));
```
**Output:** `[{"status":"fulfilled","value":1},{"status":"rejected","reason":"no"},{"status":"fulfilled","value":3}]`
**Interview tip:** The only difference from `all` is the rejection handler that records instead of rejecting.

### 81. How do you implement Promise.race?

**Short answer:** Return a promise that subscribes `resolve` and `reject` to every item; whichever settles first wins because later calls are ignored.
- Wrap items in `Promise.resolve` so plain values count as already settled.
- An empty input leaves the promise pending forever, matching native behavior.
- No counters are needed because a promise settles only once.
```js
Promise.myRace = function (iterable) {
  return new Promise((resolve, reject) => {
    for (const item of iterable) Promise.resolve(item).then(resolve, reject);
  });
};

const later = (v, ms) => new Promise((r) => setTimeout(() => r(v), ms));
Promise.myRace([later("slow", 30), later("fast", 10)]).then(console.log);
```
**Output:** `fast`
**Interview tip:** Offer the timeout use case: `Promise.race([fetch(url), timeoutPromise])`.

### 82. How do you implement Promise.any?

**Short answer:** Resolve with the first fulfilment; collect rejection reasons by index and, if all items reject, reject with an `AggregateError` containing them.
- Mirror image of `all`: first success wins, all failures lose.
- Empty input rejects immediately with `AggregateError`.
- `AggregateError(errors, message)` is the built-in error type for this.
```js
Promise.myAny = function (iterable) {
  return new Promise((resolve, reject) => {
    const items = Array.from(iterable);
    const errors = new Array(items.length);
    let remaining = items.length;
    const fail = () => reject(new AggregateError(errors, "All promises were rejected"));
    if (remaining === 0) return fail();
    items.forEach((item, i) => {
      Promise.resolve(item).then(resolve, (reason) => {
        errors[i] = reason;
        if (--remaining === 0) fail();
      });
    });
  });
};

Promise.myAny([Promise.reject("a"), new Promise((r) => setTimeout(() => r("b"), 10))]).then(console.log);
Promise.myAny([Promise.reject("a"), Promise.reject("b")]).catch((e) => console.log(e.name, e.errors));
```
**Output:** `AggregateError [ 'a', 'b' ]`, then `b`
**Interview tip:** Explain `any` as `all` with success and failure swapped.

### 83. How do you implement Array.prototype.flat?

**Short answer:** Walk the array recursively; when an element is an array and depth remains, flatten it with `depth - 1`, otherwise push it; default depth is 1 and `Infinity` flattens fully.
- Holes are skipped, like the native method.
- Do not mutate the original array.
- An iterative stack version avoids recursion limits on very deep nesting.
```js
Array.prototype.myFlat = function (depth = 1) {
  const result = [];
  const walk = (arr, d) => {
    for (let i = 0; i < arr.length; i++) {
      if (!(i in arr)) continue;
      const el = arr[i];
      if (Array.isArray(el) && d > 0) walk(el, d - 1);
      else result.push(el);
    }
  };
  walk(this, depth);
  return result;
};

const nested = [1, [2, [3, [4]]]];
console.log(JSON.stringify(nested.myFlat()), JSON.stringify(nested.myFlat(Infinity)));
```
**Output:** `[1,2,[3,[4]]] [1,2,3,4]`
**Interview tip:** Offer the one-liner `arr.toString().split(",")` only as a trick and say why it fails (it stringifies everything).

## Part 12: Coding Questions

### 84. Implement debounce (with cancel and optional leading edge).
**Short answer:** Keep a timer id in a closure; each call clears the previous timer and schedules a new one, so only the last call within the delay runs.
- Preserve `this` and arguments with `fn.apply(this, args)`.
- Leading mode fires immediately on the first call, then ignores calls until the pause.
- Expose `cancel()` for cleanup on unmount.
```js
function debounce(fn, delay, { leading = false } = {}) {
  let timer = null;
  function debounced(...args) {
    const callNow = leading && timer === null;
    clearTimeout(timer);
    timer = setTimeout(() => {
      timer = null;
      if (!leading) fn.apply(this, args);
    }, delay);
    if (callNow) fn.apply(this, args);
  }
  debounced.cancel = () => { clearTimeout(timer); timer = null; };
  return debounced;
}
const trailing = debounce((x) => console.log("trailing", x), 20);
trailing(1); trailing(2); trailing(3);
const lead = debounce((x) => console.log("leading", x), 20, { leading: true });
lead("a"); lead("b");
const c = debounce(() => console.log("never"), 20); c(); c.cancel();
```
**Output:** `leading a` then `trailing 3`
**Interview tip:** Ask whether leading or trailing edge is wanted; interviewers reward that question.

### 85. Implement throttle.
**Short answer:** Record the last execution time; run the function only if at least `limit` ms have passed, optionally scheduling a trailing call with the latest arguments.
- The trailing call ensures the final state is not lost.
- Use `Date.now()` for simplicity or a timer for the trailing edge.
```js
function throttle(fn, limit) {
  let last = 0, timer = null, lastArgs;
  return function (...args) {
    const now = Date.now();
    lastArgs = args;
    if (now - last >= limit) {
      last = now;
      fn.apply(this, args);
    } else if (!timer) {
      timer = setTimeout(() => {
        last = Date.now(); timer = null;
        fn.apply(this, lastArgs);
      }, limit - (now - last));
    }
  };
}
const t = throttle((x) => console.log("run", x), 50);
t(1); t(2); t(3);
setTimeout(() => t(4), 20);
setTimeout(() => t(5), 120);
```
**Output:** `run 1`, `run 4`, `run 5` (one per line)
**Interview tip:** Describe the trailing call and why it matters for the last scroll position.

### 86. Implement memoize (with a cache key for multiple arguments).
**Short answer:** Wrap the function with a cache keyed on the arguments; return the cached result when the key exists.
- Use `JSON.stringify(args)` for simple keys; use a Map tree or WeakMap for object arguments.
- Only memoize pure functions, and bound the cache size to avoid memory growth.
- Use `cache.has(key)` rather than a truthy check so falsy results are cached.
```js
function memoize(fn) {
  const cache = new Map();
  return function (...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}
let calls = 0;
const slowSquare = (n) => { calls++; return n * n; };
const fast = memoize(slowSquare);
fast(4); fast(4); fast(5);
console.log(calls);
const fib = memoize((n) => (n < 2 ? n : fib(n - 1) + fib(n - 2)));
console.log(fib(50));
```
**Output:** `2` then `12586269025`
**Interview tip:** Mention cache-size limits (LRU) and that only pure functions are safe to memoize.

### 87. Implement once (run a function only one time).
**Short answer:** Keep a flag and the first result in a closure; later calls return the stored result without calling the function.
- Use cases: initialisation, analytics events, one-time setup.
- Cache the result so later calls return the same value.
```js
function once(fn) {
  let called = false, result;
  return function (...args) {
    if (!called) { called = true; result = fn.apply(this, args); }
    return result;
  };
}
const init = once((x) => { console.log("init", x); return x * 2; });
console.log(init(5), init(100));
```
**Output:** `init 5` then `10 10`
**Interview tip:** Mention that the result is cached so later calls return the same value.

### 88. Implement an EventEmitter (on, off, once, emit).
**Short answer:** Store listeners in a Map from event name to a list; `emit` calls each listener, `once` wraps the listener so it removes itself.
- Copy the listener array before iterating so removal during emit is safe.
- `off` should also remove `once`-wrapped listeners (keep a reference to the original).
```js
class EventEmitter {
  #events = new Map();
  on(name, fn) { (this.#events.get(name) ?? this.#events.set(name, []).get(name)).push(fn); return this; }
  off(name, fn) {
    const list = this.#events.get(name);
    if (list) this.#events.set(name, list.filter((l) => l !== fn && l.original !== fn));
    return this;
  }
  once(name, fn) {
    const wrapper = (...args) => { this.off(name, wrapper); fn(...args); };
    wrapper.original = fn;
    return this.on(name, wrapper);
  }
  emit(name, ...args) {
    const list = this.#events.get(name) ?? [];
    [...list].forEach((fn) => fn(...args));
    return list.length > 0;
  }
}
const e = new EventEmitter();
const hi = (n) => console.log("hi", n);
e.on("greet", hi); e.once("greet", (n) => console.log("once", n));
e.emit("greet", "A"); e.emit("greet", "B");
e.off("greet", hi);
console.log(e.emit("greet", "C"));
```
**Output:** `hi A`, `once A`, `hi B`, then `false`
**Interview tip:** Mention copying the array before iterating, and the once/off interaction.

### 89. Implement deepEqual.
**Short answer:** Compare primitives with `Object.is`, then compare types, key counts, and each key recursively; handle arrays, Dates, and cycles if asked.
- Short-circuit on same reference.
- Check `Object.keys` lengths and that every key exists in both.
- Treat `NaN` as equal to itself with `Object.is`.
```js
function deepEqual(a, b, seen = new WeakMap()) {
  if (Object.is(a, b)) return true;
  if (typeof a !== "object" || typeof b !== "object" || a === null || b === null) return false;
  if (Object.getPrototypeOf(a) !== Object.getPrototypeOf(b)) return false;
  if (a instanceof Date) return a.getTime() === b.getTime();
  if (seen.get(a) === b) return true;
  seen.set(a, b);
  const ka = Reflect.ownKeys(a), kb = Reflect.ownKeys(b);
  if (ka.length !== kb.length) return false;
  return ka.every((k) => Object.hasOwn(b, k) && deepEqual(a[k], b[k], seen));
}
console.log(deepEqual({ a: [1, { b: NaN }] }, { a: [1, { b: NaN }] }));
console.log(deepEqual({ a: 1 }, { a: 1, b: undefined }), deepEqual([1, 2], { 0: 1, 1: 2 }));
console.log(deepEqual(new Date(5), new Date(5)));
```
**Output:** `true` then `false false` then `true`
**Interview tip:** Mention key-count checks, NaN via `Object.is`, and cycle handling.

### 90. Implement a function that retries a promise-returning task N times (with delay).
**Short answer:** Loop up to N attempts, `await` the task in try/catch, wait with a delay between failures, and throw the last error when attempts run out.
- Use exponential backoff in real code (delay doubles) plus jitter.
- Only retry transient errors (network, 5xx), not 4xx.
```js
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
async function retry(task, attempts = 3, delay = 10) {
  let lastErr;
  for (let i = 1; i <= attempts; i++) {
    try { return await task(i); }
    catch (err) {
      lastErr = err;
      if (i < attempts) await sleep(delay * 2 ** (i - 1));
    }
  }
  throw lastErr;
}
(async () => {
  console.log(await retry(async (n) => { if (n < 3) throw new Error("flaky " + n); return "ok on " + n; }));
  try { await retry(async (n) => { throw new Error("always " + n); }, 2); } catch (e) { console.log(e.message); }
})();
```
**Output:** `ok on 3` then `always 2`
**Interview tip:** Mention exponential backoff with jitter and not retrying 4xx errors.

### 91. Split an array into chunks of size N.
**Short answer:** Step through the array with a stride of N and slice each segment.
- Handle a last shorter chunk naturally via `slice`.
- Validate `size > 0` to avoid infinite loops.
```js
function chunk(arr, size) {
  if (!Number.isInteger(size) || size < 1) throw new RangeError("size must be a positive integer");
  const out = [];
  for (let i = 0; i < arr.length; i += size) out.push(arr.slice(i, i + size));
  return out;
}
console.log(chunk([1, 2, 3, 4, 5], 2), chunk([], 3));
try { chunk([1], 0); } catch (e) { console.log(e.name); }
```
**Output:** `[ [ 1, 2 ], [ 3, 4 ], [ 5 ] ] []` then `RangeError`
**Interview tip:** Validate `size` and mention using `Array.from` with `Math.ceil(len / size)` as the one-liner alternative.

### 92. Implement groupBy (and know the built-in Object.groupBy).
**Short answer:** Reduce the array into an object whose keys come from a callback and whose values are arrays of matching items.
- Modern runtimes provide `Object.groupBy(items, fn)` (plain object) and `Map.groupBy`.
- Create the target with `Object.create(null)` to avoid prototype-key collisions.
```js
const groupBy = (arr, fn) =>
  arr.reduce((acc, item) => {
    const key = typeof fn === "function" ? fn(item) : item[fn];
    (acc[key] ||= []).push(item);
    return acc;
  }, Object.create(null));
const people = [{ n: "A", age: 20 }, { n: "B", age: 30 }, { n: "C", age: 20 }];
const byAge = groupBy(people, "age");
console.log(Object.keys(byAge), byAge[20].map((p) => p.n));
console.log(typeof Object.groupBy, Object.groupBy([1, 2, 3], (x) => (x % 2 ? "odd" : "even")).odd);
```
**Output:** `[ '20', '30' ] [ 'A', 'C' ]` then `function [ 1, 3 ]`
**Interview tip:** Name `Object.groupBy` to show you track the language, then still write the reduce version.

## Part 13: Output-Based Trick Questions

### 93. What does ["1", "2", "3"].map(parseInt) return?
**Short answer:** `[1, NaN, NaN]`, because `map` passes (value, index, array) and `parseInt` treats the index as the radix.
- `parseInt("1", 0)` uses radix 10 so it returns 1.
- `parseInt("2", 1)` is invalid (radix 1) so NaN; `parseInt("3", 2)` is NaN because 3 is not a binary digit.
- Fix with `map(Number)` or `map((s) => parseInt(s, 10))`.
```js
console.log(["1", "2", "3"].map(parseInt));
console.log(["1", "2", "3"].map(Number), ["1", "2", "3"].map((s) => parseInt(s, 10)));
```
**Output:** `[ 1, NaN, NaN ]` then `[ 1, 2, 3 ] [ 1, 2, 3 ]`
**Interview tip:** Explain the extra callback arguments; that is the lesson.

### 94. What is the output of the default sort on numbers?
**Short answer:** Default `sort()` compares elements as strings, so `[10, 9, 1].sort()` gives `[1, 10, 9]`.
- Pass a comparator `(a, b) => a - b` for numeric order.
- `sort` mutates the array in place; `toSorted` returns a copy.
- `undefined` values always sort to the end.
```js
console.log([10, 9, 1].sort(), [10, 9, 1].sort((a, b) => a - b));
console.log([3, undefined, 1].sort(), ["b", "a", "B"].sort());
```
**Output:** `[ 1, 10, 9 ] [ 1, 9, 10 ]` then `[ 1, 3, undefined ] [ 'B', 'a', 'b' ]`
**Interview tip:** Mention that sort mutates and offers `toSorted` as the copy.

### 95. What does this chained assignment print?
**Short answer:** `a.x = a = { n: 2 }` evaluates the left reference `a.x` first (against the old object), so the old object gets `x`, while `a` now points to the new object.
- The property target is resolved before the right side executes.
- `b` still references the old object, which now has `x`.
```js
let a = { n: 1 };
const b = a;
a.x = a = { n: 2 };
console.log(a.x, b.x, b);
```
**Output:** `undefined { n: 2 } { n: 1, x: { n: 2 } }`
**Interview tip:** Say "left-hand reference is evaluated first".

### 96. What is the output when objects are used as object keys?
**Short answer:** Object keys are converted to strings, so every plain object becomes `"[object Object]"` and later assignments overwrite earlier ones.
- Use a `Map` when keys should be objects.
```js
const a = {}, b = { key: "b" }, c = { key: "c" };
a[b] = 123;
a[c] = 456;
console.log(a[b], Object.keys(a));
```
**Output:** `456 [ '[object Object]' ]`
**Interview tip:** Offer Map or WeakMap as the fix.

### 97. In what order does this async/await code print?
**Short answer:** Code after `await` is a microtask continuation, so it runs after the current synchronous code and before timers.
- `async` functions run synchronously until the first `await`.
- The awaited value is wrapped in a resolved promise; the continuation runs in a microtask.
```js
async function a1() { console.log("a1 start"); await a2(); console.log("a1 end"); }
async function a2() { console.log("a2"); }
console.log("script start");
setTimeout(() => console.log("timeout"), 0);
a1();
new Promise((res) => { console.log("promise executor"); res(); }).then(() => console.log("promise then"));
console.log("script end");
```
**Output:** `script start`, `a1 start`, `a2`, `promise executor`, `script end`, `a1 end`, `promise then`, `timeout` (one per line)
**Interview tip:** Say "everything after await is a microtask" and walk through line by line.

### 98. What does a return in try/finally produce?
**Short answer:** `finally` always runs, and a `return` inside `finally` overrides the value returned from `try` or `catch`.
- Avoid returning from `finally`; it also swallows thrown errors.
```js
function f() { try { return "try"; } finally { console.log("finally runs"); } }
function g() { try { return "try"; } finally { return "finally"; } }
function h() { try { throw new Error("lost"); } finally { return "swallowed"; } }
console.log(f(), g(), h());
```
**Output:** `finally runs` then `try finally swallowed`
**Interview tip:** Mention that finally can swallow errors; that is why it is a lint rule.

### 99. Why is [] == ![] true?
**Short answer:** `![]` is `false`, then `[] == false` converts both sides to numbers: `[]` becomes `""` then 0, and `false` becomes 0.
- Step by step: `![]` gives false; false becomes 0; `[]` becomes `""` then 0; `0 == 0` is true.
- It shows why `===` is preferred.
```js
console.log(![], [] == ![], [] == false, [] === false);
console.log(Number([]), Number([5]), Number([1, 2]), Number({}));
```
**Output:** `false true true false` then `0 5 NaN NaN`
**Interview tip:** Show the conversion steps rather than memorising the answer.

## Part 14: Browser, Rendering & Performance

### 100. How does a browser turn HTML, CSS and JS into pixels (critical rendering path)?

**Short answer:** It builds the DOM from HTML and the CSSOM from CSS, runs JavaScript that may change either, combines them into a render tree of visible elements, computes layout (size and position), paints pixels into layers, composites the layers and draws them to the screen.
- 1. Parse HTML into the DOM (incrementally, as bytes arrive).
- 2. Parse CSS into the CSSOM (CSS blocks rendering, not parsing).
- 3. Run JavaScript; scripts can read or modify the DOM and CSSOM, so a classic script blocks the parser.
- 4. Combine into the render tree (elements with `display: none` and `<head>` are excluded).
- 5. Style and layout (reflow): compute the geometry of each element.
- 6. Paint: fill in pixels for text, colours, borders and images in layers.
- 7. Composite: stack overlapping layers (often on the GPU) and display the result.
- The first meaningful paint cannot happen until the HTML, render-blocking CSS and render-blocking JS needed for initial render have arrived.
**Interview tip:** Name which steps are re-run on a change: layout changes redo layout, paint and composite; colour changes redo paint and composite; `transform`/`opacity` often redo only composite.

### 101. What is the difference between reflow and repaint?

**Short answer:** Reflow (layout) recalculates the size and position of elements after a geometry change; repaint redraws pixels after a visual change that does not affect layout. Reflow is more expensive and always forces a repaint.
- Reflow triggers: adding or removing DOM elements, changing width, height, margin, padding, font size, window resize, reading layout properties like `offsetHeight` after a write (forced synchronous layout).
- Repaint-only triggers: `color`, `background-color`, `visibility`, `box-shadow`, `outline`.
- Composite-only (cheapest): `transform` and `opacity` on their own layer.
- Avoid layout thrashing: batch all reads, then all writes; use `requestAnimationFrame` and CSS classes instead of many inline style changes.
```js
// Layout thrashing: each read forces a fresh layout after the previous write
items.forEach((el) => { el.style.width = el.offsetWidth + 10 + "px"; });

// Better: read all first, then write
const widths = items.map((el) => el.offsetWidth);
items.forEach((el, i) => { el.style.width = widths[i] + 10 + "px"; });
```
**Interview tip:** Mention `will-change` and `transform` animations as ways to move work to the compositor.

### 102. What is the difference between normal, async and defer scripts?

**Short answer:** A normal script blocks HTML parsing while it downloads and runs; `async` downloads in parallel and runs as soon as it arrives (pausing parsing at that moment); `defer` downloads in parallel and runs after the HTML is fully parsed, in document order.

| | Normal | async | defer |
|---|---|---|---|
| Download | blocks parser | parallel | parallel |
| Execution | immediately, blocking | as soon as downloaded, may pause parsing | after parsing, before `DOMContentLoaded` |
| Order preserved | yes | no | yes |
| Best for | rare, must-run-first code | independent scripts (analytics, ads) | app code that touches the DOM |

- Note: by default scripts load and execute synchronously in the order they appear.
- `defer` has no effect on inline scripts, and `type="module"` scripts are deferred by default.
- Put normal scripts at the end of `<body>`, or use `defer`, so they do not delay first render.
**Interview tip:** Use `async` only for scripts with no dependencies on each other or on the DOM being ready.

### 103. What are render-blocking and parser-blocking resources, and what is the preload scanner?

**Short answer:** Render-blocking resources (CSS, and JS in `<head>`) stop the browser from painting until they are processed; parser-blocking resources (synchronous scripts) stop the HTML parser itself. The preload scanner is a secondary parser that discovers upcoming resources while the main parser is blocked.
- CSS blocks rendering but not HTML parsing: the browser keeps building the DOM in the background.
- Synchronous JS is parser-blocking because it could change the DOM (`document.write`) or CSSOM, and it also waits for any pending CSS.
- The preload scanner looks ahead in the raw HTML for `<img>`, `<link>` and `<script>` and starts those downloads early.
- It cannot see resources injected by JavaScript or referenced from CSS (`background-image`), so those start late; use `<link rel="preload">`.
**Interview tip:** Keep important resources discoverable in the initial HTML so the preload scanner can find them.

### 104. How would you optimise the critical rendering path?

**Short answer:** Reduce the number of critical resources, their size and the number of round trips: inline or split critical CSS, defer non-critical CSS and JS, compress and minify, and preload what the first view needs.
- Minimise critical bytes: minify and compress (gzip or Brotli), remove unused CSS and JS.
- Reduce critical resources: inline small critical CSS, use `media` attributes so non-matching stylesheets do not block render.
- Shorten the path: `defer`/`async` scripts, `<link rel="preload">` for fonts and hero images, `preconnect` for required origins.
- Cut server time with caching and a CDN, which helps TTFB and LCP.
- Lazy-load below-the-fold images and code-split JavaScript.
**Interview tip:** Tie each tactic to a metric: preload and inlined CSS help LCP, `defer` helps both LCP and INP.

### 105. How does HTTP caching with Cache-Control work?

**Short answer:** The `Cache-Control` response header tells the browser whether and for how long it may reuse a response: `no-store` means never cache, `no-cache` means cache but revalidate with the server before every reuse, and `max-age=N` means reuse for N seconds without asking.
- `no-store`: nothing is written to the cache; use for sensitive data.
- `no-cache`: a stored copy may exist but must be validated (usually with `ETag` or `Last-Modified`) before use.
- `max-age=30`: for 30 seconds the browser serves from its memory or disk cache with no network request; afterwards it revalidates or refetches.
- `public`/`private` control whether shared caches (CDNs) may store it; `immutable` plus a long `max-age` suits fingerprinted assets like `app.3f9a1c.js`.
- `no-cache` is a common misnomer: it does not mean "do not cache".
**Interview tip:** Strategy: fingerprint static assets with long `max-age` and `immutable`, and serve HTML with `no-cache` so deploys are picked up immediately.

### 106. What is the difference between ETag and Last-Modified (conditional requests, 304)?

**Short answer:** Both enable revalidation: the server sends a validator (`ETag`, a content fingerprint, or `Last-Modified`, a timestamp), the browser sends it back in `If-None-Match` or `If-Modified-Since`, and the server replies `304 Not Modified` with no body if nothing changed.
- `ETag` is an identifier (usually a hash of the content) that must change whenever the content changes; it is more precise than a timestamp.
- `Last-Modified` has one-second resolution and can mismatch across servers.
- A 304 saves bandwidth, not the round trip.
- When both are present, `If-None-Match` takes priority.
- Hashing the current time (as in some examples) defeats the purpose because the validator would change on every request; hash the content or use a version number.
```js
const crypto = require("crypto");

function respond(reqHeaders, body, lastModified) {
  const etag = '"' + crypto.createHash("md5").update(body).digest("hex") + '"';
  const headers = { "cache-control": "no-cache", etag, "last-modified": lastModified.toUTCString() };
  const notModified = reqHeaders["if-none-match"]
    ? reqHeaders["if-none-match"] === etag
    : reqHeaders["if-modified-since"] &&
      lastModified <= new Date(reqHeaders["if-modified-since"]);
  return notModified ? { status: 304, headers } : { status: 200, headers, body };
}

const modified = new Date("2024-01-01T00:00:00Z");
const first = respond({}, "hello", modified);
console.log(first.status);
console.log(respond({ "if-none-match": first.headers.etag }, "hello", modified).status);
console.log(respond({ "if-none-match": first.headers.etag }, "changed", modified).status);
console.log(respond({ "if-modified-since": modified.toUTCString() }, "hello", modified).status);
```
**Output:** `200`, `304`, `200`, `304`
**Interview tip:** Mention the `Cache-Control: no-cache` plus `ETag` pairing, because a validator without a freshness policy leaves the browser guessing with heuristics.

### 107. What are Core Web Vitals?

**Short answer:** Web Vitals are Google's user-experience metrics; the Core ones are LCP (loading), INP (responsiveness) and CLS (visual stability), with TTFB as a supporting server metric.

| Metric | Measures | Good threshold |
|---|---|---|
| LCP (Largest Contentful Paint) | time until the largest visible element renders | 2.5 s or less |
| INP (Interaction to Next Paint) | latency of interactions across the whole visit | 200 ms or less |
| CLS (Cumulative Layout Shift) | unexpected layout movement | 0.1 or less |
| TTFB (Time to First Byte) | time until the first byte of the response arrives | about 0.8 s or less |
| FID (First Input Delay) | delay before the first interaction is handled (replaced by INP in March 2024) | 100 ms or less |

- Fix LCP with a faster server, preloading the hero image and cutting render-blocking resources.
- Fix INP by breaking up long tasks and reducing main-thread JavaScript.
- Fix CLS by setting width and height on images and reserving space for ads and embeds.
**Interview tip:** The notes list FID, which is deprecated; saying INP replaced it shows you are current.

## Part 15: Security and Miscellaneous

### 108. What is XSS and how do you prevent it?
**Short answer:** Cross-site scripting lets an attacker inject script that runs in your page's origin; prevent it by escaping output, avoiding raw HTML sinks, sanitising, and using a Content Security Policy.
- Types: stored (saved in DB), reflected (in URL/response), DOM-based (client-side sink).
- Dangerous sinks: `innerHTML`, `document.write`, `eval`, `dangerouslySetInnerHTML`, `javascript:` URLs.
- Use `textContent`, framework auto-escaping, and DOMPurify when HTML is needed.
- Add a strict CSP, `HttpOnly` cookies, and validate input on the server.
```js
const escapeHtml = (s) =>
  s.replace(/[&<>"']/g, (c) => ({ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" }[c]));
console.log(escapeHtml('<img src=x onerror="alert(1)">'));
```
**Output:** `&lt;img src=x onerror=&quot;alert(1)&quot;&gt;`
**Interview tip:** Name the three types and say "textContent over innerHTML, plus CSP".

### 109. What is CSRF and how do you prevent it?
**Short answer:** Cross-site request forgery tricks a logged-in user's browser into sending an authenticated request to your site from another site; prevent it with SameSite cookies and CSRF tokens.
- It works because browsers attach cookies automatically.
- Defences: `SameSite=Lax/Strict` cookies, anti-CSRF tokens, checking `Origin`/`Referer`, custom headers.
- Never change state with GET requests.
- CSRF abuses the cookie; XSS abuses script execution, and XSS defeats CSRF tokens.
```js
// Server-side token check sketch (Node)
const crypto = require("crypto");
const sessionToken = crypto.randomBytes(16).toString("hex");
const verify = (submitted) => submitted.length === sessionToken.length &&
  crypto.timingSafeEqual(Buffer.from(submitted), Buffer.from(sessionToken));
console.log(verify(sessionToken), verify("x".repeat(32)));
```
**Output:** `true false`
**Interview tip:** Contrast it with XSS and name SameSite as the modern first defence.

### 110. What is CORS and why do preflight requests happen?
**Short answer:** CORS is a browser mechanism that lets a server declare which other origins may read its responses; it relaxes the same-origin policy and is enforced by the browser, not the server.
- Origin is scheme + host + port.
- Simple requests (GET/POST with basic headers) are sent directly; the browser then checks `Access-Control-Allow-Origin`.
- Non-simple requests (PUT/DELETE, JSON content-type, custom headers) trigger an OPTIONS preflight first.
- Credentials need `Access-Control-Allow-Credentials: true` and a specific origin (not `*`).
- CORS errors do not stop curl or Postman; they only protect browser users.
```js
const sameOrigin = (a, b) => {
  const x = new URL(a), y = new URL(b);
  return x.protocol === y.protocol && x.host === y.host;
};
console.log(sameOrigin("https://app.com/a", "https://app.com/b"));
console.log(sameOrigin("https://app.com", "https://api.app.com"), sameOrigin("http://app.com", "https://app.com"));
```
**Output:** `true` then `false false`
**Interview tip:** Say "browser-enforced, server-configured" and explain the preflight trigger.

### 111. What are common causes of memory leaks in JavaScript?
**Short answer:** A leak happens when objects stay reachable but are no longer needed; common causes are forgotten timers or listeners, accidental globals, detached DOM nodes, and unbounded caches.
- Uncleared `setInterval` or event listeners keep closures (and everything they capture) alive.
- Accidental globals from missing `let/const` never get collected.
- Closures holding large objects, and Maps used as caches that never evict.
- Detect with Chrome DevTools heap snapshots and the Performance memory timeline.
- Fix: clean up in unmount/dispose, use `WeakMap`/`WeakRef`, bound caches, and `AbortController` for listeners.
```js
const bigHolders = [];
function leaky() { const big = new Array(1e5).fill("x"); bigHolders.push(() => big.length); }
leaky(); leaky();
console.log(bigHolders.length, bigHolders[0]());
const timer = setInterval(() => {}, 1000);
clearInterval(timer);
console.log("timer cleared");
```
**Output:** `2 100000` then `timer cleared`
**Interview tip:** List the four causes (timers, listeners, globals, detached DOM) and the DevTools tool.

### 112. How does garbage collection work in JavaScript?
**Short answer:** Engines use automatic mark-and-sweep: starting from roots (globals, the stack), they mark everything reachable and free the rest.
- Reference counting is not used by modern engines because it cannot collect cycles.
- V8 is generational: a young generation for short-lived objects (scavenge) and an old generation (mark-sweep-compact).
- GC runs incrementally and concurrently to reduce pauses; timing is not controllable.
- `WeakRef` and `FinalizationRegistry` exist but their callbacks are not guaranteed to run.
```js
let obj = { data: new Array(10).fill(0) };
const ref = new WeakRef(obj);
console.log(ref.deref() === obj);
obj = null; // now unreachable; collection timing is up to the engine
console.log(typeof ref.deref());
```
**Output:** `true` then `object`
**Interview tip:** Say "reachability, not reference counting" and mention cycles are fine for mark-and-sweep.

## Part 16: Browser & React internals

### 113. How does React update the DOM (virtual DOM and reconciliation)?

**Short answer:** React keeps a lightweight tree of JavaScript objects (the virtual DOM) describing the UI; on a state or props change it renders a new tree, diffs it against the previous one (reconciliation) and applies only the minimal changes to the real DOM.
- Initial render: components run, returning React elements, and React builds the real DOM from them.
- Update trigger: `setState` or a new prop schedules a re-render of that component and its children.
- Reconciliation (diffing): React compares old and new trees; different element types mean replace the subtree, same types mean update props, and `key`s identify moved list items.
- Commit: the computed changes (patches) are applied to the real DOM in one batch.
- After the commit the browser still runs its own style, layout (reflow) and paint steps.
**Interview tip:** The virtual DOM is not the speed secret; batching and minimal DOM writes are, and React 18 added automatic batching and concurrent rendering on top.

### 114. What does React.memo do?

**Short answer:** `React.memo` is a higher-order component that skips re-rendering a component when its props are shallowly equal to the previous props.
- It compares props with `Object.is` per prop, so new object, array or function props defeat it.
- Stabilise props with `useMemo` and `useCallback`, or pass a custom comparison function as the second argument.
- It does not stop re-renders caused by the component's own state or context changes.
- Use it for expensive components that re-render often with unchanged props, not by default everywhere.
```js
const Row = React.memo(function Row({ item, onSelect }) {
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});
```
**Interview tip:** If memo "does not work", check whether the parent passes an inline arrow function or object literal each render.

