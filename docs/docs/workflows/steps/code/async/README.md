# Running asynchronous code

If you're not familiar with asynchronous programming concepts like [callback functions](https://developer.mozilla.org/en-US/docs/Glossary/Callback_function) or [Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises), see [this overview](https://eloquentjavascript.net/11_async.html).

[[toc]]

## The problem

**Any asynchronous code within a Node.js code step must complete before the next step runs**. This ensures future steps have access to its data. If Pipedream detects that code is still running by the time the step completes, you'll see the following warning below the code step:

> **This step was still trying to run code when the step ended. Make sure you await all Promises, or wrap callback functions in Promises.**

As the warning notes, this often arises from one of two issues:

- You forgot to `await` a Promise. [All Promises must be awaited](#await-all-promises) so they run synchronously.
- You tried to run a callback function. Since callback functions run asynchronously, they typically will not finish before the step ends. [You can wrap your function in a Promise](#wrap-callback-functions-in-a-promise) to run it synchronously.

## Solutions

### `await` all Promises

Most Node.js packages that run async code return Promises as ther result of method calls. For example, [`axios`](https://docs.pipedream.com/workflows/steps/code/nodejs/http-requests/#basic-axios-usage-notes) is an HTTP client. If you make an HTTP request like this in a Pipedream code step:

```javascript
const resp = axios({
  method: "GET",
  url: `https://swapi.co/api/films/`,
});
```

It won't send the HTTP request, since **`axios` returns a Promise**. Instead, add an `await` in front of the call to `axios`:

```javascript
const resp = await axios({
  method: "GET",
  url: `https://swapi.co/api/films/`,
});
```

In short, always do this:

```javascript
const res = await runAsyncCode();
```

instead of this:

```javascript
// This code may not finish by the time the workflow finishes
runAsyncCode();
```

### Wrap callback functions in a Promise

Before support for Promises was widespread, [callback functions](https://developer.mozilla.org/en-US/docs/Glossary/Callback_function) were a popular way to run some code asynchronously, after some operation was completed. For example, [PDFKit](https://pdfkit.org/) lets you pass a callback function that runs when certain events fire, like when the PDF has been finalized:

```javascript
const PDFDocument = require("pdfkit");
const fs = require("fs");

let doc = new PDFDocument({ size: "A4", margin: 50 });
this.fileName = `tmp/test.pdf`;
let file = fs.createWriteStream(this.fileName);
doc.pipe(file);
doc.text("Hello world!");

// Finalize PDF file
doc.end();
file.on("finish", () => {
  console.log(fs.statSync(this.fileName));
});
```

This is the callback function:

```javascript
() => {
  console.log(fs.statSync(this.fileName));
};
```

and **it will not run**. By running a callback function in this way, we're saying that we want the function to be run asynchronously. But on Pipedream, this code must be run _synchronously_ so it finishes by the time the step ends.

**You can wrap this callback function in a Promise to run it synchronously**. Instead of running

```javascript
file.on("finish", () => {
  console.log(fs.statSync(this.fileName));
});
```

run:

```javascript
await new Promise((resolve, reject) => {
  file.on("finish", () => {
    const stats = fs.statSync(this.filePath);
    // When you're done, call resolve() to finish
    resolve();
  });
});
```

### Other solutions

Many functions that accept callbacks can also be "[promisified](https://javascript.info/promisify)": converted into a function that returns a Promise, and that you can `await`.

If a specific library doesn't support Promises, you can often find an equivalent library that does support Promises. For example, many older HTTP clients like `request` didn't support Promises natively, but the community [published packages that wrapped it with a Promise-based interface](https://www.npmjs.com/package/request#promises--asyncawait) (note: `request` has been deprecated, this is just an example).