Modules are the basic building blocks for constructing Node.js applications, A Node.js module encapsulates functions, hiding details inside a well-protected container, and exposing an explicitly declared API. Every source file used in an application on the Node.js platform is a module.

**Examining the traditional Node.js module format**

```javascript
const fs = require('fs'); 
```
The require function is given a module identifier, and it searches for the module named by that identifier. If found, it loads the module definition into the Node.js runtime and making its functions available. In this case, the fs object contains the code (and data) exported by the fs module. The fs module is part of the Node.js core and provides filesystem functions. By declaring fs as const, we have a little bit of assurance against making coding mistakes. We could mistakenly assign a value to fs, and then the program would fail, but as a const we know the reference to the fs module will not be changed \
What does it mean to say the fs object contains the code exported by the fs module? In a CommonJS module, there is an object, module, provided by Node.js, with which the module's author describes the *module*. Within this object is a field, *module.exports*, containing the functions and data exported by the module. The return value of the require function is the object \
Anything added to the module.exports object is available to other pieces of code, and everything else is hidden. As a convenience, the module.exports object is also available as exports
Because exports is an alias of module.exports, the following two lines of code are equivalent:
``` javascript
exports.funcName = function(arg, arg1) { ... };
module.exports.funcName = function(arg, arg2) { .. }; 
```
Do not ever do anything like the following:
``` javascript
exports = function(arg, arg1) { ... };
```
Any assignment to exports will break the alias, and it will no longer be equivalent to module.exports
If your intent is to assign a single object or function to be returned by require, do this instead
``` javascript
module.exports = function(arg, arg1) { ... };
```
To give us a brief example, let's create a simple module, named simple.js:
``` javascript
var count = 0;
exports.next = function() { return ++count; };
exports.hello = function() {
  return "Hello, world!";
};
```
We have one variable, count, which is not attached to the exports object, and a function, next, which is attached. Because count is not attached to exports, it is private to the module. 
Any module can have private implementation details that are not exported and are therefore not available to any other code.

This is how Node.js solves the global object problem of browser-based JavaScript. The variables that look like they are global variables are only global to the module containing the variable. These variables are not visible to any other code.

*Note: The Node.js package format is derived from the CommonJS module system (http://commonjs.org)* \
The module object is a global-to-the-module object injected by Node.js. It also injects two other variables: __dirname and __filename. These are useful for helping code in a module know where it is located in the filesystem. Primarily, this is used for loading other files using a path relative to the module's location. 

For example, one can store assets like CSS or image files in a directory relative to the module. An app framework can then make the files available via an HTTP server. In Express, we do so with this code snippet:
``` javascript
app.use('/assets/vendor/jquery', express.static( 
 path.join(__dirname, 'node_modules', 'jquery'))); 
```

**ES6/ES2015 module format**

ES6 modules are a new module format designed for all JavaScript environments

The ES6 and CommonJS modules are conceptually similar. Both support exporting data and functions from a module, and both support hiding implementation inside a module. But they are very different in many practical ways.

An issue we have to deal with is the file extension to use for ES6 modules. Node.js needs to know whether to parse using the CommonJS or ES6 module syntax. To distinguish between them, Node.js uses the file extension .mjs to denote ES6 modules, and .js to denote CommonJS modules.  However, that's not the entire story since Node.js can be configured to recognize the .js files as ES6 modules.

Let's start with defining an ES6 module
``` javascript
let count = 0;
export function next() { return ++count; }
function squared() { return Math.pow(count, 2); }
export function hello() {
    return "Hello, world!";
}
export default function() { return count; }
export const meaning = 42;
export let nocount = -1;
export { squared };
```
The export keyword declares what is being exported from an ES6 module. In this case, we have several exported functions and two exported variables. The export keyword can be put in front of any top-level declaration, such as variable, function, or class declarations:
``` javascript
export function next() { .. }
```
The effect of this is similar to the following:
``` javascript
module.exports.next = function() { .. }
```
A statement such as export function next() is a named export, meaning the exported function (as here) or object has a name, and that code outside the module uses that name to access the object. As we see here, named exports can be functions or objects, and they may also be class definitions.

The default export from a module, defined with export default, can be done once per module. The default export is what code outside the module accesses when using the module object itself, rather than when using one of the exports from the module.

Now let's see how to use the ES2015 module
``` javascript
import * as simple2 from './simple2.mjs';

console.log(simple2.hello());
console.log(`${simple2.next()} ${simple2.squared()}`);
console.log(`${simple2.next()} ${simple2.squared()}`);
console.log(`${simple2.default()} ${simple2.squared()}`);
console.log(`${simple2.next()} ${simple2.squared()}`);
console.log(`${simple2.next()} ${simple2.squared()}`);
console.log(`${simple2.next()} ${simple2.squared()}`);
console.log(simple2.meaning);
```