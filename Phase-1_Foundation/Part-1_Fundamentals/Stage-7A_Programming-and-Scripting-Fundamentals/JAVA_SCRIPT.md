# JavaScript Security, APIs & Automation — Practice Questions

Use this as a learning checklist. Solve each question yourself, test edge cases, and keep solutions in clearly named files. For security exercises, use only systems, accounts, and applications you own or are explicitly authorized to test.

---

## 1. Core JavaScript Fundamentals

### 1.1 Variables, Data Types, and Operators

- [ ] Declare variables using `var`, `let`, and `const`; explain the difference in scope, hoisting, and re-assignment rules for each.
- [ ] Identify the seven primitive types (`string`, `number`, `bigint`, `boolean`, `undefined`, `null`, `symbol`) and the one non-primitive (`object`); use `typeof` on each.
- [ ] Explain why `typeof null === 'object'` is a historical quirk and how to correctly check for `null`.
- [ ] Take two numbers as input and display their addition, subtraction, multiplication, division, floor division (`Math.floor`), modulus, and exponentiation (`**`).
- [ ] Demonstrate the difference between `==` (loose equality with coercion) and `===` (strict equality); give three examples where `==` produces a surprising result.
- [ ] Explain type coercion: predict the result of `'5' + 3`, `'5' - 3`, `true + 1`, `null + 1`, and `undefined + 1` before running the code.
- [ ] Use the logical operators `&&`, `||`, and `??` (nullish coalescing); explain how `??` differs from `||` when a value is `0` or `''`.
- [ ] Swap two variables without a temporary variable using destructuring assignment.
- [ ] Demonstrate bitwise operators (`&`, `|`, `^`, `~`, `<<`, `>>`, `>>>`) on small integers and explain each result.
- [ ] Use augmented assignment operators (`+=`, `-=`, `*=`, `/=`, `%=`, `**=`) inside a loop.
- [ ] Use the optional chaining operator (`?.`) to safely access a nested property that may not exist; compare with a manual `&&` guard.
- [ ] Use the logical assignment operators (`&&=`, `||=`, `??=`) and explain what each one does in one sentence.

### 1.2 Control Flow

- [ ] Write an `if / else if / else` chain that converts a numeric score to a grade (A, B, C, D, F).
- [ ] Use a `switch` statement to map HTTP status codes (200, 301, 400, 401, 403, 404, 500) to human-readable descriptions; include a `default` case.
- [ ] Print the multiplication table for a number from 1 to 10 using a `for` loop.
- [ ] Use `while` to keep prompting until valid numeric input is provided; handle `NaN` explicitly.
- [ ] Use `do...while` to show a menu at least once before checking whether to exit.
- [ ] Use `break` to exit a loop early and `continue` to skip a specific iteration; give a practical example of each.
- [ ] Use `for...of` to iterate over an array and `for...in` to iterate over an object's own enumerable keys; explain why `for...in` on an array is unreliable.
- [ ] Use `Array.prototype.forEach` on an array and explain why you cannot `break` out of it — contrast with `for...of`.
- [ ] Use labeled statements with `break` to exit a nested loop; explain when this is cleaner than a flag variable.

### 1.3 Strings and Template Literals

- [ ] Access the first, last, and middle characters of a string using index notation and the `.at()` method.
- [ ] Reverse a string and check whether it is a palindrome.
- [ ] Use `toUpperCase`, `toLowerCase`, `trim`, `trimStart`, `trimEnd`, `replace`, `replaceAll`, `split`, `join`, `startsWith`, `endsWith`, `includes`, `indexOf`, and `padStart` in meaningful examples.
- [ ] Count vowels, consonants, digits, spaces, and special characters in a user-supplied string.
- [ ] Use template literals (backticks) to embed expressions, multiline text, and a tagged template; explain the advantage over string concatenation.
- [ ] Use `String.prototype.matchAll` with a regex to extract all occurrences of a pattern from a string.
- [ ] Explain the difference between a character and a code point; use `String.prototype.codePointAt` and `String.fromCodePoint` on an emoji character.
- [ ] Use `padStart` to format numbers with leading zeros (e.g., `'007'`) and `padEnd` to align tabular output.

### 1.4 Functions

- [ ] Write named functions, function expressions, and arrow functions for the same logic; explain how `this` behaves differently in each.
- [ ] Explain function hoisting: show that a named function declaration can be called before its definition, while a `const` arrow function cannot.
- [ ] Use default parameters, rest parameters (`...args`), and the spread operator (`...`) in function calls.
- [ ] Explain the difference between `arguments` (available in regular functions) and rest parameters; show why `arguments` is unavailable in arrow functions.
- [ ] Write a higher-order function that accepts a callback and calls it with a result; then use it with `map`, `filter`, and `reduce`.
- [ ] Use `Array.prototype.map` to transform an array, `filter` to keep matching elements, and `reduce` to aggregate — chain all three in one expression.
- [ ] Write an immediately invoked function expression (IIFE) and explain why it was used for encapsulation before ES modules.
- [ ] Write a memoization utility using a closure to cache return values of an expensive pure function.
- [ ] Write a function that uses `bind`, `call`, and `apply` to explicitly set `this`; explain when each is appropriate.
- [ ] Explain the difference between a pure function and a function with side effects; refactor a side-effecting function to be pure.

### 1.5 Arrays

- [ ] Create an array, access elements by index, use negative indexing with `.at()`, and explain that JavaScript arrays are zero-indexed objects.
- [ ] Use `push`, `pop`, `shift`, `unshift`, `splice`, `slice`, `concat`, `flat`, `flatMap`, `indexOf`, `findIndex`, `find`, `includes`, `every`, `some`, and `fill` in practical examples.
- [ ] Remove duplicates from an array using a `Set`; preserve original order.
- [ ] Sort an array of numbers correctly (pass a comparator — explain why the default sort is alphabetical and wrong for numbers).
- [ ] Sort an array of objects by a string property and separately by a numeric property.
- [ ] Destructure an array: unpack the first two elements into variables and collect the rest with `...rest`.
- [ ] Use `Array.from` to convert a `NodeList`, a `Set`, and a string into arrays.
- [ ] Use `Array.isArray` to distinguish an array from a plain object; explain why `typeof []` is misleading.
- [ ] Build a 2D array (matrix), iterate its rows and columns with nested loops, and transpose it.
- [ ] Use `reduce` to group an array of objects by a key property.

### 1.6 Objects

- [ ] Create an object literal with properties and methods; access properties with dot notation and bracket notation; explain when bracket notation is required.
- [ ] Use shorthand property names, computed property names, and method shorthand in an object literal.
- [ ] Use destructuring assignment to extract properties from an object, rename them, and provide defaults.
- [ ] Use the spread operator (`...obj`) to shallow-clone an object and to merge two objects; explain what a shallow clone means for nested objects.
- [ ] Use `Object.keys`, `Object.values`, `Object.entries`, `Object.assign`, `Object.freeze`, and `Object.create` in meaningful examples.
- [ ] Check whether a property is an object's own property using `hasOwnProperty` and `Object.hasOwn`; explain why the latter is safer.
- [ ] Iterate over an object's key-value pairs using `for...in` with a `hasOwnProperty` guard and with `Object.entries`.
- [ ] Write a deep equality check for two plain objects without using a library; handle nested objects and arrays.
- [ ] Explain what happens when you use an object as a `Map` key; demonstrate that `Map` handles non-string keys correctly while plain objects do not.

### 1.7 Scope, Closures, and `this`

- [ ] Explain global scope, function scope, and block scope (`let`/`const`); demonstrate a variable shadowing bug and fix it.
- [ ] Explain hoisting: show the difference in behavior between `var` declarations, `let`/`const` declarations (temporal dead zone), and function declarations.
- [ ] Write a closure that creates a counter with private state; explain why the inner function retains access to the outer function's variables after the outer function has returned.
- [ ] Demonstrate the classic `var` in a loop bug (all callbacks reference the same `i`); fix it with `let` and separately with a closure using an IIFE.
- [ ] Explain what `this` refers to in: a regular function, an arrow function, a method call, a constructor call, and after `bind`.
- [ ] Write a class method that loses `this` when passed as a callback; fix it using `bind`, an arrow function, or a class field arrow.
- [ ] Explain the prototype chain: show that properties and methods are looked up through `__proto__` links, not copied into each object.

### 1.8 Object-Oriented Programming

- [ ] Create an ES6 `class` with a `constructor`, instance methods, and a `toString` method; instantiate it and call its methods.
- [ ] Add `static` methods and properties to a class; explain that they belong to the class itself, not to instances.
- [ ] Use `get` and `set` accessors to control read/write access to a private-style property (use the `#` private field syntax).
- [ ] Create a `Person` parent class and an `Employee` child class using `extends` and `super`; override a method and call the parent's version.
- [ ] Explain why JavaScript uses prototypal inheritance under the hood even when the `class` syntax is used.
- [ ] Use `instanceof` to check an object's class; explain its limitations when objects cross iframe/realm boundaries.
- [ ] Implement the Iterator protocol (`[Symbol.iterator]`) on a custom class so it can be used in a `for...of` loop.
- [ ] Implement a custom iterable that generates a Fibonacci sequence up to a limit, usable with `for...of` and the spread operator.
- [ ] Use `Symbol.toPrimitive` to control how an object is coerced to a string or number.

### 1.9 Asynchronous JavaScript

- [ ] Explain the event loop, call stack, task queue, and microtask queue; predict the console output order for a mix of `setTimeout`, `Promise.resolve`, and synchronous code.
- [ ] Write a function that returns a `Promise`; resolve it on success and reject it on failure; consume it with `.then().catch().finally()`.
- [ ] Rewrite the same Promise-based function using `async/await` with `try/catch/finally`; compare readability.
- [ ] Use `Promise.all` to run several async operations in parallel and collect all results; explain what happens if one rejects.
- [ ] Use `Promise.allSettled` to run several async operations and inspect each result regardless of success or failure.
- [ ] Use `Promise.race` to implement a timeout: race a fetch against a `Promise` that rejects after 3 seconds.
- [ ] Use `Promise.any` and explain how it differs from `Promise.race`.
- [ ] Write an async generator function that `yield`s values one at a time and consume it with `for await...of`.
- [ ] Explain the difference between `setTimeout(fn, 0)` and `queueMicrotask(fn)`; show which runs first and why.
- [ ] Demonstrate an unhandled Promise rejection; add a global `unhandledrejection` event listener to catch it.
- [ ] Chain Promises correctly: show how a `.then` returning a new Promise flattens into the chain (no nested `.then` calls).

### 1.10 Error Handling

- [ ] Use `try / catch / finally` to handle a runtime error; explain when `finally` runs even if `catch` also throws.
- [ ] Create a custom error class that extends `Error`; include a `name` and additional context in the constructor.
- [ ] Throw and catch specific error types using `instanceof`; explain why catching every error with a single handler can mask bugs.
- [ ] Handle both synchronous and asynchronous errors consistently: show that `try/catch` does NOT catch a rejection unless `await` is present.
- [ ] Explain `Error.prototype.stack`; log it in a caught error and describe what information it reveals.
- [ ] Write a safe JSON parser function that returns `{ ok: true, value }` on success and `{ ok: false, error }` on failure — no exceptions propagate.

### 1.11 Modules (ES Modules & CommonJS)

- [ ] Export functions and constants from an ES module using named exports and a default export; import them in another file.
- [ ] Explain the difference between `import` (ES modules, static, hoisted) and `require` (CommonJS, dynamic, synchronous).
- [ ] Use dynamic `import()` to lazy-load a module only when needed; explain the use case in large SPAs.
- [ ] Explain `import.meta.url` and how to derive the current file's directory from it in Node.js without `__dirname`.
- [ ] Re-export named exports from a barrel file (`index.js`) and explain the purpose of barrel files in large codebases.
- [ ] Explain circular module dependencies: demonstrate that they cause partially-initialized exports in CommonJS and how ES modules handle them differently.

### 1.12 Modern JavaScript Syntax (ES2018 – ES2024)

- [ ] Use optional chaining (`?.`) and nullish coalescing (`??`) together to safely access deeply nested API response fields with a fallback.
- [ ] Use `Promise.allSettled`, `Promise.any`, and `Promise.race` — write one example showing when each is the right choice.
- [ ] Use `Object.fromEntries` to reconstruct an object from an array of `[key, value]` pairs (the inverse of `Object.entries`).
- [ ] Use `Array.prototype.at(-1)` to get the last element and explain why it is cleaner than `arr[arr.length - 1]`.
- [ ] Use `structuredClone` to deep-clone an object and explain how it handles `Date`, `Map`, `Set`, and circular references — compare with `JSON.parse(JSON.stringify(...))`.
- [ ] Use `String.prototype.replaceAll` and explain how it differs from `replace` with a global regex.
- [ ] Use top-level `await` in an ES module and explain the environments that support it.
- [ ] Use `Array.prototype.toSorted`, `toReversed`, and `with` (non-mutating array methods, ES2023) and explain why immutability matters.
- [ ] Use `Object.groupBy` (ES2024) to group an array of objects by a property value without a `reduce`.

---

## 2. Standard Library & Built-ins Deep Dive

### 2.1 `Math` and `Number`

- [ ] Use `Math.floor`, `Math.ceil`, `Math.round`, `Math.trunc`, `Math.abs`, `Math.pow`, `Math.sqrt`, `Math.max`, `Math.min`, and `Math.log` in one script.
- [ ] Explain floating-point precision: show why `0.1 + 0.2 !== 0.3` and how to compare floats safely using a tolerance (`Number.EPSILON`).
- [ ] Use `Number.isInteger`, `Number.isFinite`, `Number.isNaN`, `Number.parseInt`, and `Number.parseFloat`; explain why these are preferred over global `isNaN` and `parseInt`.
- [ ] Explain `Number.MAX_SAFE_INTEGER` and `Number.MIN_SAFE_INTEGER`; use `BigInt` to handle integers beyond the safe range.
- [ ] Use `Math.random()` to simulate a dice roll (1–6); explain why `Math.random()` is not cryptographically secure.

### 2.2 `Date`

- [ ] Get the current date and time with `new Date()`; log the year, month, day, hour, minute, and second using instance methods.
- [ ] Calculate a person's age in years from a birth-date string.
- [ ] Add and subtract days from a date using timestamp arithmetic (milliseconds).
- [ ] Format a date as `YYYY-MM-DD` without a library; explain why `toLocaleDateString` output is locale-dependent.
- [ ] Use `Date.now()` to benchmark a block of code and compare it with `performance.now()` — explain the precision difference.

### 2.3 `Map` and `Set`

- [ ] Create a `Map`, set key-value pairs with non-string keys (objects, numbers), retrieve them, and iterate with `for...of`.
- [ ] Explain when to use `Map` over a plain object: insertion-order guarantee, non-string keys, and `.size` property.
- [ ] Create a `Set`, add duplicates, and observe that the size stays correct; convert between `Set` and `Array` in both directions.
- [ ] Use `Set` to compute union, intersection, and difference of two arrays without nested loops.
- [ ] Use `WeakMap` to attach private metadata to an object without preventing garbage collection; explain the difference from `Map`.

### 2.4 `RegExp`

- [ ] Write patterns to validate an email-like string, an IPv4 address, a date (`YYYY-MM-DD`), a URL scheme, and a JWT format.
- [ ] Use `RegExp.prototype.test`, `String.prototype.match`, `matchAll`, `replace` (with a function replacer), `search`, and `split`; describe how each differs.
- [ ] Explain flags: `g` (global), `i` (case-insensitive), `m` (multiline), `s` (dotAll), `u` (unicode), `d` (indices); demonstrate each with a short example.
- [ ] Use named capture groups (`(?<name>...)`) and reference them in a replacement string and via `match.groups`.
- [ ] Explain catastrophic backtracking with a vulnerable pattern like `(a+)+` against a long string of `a`s; rewrite it safely.
- [ ] Redact an API token or password from a log line using `String.prototype.replace` with a regex.

### 2.5 Iterators and Generators

- [ ] Write a generator function (`function*`) that yields Fibonacci numbers indefinitely; consume only the first 10 using a `for...of` loop with a counter.
- [ ] Use `yield*` to delegate to another generator and flatten two generators into one sequence.
- [ ] Write a generator-based range utility: `range(start, end, step)` usable in `for...of`.
- [ ] Use a generator to implement lazy pagination: yield one page of API results at a time without loading everything upfront.
- [ ] Explain the four states of a generator (`suspended`, `running`, `completed`, `closed`) and what `generator.return()` and `generator.throw()` do.

### 2.6 Proxy and Reflect

- [ ] Create a `Proxy` with a `get` trap that logs every property access on an object; explain the use case in security monitoring.
- [ ] Create a `Proxy` with a `set` trap that validates the type of a value before allowing it to be written.
- [ ] Use a `Proxy` to create a read-only view of an object that throws on any `set` or `deleteProperty` attempt.
- [ ] Explain the relationship between `Proxy` traps and `Reflect` methods; show why forwarding to `Reflect` in a trap is the correct pattern.
- [ ] Use a `Proxy` to detect Prototype Pollution attempts: intercept `set` operations where the key is `__proto__`, `constructor`, or `prototype`.

---

## 3. Node.js Core Modules

### 3.1 `fs` — File System

- [ ] Read a file synchronously with `fs.readFileSync` and asynchronously with `fs.promises.readFile`; compare error handling in both.
- [ ] Write text to a file with `fs.promises.writeFile` and append lines with `fs.promises.appendFile`; explain the difference between the two.
- [ ] List directory contents with `fs.promises.readdir`; use `{ withFileTypes: true }` to distinguish files from directories without an extra `stat` call.
- [ ] Walk a directory tree recursively using `fs.promises.readdir` with `{ recursive: true }` (Node 18+) and collect all `.js` file paths.
- [ ] Use `fs.createReadStream` to process a large file line-by-line without loading it entirely into memory.
- [ ] Check whether a path exists using `fs.promises.access` with `fs.constants.F_OK` and handle the `ENOENT` error.
- [ ] Use `path.join` and `path.resolve` to build file paths safely; explain how `path.resolve` prevents path traversal when combined with an allowlist check.

### 3.2 `path` Module

- [ ] Use `path.join`, `path.resolve`, `path.dirname`, `path.basename`, `path.extname`, and `path.parse` on a sample file path.
- [ ] Detect a path traversal attempt: check that a user-supplied path resolved with `path.resolve` still starts with the allowed base directory.
- [ ] Use `path.relative` to compute the relative path between two absolute paths.
- [ ] Explain the difference between POSIX paths and Windows paths; use `path.posix` to produce forward-slash paths on any platform.

### 3.3 `os` and `process`

- [ ] Use `os.platform`, `os.arch`, `os.hostname`, `os.homedir`, `os.tmpdir`, `os.cpus`, and `os.freemem` in a system-info script.
- [ ] Read environment variables with `process.env.VAR_NAME`; provide a safe default when a variable is missing; reject startup if a required variable is absent.
- [ ] Use `process.argv` to read command-line arguments; print a usage message when required arguments are missing.
- [ ] Use `process.exit(1)` to signal an error; explain the difference between exit code `0` (success) and non-zero (failure).
- [ ] Use `process.on('uncaughtException', ...)` and `process.on('unhandledRejection', ...)` to log unexpected errors before exiting; explain why silent swallowing is dangerous.

### 3.4 `events` Module

- [ ] Create an `EventEmitter`, register a listener with `.on`, emit an event with `.emit`, and remove the listener with `.off`.
- [ ] Use `.once` to register a one-time listener and explain when it is preferable to `.on`.
- [ ] Explain the `error` event convention: if an `EventEmitter` emits `'error'` and no listener is registered, Node.js throws — demonstrate this and add a handler.
- [ ] Use `EventEmitter.listenerCount` and `.eventNames` to inspect registered listeners; explain why too many listeners trigger a memory-leak warning.

### 3.5 `stream` Module (Awareness)

- [ ] Explain the four stream types: `Readable`, `Writable`, `Duplex`, and `Transform`; give a real-world example of each.
- [ ] Pipe a `fs.createReadStream` into a `fs.createWriteStream` to copy a file without loading it into memory.
- [ ] Use `stream.pipeline` (instead of `.pipe`) and explain why it handles backpressure and errors more reliably.
- [ ] Use a `Transform` stream to convert all bytes in a file to uppercase as they are piped through.

---

## 4. Mini Projects (Fundamentals)

- [ ] Build a CLI number-guessing game: generate a random number 1–100, accept guesses via `readline`, and report "Too High", "Too Low", or "Correct!" with a guess counter.
- [ ] Build a contact book: store contacts in a JSON file; support `add`, `search`, `list`, and `delete` commands via `process.argv`; validate all input before writing.
- [ ] Build a word frequency counter: read a text file, tokenize it, count each word, and print the top 10 most frequent words in a formatted table.
- [ ] Build a CSV parser from scratch (no libraries): read a `.csv` file with `fs`, split into rows and columns, and output an array of objects keyed by the header row.
- [ ] Build a file watcher using `fs.watch`: detect when any `.js` file in a directory changes, log the filename and event type, and run a linting command on it.
- [ ] Build a simple HTTP server using Node's built-in `http` module (no framework): serve static files from a directory, set correct `Content-Type` headers, and return `404` for missing files.
- [ ] Build an event-driven logging system: create a custom `EventEmitter` that emits `log`, `warn`, and `error` events; attach listeners that write to different output destinations.
- [ ] Refactor one of the above projects: separate it into modules (`utils.js`, `cli.js`, `core.js`), add JSDoc comments, and write Jest unit tests for the core logic.

---

## 5. HTTP & API Communication

### 5.1 `fetch` — Modern HTTP Requests

- [ ] Send a `GET` request with `fetch` to a public test API and log the parsed JSON response.
- [ ] Send a `POST` request with `fetch`, set `Content-Type: application/json`, and include a JSON body; inspect the response status and body.
- [ ] Handle `fetch` errors correctly — explain why `fetch` does **not** reject on 4xx/5xx responses and how to detect them using `response.ok`.
- [ ] Use `fetch` with `AbortController` to cancel a request after a 3-second timeout; log a meaningful message when the abort fires.
- [ ] Chain `.then()` and `.catch()` for a fetch request, then rewrite the same logic using `async/await` with `try/catch`; compare readability.
- [ ] Set custom request headers (`Authorization: Bearer <token>`, `Accept`, `X-Custom-Header`) on a `fetch` call.
- [ ] Use `fetch` with `credentials: 'include'` and explain when cookies are sent cross-origin and what CORS headers the server must return.
- [ ] Implement a retry wrapper around `fetch` that retries up to 3 times on network failure or 5xx status, with exponential back-off.

### 5.2 `axios` — API Requests and Interceptors

- [ ] Install `axios` and make a `GET` request; log response data, status, and headers.
- [ ] Create an `axios` instance with a `baseURL`, default `timeout`, and default `Authorization` header.
- [ ] Add a request interceptor that attaches a Bearer token to every outgoing request from an environment variable.
- [ ] Add a response interceptor that detects a `401` response, refreshes a token, and retries the original request automatically.
- [ ] Add a response interceptor that logs request duration for every call.
- [ ] Handle `AbortController` to cancel an in-flight request when a user navigates away.
- [ ] Use `axios` to upload a file with `multipart/form-data` and track upload progress using `onUploadProgress`.
- [ ] Write a centralized error-handling utility that differentiates network errors, timeouts, and HTTP error responses.

### 5.3 `WebSocket API` — Real-Time Communication

- [ ] Open a WebSocket connection to a public echo server, send a message, and log what the server echoes back.
- [ ] Handle all four WebSocket events: `onopen`, `onmessage`, `onerror`, and `onclose`.
- [ ] Implement a reconnection strategy with exponential back-off up to 5 attempts.
- [ ] Build a minimal client-side chat UI that sends messages via WebSocket and renders incoming messages in the DOM.
- [ ] Explain the difference between `ws://` and `wss://`; demonstrate why sending sensitive data over an unencrypted WebSocket is dangerous.
- [ ] Implement a heartbeat mechanism using `setInterval` to send a `ping` message every 30 seconds and detect dead connections.
- [ ] Describe three WebSocket security risks: missing origin validation, missing authentication, and message injection; propose mitigations for each.

### 5.4 `EventSource (SSE)` — Server-Sent Events

- [ ] Open an `EventSource` to a public SSE endpoint and log each incoming `data` field.
- [ ] Listen for named custom events using `addEventListener` on an `EventSource`.
- [ ] Implement a UI progress bar driven by SSE — update it as percentage values arrive from the server.
- [ ] Explain how SSE differs from WebSockets in terms of directionality, protocol, reconnection, and use cases.
- [ ] Describe the security concern of SSE endpoints that do not validate the `Origin` or `Authorization` headers before streaming data.

---

## 6. DOM Manipulation

### 6.1 `DOM API` — querySelector, Events, and Element Manipulation

- [ ] Use `document.querySelector` and `document.querySelectorAll` to select elements by tag, class, ID, attribute, and compound CSS selector.
- [ ] Create a new `<div>`, set its `textContent`, add a class, set a `data-*` attribute, and append it to an existing element — avoid using `innerHTML`.
- [ ] Use `addEventListener` to attach click, keydown, and input events; remove a listener with `removeEventListener` and explain why you need a named reference.
- [ ] Demonstrate event delegation: attach one listener to a `<ul>` to handle clicks on all current and future `<li>` children using `event.target`.
- [ ] Explain `event.stopPropagation()` vs `event.preventDefault()` with a concrete example of each.
- [ ] Build a live character-counter for a `<textarea>` using the `input` event that turns red when the limit is exceeded.
- [ ] Sanitize user input before inserting it into the DOM; demonstrate why assigning untrusted data to `innerHTML` creates an XSS vulnerability.
- [ ] Use `document.createDocumentFragment()` to batch-insert 1000 list items efficiently and compare DOM performance with inserting items one by one.

### 6.2 `DOMParser` — HTML/XML Parsing

- [ ] Parse an HTML string with `DOMParser` and extract all `<a>` href values without setting them on the live document.
- [ ] Parse an XML string with `DOMParser` and navigate its element tree using `getElementsByTagName` and `querySelector`.
- [ ] Explain why using `DOMParser` to process untrusted HTML is safer than assigning to `innerHTML` directly, and what risks still remain.
- [ ] Extract all form `<input>` names and values from a parsed HTML string — simulate reading a target page's form structure offline.

### 6.3 `MutationObserver` — Detecting DOM Changes

- [ ] Create a `MutationObserver` that logs when a child element is added to or removed from a container `<div>`.
- [ ] Observe attribute changes on a specific element and log the old value, new value, and attribute name on each change.
- [ ] Use `subtree: true` to monitor all descendants of a container and explain the performance implications of broad observation.
- [ ] Disconnect an observer at the right time and explain why failing to disconnect can cause memory leaks.
- [ ] Use `MutationObserver` in a security context: detect when a script or iframe is dynamically injected into the DOM and log a warning.

---

## 7. Parsing & Data Handling

### 7.1 `JSON` — Serialization and Parsing

- [ ] Parse a valid JSON string with `JSON.parse` and serialize an object with `JSON.stringify`; round-trip and verify the result is identical.
- [ ] Use the `replacer` parameter of `JSON.stringify` to omit sensitive fields (e.g., `password`, `token`) before logging.
- [ ] Use the `reviver` parameter of `JSON.parse` to automatically convert ISO date strings into `Date` objects.
- [ ] Handle `JSON.parse` errors with `try/catch`; write a safe parser wrapper that returns `null` and logs a warning on invalid input.
- [ ] Explain what happens to `undefined`, `Function`, `Symbol`, `NaN`, `Infinity`, and circular references during serialization; fix the circular reference case.

### 7.2 `URL` & `URLSearchParams` — URL Parsing and Manipulation

- [ ] Parse a full URL string with `new URL()` and read `protocol`, `hostname`, `port`, `pathname`, `search`, and `hash` individually.
- [ ] Use `URLSearchParams` to read, add, delete, and iterate over query parameters without manual string manipulation.
- [ ] Build an API URL safely by constructing a `URL` object from a base and appending user-supplied parameters via `URLSearchParams` — never concatenate strings.
- [ ] Detect open-redirect risk: validate that a redirect target URL has the same origin as the current application before redirecting.
- [ ] Parse the fragment (`#hash`) from a URL and explain how DOM XSS can originate from `location.hash` being passed to `innerHTML`.

### 7.3 `FormData` — Multipart Form Handling

- [ ] Build a `FormData` object manually, append text fields and a `Blob`, and send it with `fetch`.
- [ ] Explain why `Content-Type` must **not** be set manually when sending `FormData` with `fetch`.
- [ ] Describe a file-upload CSRF attack scenario and explain how `SameSite` cookies and CSRF tokens prevent it.

### 7.4 `TextEncoder` / `TextDecoder` — Encoding and Decoding

- [ ] Encode a UTF-8 string to a `Uint8Array` with `TextEncoder` and decode it back with `TextDecoder`; verify the round-trip is lossless.
- [ ] Encode a string containing non-ASCII characters (emoji, CJK) and log the raw byte values; explain multi-byte UTF-8 sequences.
- [ ] Use `TextEncoder` to convert a plaintext message before passing it to the Web Crypto API for hashing or encryption.

---

## 8. Security & Cryptography

### 8.1 `Web Crypto API` — Hashing, Encryption, and Key Generation

- [ ] Hash a string with `crypto.subtle.digest('SHA-256', ...)` using `TextEncoder`; log the result as a hex string.
- [ ] Generate a symmetric AES-GCM key with `crypto.subtle.generateKey`, encrypt a message, then decrypt it and verify the round-trip.
- [ ] Export and import a CryptoKey using `crypto.subtle.exportKey` and `importKey`; explain why keys must never be stored as plain strings.
- [ ] Generate an RSA-PSS key pair, sign a message with the private key, and verify the signature with the public key.
- [ ] Generate a random `Uint8Array` IV/nonce with `crypto.getRandomValues` and explain why it must never be reused with the same key.
- [ ] Derive a key from a password using `PBKDF2` via `crypto.subtle.deriveKey`; use a random salt and at least 100,000 iterations.
- [ ] Explain the difference between `crypto.subtle` (Web Crypto) and `Math.random()`; demonstrate why `Math.random()` must never be used for security purposes.

### 8.2 `jsonwebtoken` — JWT Creation and Verification

- [ ] Install `jsonwebtoken` and sign a payload with `jwt.sign` using a strong secret; log the resulting token structure (header, payload, signature).
- [ ] Verify a token with `jwt.verify`; handle `JsonWebTokenError` and `TokenExpiredError`.
- [ ] Decode a JWT without verification using `jwt.decode` and explain the security risk of trusting unverified JWT data.
- [ ] Demonstrate the `alg: none` attack: craft a token with no algorithm and explain why servers must explicitly disallow `none`.
- [ ] Sign a token using RSA (`RS256`) instead of HMAC (`HS256`); explain the key-confusion attack.
- [ ] Implement a token-refresh flow: issue an access token (15 min) and a refresh token (7 days); validate both correctly on the server side.
- [ ] Describe three JWT misuse patterns: storing JWTs in `localStorage`, including PII in the payload, and using a weak secret.

### 8.3 `crypto` (Node.js) — Server-Side Cryptography

- [ ] Use `crypto.createHash('sha256')` to hash a file's contents.
- [ ] Use `crypto.createHmac('sha256', secret)` to generate an HMAC; verify the HMAC on receipt to detect tampering.
- [ ] Use `crypto.randomBytes(32)` to generate a secure random token and compare it with `Math.random()`.
- [ ] Use `crypto.createCipheriv` and `crypto.createDecipheriv` with AES-256-GCM; include an IV and authentication tag.
- [ ] Use `crypto.timingSafeEqual` to compare two buffers and explain why a regular `===` comparison leaks timing information.
- [ ] Hash a password with `bcrypt` (or `argon2`) and verify it; explain why these are preferable to raw SHA-256 for password storage.

---

## 9. Automation & Testing

### 9.1 `Puppeteer` — Chrome Automation

- [ ] Launch a headless Chrome instance with Puppeteer, navigate to a URL, take a full-page screenshot, and save it to disk.
- [ ] Fill a login form using `page.type`, click the submit button with `page.click`, and wait for navigation to complete.
- [ ] Use `page.waitForSelector` instead of `page.waitForTimeout` and explain why fixed sleeps produce flaky automation scripts.
- [ ] Intercept network requests with `page.setRequestInterception(true)`; block image requests to speed up page loads and log all XHR requests.
- [ ] Evaluate JavaScript inside the browser context with `page.evaluate`; return a value from the browser to the Node.js context.
- [ ] Extract all links from a page using `page.evaluate` and resolve relative URLs; deduplicate the result set.
- [ ] Use Puppeteer to test for reflected XSS: inject a payload into a URL parameter and detect if `alert` fires using `page.on('dialog', ...)`.

### 9.2 `Playwright` — Cross-Browser Automation

- [ ] Install Playwright and run a basic test in Chromium, Firefox, and WebKit; log the browser name and page title for each.
- [ ] Use `page.locator()` and explain why locators are more resilient than CSS/XPath selectors.
- [ ] Use `expect(locator).toBeVisible()` and `expect(page).toHaveURL()` assertions instead of manual polling loops.
- [ ] Intercept network requests using `page.route` to mock an API response; verify that the UI renders the mocked data correctly.
- [ ] Run Playwright tests in parallel across multiple browsers using `--workers` and measure the total time saved.
- [ ] Detect open redirects with Playwright: navigate to a crafted URL and assert the final origin is the expected one.

### 9.3 `jsdom` — DOM Emulation for Testing

- [ ] Install `jsdom` and create a simulated document; append elements, query them, and read their `textContent` without a real browser.
- [ ] Use `jsdom` to test an XSS sanitization function: inject payloads and assert that no `<script>` tags survive.
- [ ] Explain the limitations of `jsdom` compared to a real browser and when Playwright is the better choice.
- [ ] Use `jsdom` with Jest's `testEnvironment: 'jsdom'` to run DOM tests inside a unit-test suite without spinning up a browser.

---

## 10. Security Context

### 10.1 Cross-Site Scripting (XSS)

- [ ] Explain the difference between Reflected XSS, Stored XSS, and DOM-based XSS; give one concrete attack payload for each.
- [ ] Demonstrate DOM XSS by writing code that passes `location.hash` directly to `innerHTML`; then fix it using `textContent` instead.
- [ ] Build a simple demo page that stores a comment in `localStorage` and renders it unsafely; exploit it, then add `DOMPurify` and re-test.
- [ ] Use a `Content-Security-Policy` header to block inline scripts; test that your XSS payload is blocked and log the CSP violation report.
- [ ] Write a CSP that allows only scripts from your own origin and `'nonce-{random}'`; explain why `'unsafe-inline'` defeats the policy.
- [ ] Identify a CSP bypass using `script-src 'unsafe-eval'`; explain what eval-based frameworks require and what the safer alternative is.

### 10.2 Prototype Pollution

- [ ] Explain what Prototype Pollution is; write a proof-of-concept that sets `Object.prototype.polluted = true` via a crafted object merge.
- [ ] Show how Prototype Pollution can lead to property injection in an application that trusts `obj.admin` for authorization.
- [ ] Demonstrate a safe deep-merge function that rejects keys like `__proto__`, `constructor`, and `prototype`.
- [ ] Use `Object.create(null)` to create a prototype-free dictionary and explain when this pattern is appropriate.
- [ ] Explain how Prototype Pollution can escalate to RCE in a Node.js application that uses a polluted property in a `child_process` call.

### 10.3 Client-Side Injection & Open Redirects

- [ ] Demonstrate HTML injection via `element.innerHTML = userInput`; exploit it, then patch it.
- [ ] Demonstrate JavaScript injection via `eval(userInput)` and `setTimeout(userInput, 0)`; explain why both sinks are dangerous.
- [ ] Detect an open redirect vulnerability: manipulate a `?redirect=` parameter to point to an external domain; then add an allowlist to fix it.

### 10.4 CSRF & CORS

- [ ] Explain Cross-Site Request Forgery: write a minimal HTML page that submits a form to a target origin when loaded.
- [ ] Explain how `SameSite=Strict`, `SameSite=Lax`, and CSRF tokens each prevent CSRF and the limitations of each approach.
- [ ] Describe a CORS misconfiguration: an API that reflects any `Origin` header in `Access-Control-Allow-Origin`; explain the data-theft attack.
- [ ] Explain the difference between simple and preflighted CORS requests and which one an attacker can send without a preflight.
- [ ] Write a minimal Node.js CORS middleware that only allows specific trusted origins from a whitelist.

### 10.5 `postMessage` Abuse

- [ ] Write a parent page that listens for `message` events without validating `event.origin`; demonstrate an exploit from a malicious iframe.
- [ ] Fix the `postMessage` listener by checking `event.origin` against an exact allowlist before processing any data.
- [ ] Explain `targetOrigin` in `window.postMessage(data, targetOrigin)` — show what happens when `'*'` is used and why it is dangerous.

### 10.6 Clickjacking

- [ ] Demonstrate clickjacking: embed a target page in a transparent iframe and overlay a fake button on top of a sensitive action.
- [ ] Explain how `X-Frame-Options: DENY` and `Content-Security-Policy: frame-ancestors 'none'` prevent clickjacking.

### 10.7 JWT Misuse

- [ ] Decode a JWT manually using `atob` on the Base64url-encoded header and payload.
- [ ] Demonstrate the `alg: none` attack by crafting a forged token and show how a vulnerable server would accept it.
- [ ] Explain three insecure JWT storage options (cookies without flags, `localStorage`, URL parameters) and rank them by risk.

### 10.8 Cookie & Storage Security

- [ ] Set a cookie with `HttpOnly`, `Secure`, `SameSite=Strict`, and a `__Host-` prefix; explain what each flag prevents.
- [ ] Demonstrate that XSS can exfiltrate all `localStorage` data: write a payload that collects all keys and sends them to an attacker endpoint.
- [ ] Explain why storing access tokens in `localStorage` is riskier than storing them in `HttpOnly` cookies.
- [ ] Explain how a persisted malicious Service Worker survives page reloads and can intercept credentials long after the initial XSS is fixed.

### 10.9 CSP Bypasses & WebSocket Security

- [ ] Demonstrate a JSONP-based CSP bypass where an allowlisted domain exposes a JSONP callback endpoint.
- [ ] Explain an AngularJS CSP bypass using allowlisted CDN domains.
- [ ] Explain why WebSocket connections are not protected by CORS; describe the `Origin` header check that servers must perform.
- [ ] Demonstrate a Cross-Site WebSocket Hijacking (CSWSH) attack scenario.

---

## 11. Automation Projects

### 11.1 Web Crawler

- [ ] Build a web crawler using Puppeteer or Playwright that starts from a seed URL, stays within the same origin, follows links up to depth 3, and deduplicates visited URLs.
- [ ] Respect `robots.txt` in your crawler: parse `Disallow` rules and skip restricted paths.
- [ ] Add rate limiting to your crawler: enforce a minimum delay between requests and honor `Retry-After` headers.
- [ ] Detect and report forms found on crawled pages: log the form action, method, and all input names.

### 11.2 API Fuzzer

- [ ] Build an API fuzzer that reads endpoints from a JSON config and sends boundary-value payloads (empty string, `null`, very long strings, special characters).
- [ ] Log interesting responses: status codes other than 200/400, unexpected `Content-Type`, or unusually large response bodies.
- [ ] Add a dry-run mode (`--dry-run`) that prints what the fuzzer would send without making real requests.

### 11.3 JavaScript Endpoint Extractor

- [ ] Build a tool that fetches a page, extracts all `<script src>` URLs, downloads each JS file, and searches for API endpoint patterns.
- [ ] Use a regex to find relative and absolute URL strings inside JavaScript source code.
- [ ] Extend the tool to search for `fetch(`, `axios.`, and `XMLHttpRequest` calls and extract their URL arguments.

### 11.4 Secret / API Key Finder

- [ ] Build a secret scanner that reads JavaScript files and searches for patterns matching API keys and tokens (AWS keys, GitHub tokens, Stripe keys).
- [ ] Implement at least 10 regex patterns for common secret formats; suppress known placeholder values.
- [ ] Report each finding with: file path, line number, matched pattern name, and a redacted version of the matched value.

### 11.5 JWT Analyzer

- [ ] Build a CLI tool that accepts a JWT string, decodes the header and payload without verification, and pretty-prints both.
- [ ] Check for common misconfigurations: `alg: none`, missing `exp` claim, and missing `iss` claim.
- [ ] Output a structured JSON report: algorithm, claims summary, expiry status, and a list of flagged issues.

### 11.6 CORS & CSP Testing Tools

- [ ] Build a CORS tester that sends requests to a target URL with crafted `Origin` headers and reports the `Access-Control-Allow-Origin` response.
- [ ] Build a CSP analyzer that fetches a URL, parses each directive, flags dangerous values, and rates the overall policy strength.
- [ ] Add a `--json` flag to both tools to export results for integration into a larger pentest report.

---

## 12. Modern Framework Awareness

- [ ] Explain how React renders components and where user-supplied data flows into the DOM; identify which React APIs are XSS-safe and which are dangerous (`dangerouslySetInnerHTML`).
- [ ] Explain Angular's template syntax and how its `[innerHTML]` binding and `DomSanitizer` service relate to XSS; describe Angular's security model.
- [ ] Explain Vue's `v-html` directive and why it bypasses Vue's automatic escaping; describe when it is safe to use.
- [ ] Describe how Svelte compiles templates and identify `{@html}` as the unsafe escape hatch.
- [ ] Explain how Next.js server-side rendering changes the attack surface compared to a pure client-side SPA.
- [ ] Describe the Electron security model: explain context isolation, `nodeIntegration`, `contextBridge`, and why disabling context isolation creates RCE from XSS.
- [ ] Research one real-world CVE for each of the above frameworks and summarize the root cause and patch in one paragraph each.

---

## 13. AI Web Applications

- [ ] Observe a ChatGPT or Claude conversation in browser DevTools: identify the API endpoint, request format (streaming or batch), and authentication header used.
- [ ] Explain how LLM streaming responses work using `EventSource` (SSE) or chunked `fetch`; write a minimal client that renders streamed tokens incrementally.
- [ ] Describe an indirect prompt injection attack: how an attacker-controlled document embedded in an LLM's context can alter the model's behaviour.
- [ ] Explain the client-side security considerations unique to AI platforms: streaming data handling, conversation history in memory, and token exfiltration via injected instructions.
- [ ] List three browser-observable artefacts (request headers, local storage keys, endpoint paths) that reveal which LLM provider a web application is using.

---

## 14. Code Quality

### 14.1 Debugging with Chrome DevTools

- [ ] Set a breakpoint in the Sources panel, step through code, and inspect variable values — avoid relying on `console.log` debugging.
- [ ] Use the Network panel to inspect a `fetch` request: examine request headers, response headers, response body, and timing.
- [ ] Use the Application panel to read cookies, `localStorage`, `sessionStorage`, and Service Worker registrations.
- [ ] Profile a slow function with the Performance panel; identify the most time-consuming call and describe a fix.

### 14.2 ESLint & Prettier

- [ ] Initialize ESLint in a project with `npx eslint --init`; fix all reported errors in a sample file and explain each rule triggered.
- [ ] Add the `eslint-plugin-security` plugin; run it on a project and explain each security warning it produces.
- [ ] Set up a pre-commit hook using `husky` and `lint-staged` that runs ESLint and Prettier before every commit.

### 14.3 Testing with Jest / Vitest

- [ ] Write a Jest unit test for a pure function; cover normal cases, boundary values, and error cases.
- [ ] Mock a `fetch` call in Jest using `jest.fn()` or `msw` (Mock Service Worker); test the function without hitting a real network.
- [ ] Measure test coverage with `jest --coverage`; identify uncovered branches and add tests to cover them.
- [ ] Write a parameterized test with `test.each` to cover multiple input/output pairs in a single test block.

### 14.4 Dependency Auditing & Git

- [ ] Run `npm audit` on a project with known vulnerabilities; interpret the output and apply fixes.
- [ ] Explain supply-chain risks: typosquatting, dependency confusion, and malicious maintainer takeover.
- [ ] Lock dependencies with `package-lock.json`; explain why committing the lockfile to version control is important.
- [ ] Write a `.gitignore` that excludes `node_modules/`, `.env`, `*.pem`, `*.key`, and build artefacts.
- [ ] Explain what happens when a secret is committed to Git history; describe the remediation steps using `git filter-repo`.

---

## 15. Mini Projects (Advanced)

- [ ] Build a browser-based JWT decoder: paste a token, parse and display header and payload, highlight expired or missing claims, and flag dangerous algorithms.
- [ ] Build a DOM XSS playground: create a page with several sink types (`innerHTML`, `document.write`, `eval`, `location.href`) and a controlled input field; demonstrate and explain each vulnerability.
- [ ] Build a WebSocket chat client and server: implement origin validation on the server, token-based authentication, and message schema validation; demonstrate the attack when each control is absent.
- [ ] Build a Puppeteer spider that crawls a lab web application, extracts all forms, submits XSS payloads to each, and detects if a dialog fires.
- [ ] Build a JavaScript secret scanner CLI: accept a directory path, recursively read all `.js` files, run regex patterns for common secrets, and output a JSON report.

---

## 16. Secure Coding Practices

- [ ] Never pass untrusted input to `innerHTML`, `document.write`, `eval`, `setTimeout` (string form), or `location.href` without validation and sanitization.
- [ ] Use `DOMPurify.sanitize(input)` before inserting user-controlled HTML; test that `<script>`, `onerror`, and `javascript:` payloads are stripped.
- [ ] Validate every value from `location.hash`, `location.search`, `document.referrer`, and `postMessage` events before use.
- [ ] Store secrets (API keys, tokens) in environment variables, never in client-side JavaScript or `localStorage`.
- [ ] Set `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`, `Strict-Transport-Security`, and `Referrer-Policy` headers on every response.
- [ ] Implement CSRF protection for state-changing requests using `SameSite=Strict` cookies and/or synchronizer tokens.
- [ ] Use `crypto.randomUUID()` or `crypto.randomBytes` for token generation — never `Math.random()`.
- [ ] Add a responsible-use statement, target scope, and authorization requirements to every automation tool's README.
- [ ] Review one project against an OWASP ASVS checklist before release.

---

## 17. Learning Outcome

- [ ] Confidently inspect any modern JavaScript web application using browser DevTools: read network traffic, cookies, storage, and source maps.
- [ ] Identify and demonstrate at least five client-side vulnerabilities (XSS, CSRF, open redirect, postMessage abuse, Prototype Pollution) on a lab application you own.
- [ ] Build, document, and responsibly release one automation tool (crawler, fuzzer, or secret scanner) with proper scope controls, rate limiting, and a dry-run mode.
- [ ] Analyse a real-world SPA (React, Angular, or Vue) and describe its authentication flow, API communication pattern, and at least two testable attack surfaces.
- [ ] Explain to a non-technical stakeholder why client-side security matters even when the server is hardened.
- [ ] Produce a sample pentest report section covering JavaScript-specific findings using only synthetic or authorized data.
- [ ] Obtain explicit written authorization before testing any application, API, or infrastructure you do not personally own.
