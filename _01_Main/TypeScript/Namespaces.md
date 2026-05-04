[[TypeScript]]

```typescript
// ! Basically, this is necessary to combine certain elements into groups - to distinguish between types and interfaces in large projects.

// That is, namespaces contain information about types and interfaces.

// This information is not compiled into anything.

// import syntax :

// ///<reference path="path to namespace file" />

// * File "Utils.ts"

namespace Validation {
  export interface StringValidator {
    isAcceptable(s: string): boolean
  }
}

// * File "Customers.ts"

/// <reference path="Validation.ts" />

namespace Validation {
  const lettersRegexp = /^[A-Za-z]+$/
  export class LettersOnlyValidator implements StringValidator {
    isAcceptable(s: string) {
      return lettersRegexp.test(s)
    }
  }
}

// ! Data, methods placed in namespace will be compiled in IIFE :

namespace NData {
  export const SECRET: string = `js better than python`
  export const PI: number = 3.14
}

console.log(NData.SECRET)

// In this case, namespaces solves the problem with global variables (similar to IIFE or data encapsulation, as in classes with static properties).

// However, exporting data via namespaces is considered not the best solution.
// It is customary to export data using modules. They create their own namespaces.
```