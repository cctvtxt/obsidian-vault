[[TypeScript]]

```typescript
const message: string = `tsc types.ts node types`
const string: string = "Hello"
const isLoading: boolean = false
const int: number = 32
const float: number = 32
const num: number = 3e10

const numberArray_1: number[] = [1, 1, 2, 3, 4, 8]
const words: string[] = [`ts`, `js`]

// Generic
const numberArray_2: Array<number> = numberArray_1

// Tuple
const conact: [string, number] = [`Guy`, 89083105732]

// Any
let variable: any = 5
variable = []
variable = {}

// Functions
function sayMyName(name: string): void {
  console.log(name)
}

// Never
function throwError(message: string): never {
  throw new Error(message)
}

function infinite(): never {
  while (true) {}
}

// Type
type Login = string
const login: Login = `admin`

type ID = string | number
const id_1: ID = 1234
const id_2: ID = `1234`

type Some = string | null | undefined
```