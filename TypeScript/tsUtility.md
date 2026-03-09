[[TypeScript]]

```typescript
// 16 or more

// Readonly<T>

interface Util {
  name: string
}

const nonmut_obj: Readonly<Util> = {
  name: `NON-MUTABUL`,
}

// Required<T>

interface Props {
  a?: string
  b?: string
}

const filled_obj: Required<Props> = {
  a: `required val 1`,
  b: `required val 2`,
}

// Record<K, T>
interface PageInfo {
  title: string
}

type Page = "home" | "about" | "contact"

const assimilate_obj: Record<Page, PageInfo> = {
  about: { title: "about" },
  contact: { title: "contact" },
  home: { title: "home" },
}

// Pick<T,K>

interface Todo {
  title: string
  description: string
  completed: boolean
}

type TodoPrev_1 = Pick<Todo, "title" | "completed">

const definite_obj: TodoPrev_1 = {
  title: "Clean room",
  completed: false,
}

// Omit<T,K>

type TodoPrev_2 = Omit<Todo, "description">

const reduced_obj: TodoPrev_2 = {
  title: "Clean room",
  completed: false,
}

// Exclude<T, U>

type T_10 = Exclude<"a" | "b" | "c", "a">
type T_11 = Exclude<"a" | "b" | "c", "a" | "b">
type T_12 = Exclude<string | number | (() => void), Function>

// Extract<T, U>

type T_20 = Extract<"a" | "b" | "c", "a" | "f">
type T_21 = Extract<string | number | (() => void), Function>

// NonNullable<T> : undefined will beexcluded from type

type T_30 = NonNullable<string | number | undefined>
type T_31 = NonNullable<string[] | null | undefined>

// ReturnType<T>

declare function f(): { a: number; b: string }

type T_40 = ReturnType<() => string> // string
type T_41 = ReturnType<(s: string) => void> // void
type T_42 = ReturnType<<T>() => T> // {}
type T_43 = ReturnType<<T extends X, X extends number[]>() => T> // number[]
type T_44 = ReturnType<typeof f> // { a: number, b: string }
type T_45 = ReturnType<any> // any
type T_46 = ReturnType<never> // any
// type T_47 = ReturnType<string>;    // Error
// type T_48 = ReturnType<Function>;  // Error

// InstanceType<T>

class C {
  x = 0
  y = 0
}

type T_50 = InstanceType<typeof C> // C
type T_51 = InstanceType<any> // any
type T_52 = InstanceType<never> // any
// type T_53 = InstanceType<string>;       // Error
// type T_54 = InstanceType<Function>;     // Error
```