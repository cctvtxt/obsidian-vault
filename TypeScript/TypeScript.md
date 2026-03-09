#Hard

- не должно ыть эни
- можно тип remote DTO
- удалять оверхед типизацию где типы не важны
- убрать избыточные прверк и запросы к бд

```js
tsc types.ts node types
```

All TypeScript code is converted into its JavaScript equivalent for the purpose of execution.

Any valid .js file can be renamed to .ts and compiled with other TypeScript files.

TypeScript doesn’t need a dedicated VM or a specific runtime environment to execute.

TypeScript embraces features like generics and type annotations that aren’t a part of the EcmaScript6 specification.

**_Components:_**

**Language** − It comprises of the syntax, keywords, and type annotations.

The TypeScript **Compiler** − The TypeScript compiler (tsc) converts the instructions written in TypeScript to its JavaScript equivalent.

The TypeScript **Language Service** − The "Language Service" exposes an additional layer around the core compiler pipeline that are editor-like applications.

- Decorators

The idea is this: we wrap a function (object, class) with another function, which, having done its job, will call the wrapped function with the same parameters as without the wrapper. Thus, neither the function nor the code that calls it will feel the difference.

**_What does it give us?_**

**Firstly**, we can wrap a function already wrapped in another wrapper, do it as many times as we like, and nothing will break (in theory).

**Secondly**, we can flexibly add such wrappers even at runtime, and the code will work the same way (if the wrappers are done correctly). This idea is implemented in many cool things and has many names: High Order Component in React, middleware in node, decorators in Java, in Type Script.

**In general**, this is not difficult to do in JS. For example, we want to make a spy function that records how many times and with what arguments any other function has been called. We create a decorator function that takes a function to watch as an argument. Neither the calling code nor the wrapped function will feel the trick and will work as it should.

**_But there is one "but"_**

**Each decorator must be called with an argument to the wrapped function**. And if there are many of them, a chain of nested calls is obtained. This is annoying to read and write. Therefore, various ways have been devised to simplify this syntax.

For example, you can make a combinator function, to which you pass an array of decorators as the first argument, and as the second a function that you need to wrap with all of them.

And the combinator already walks through the array inside and wraps everything in turn. Type Script went further and found a **special syntax**: when the compiler encounters `@` then it looks for a function with the name specified after the `@`, wraps in it what is on the next line: passes the appropriate arguments to it and ensures that the wrapped function is called with all call arguments after processing the wrapper, provided that undefined was returned from it.
