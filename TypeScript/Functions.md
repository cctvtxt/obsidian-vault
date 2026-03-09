[[TypeScript]]

```typescript
const arrayOfNumbers: Array<number> = [1, 1, 2, 3, 5]
const arrayOfStrings: Array<string> = [`1`, `1`, `2`]
function reverse<T>(array: T[]): T[] {
  return array.reverse()
}

const promise = new Promise<string>((resolve) => {
  setTimeout(() => resolve(`RESOLVED`), 2000)
})
promise.then((data) => {
  console.log(data)
})

function mergeObjects<T extends object, R extends object>(a: T, b: R) {
  return Object.assign({}, a, b)
}
const merged = mergeObjects({ name: "USERNAME" }, { age: 55 })

interface ILength {
  length: number
}
function withCount<T extends ILength>(value: T): { value: T; count: string } {
  return {
    value,
    count: `LENGTH: ${value.length}`,
  }
}
console.log(withCount(`ARGUMENT`))
console.log(withCount([`AR`, `GU`, `MENTS`]))

function getObjectValue<T extends object, R extends keyof T>(obj: T, key: R) {
  return obj[key]
}
const person = {
  name: `USERNAME`,
  age: 55,
  job: `NODEJS`,
}
console.log(getObjectValue(person, `name`))
console.log(getObjectValue(person, `age`))
console.log(getObjectValue(person, `job`))

class Collection<T extends number | string | boolean> {
  constructor(private _items: T[] = []) {}

  add(item: T) {
    this._items.push(item)
  }
  remove(item: T) {
    this._items = this._items.filter((i) => i !== item)
  }
  get items(): T[] {
    return this._items
  }
}
const strings = new Collection<string>([`AR`, `GU`, `MENTS`])
strings.add("!")
strings.remove("GU")
const numbers = new Collection<number>([1, 2, 3])
numbers.add(4)
numbers.remove(1)

// Utility Types: Partial, Required, Readonly, Record, Pick, Omit, Exclude, Extract, NonNullable etc

// Partial<Type> - optional behavior for types

interface Motorcycle {
  model: string
  year: number
}
function createCycle(model: string, year: number): Motorcycle {
  const cicle: Partial<Motorcycle> = {}
  if (model.length > 3) {
    cicle.model = model
  }
  if (year > 2000) {
    cicle.year = year
  }
  return cicle as Motorcycle
}

// Readonly<Type> - disable mutability of arrays and objects

const cars: Readonly<Array<string>> = [`Ford`, `Audi`]
```