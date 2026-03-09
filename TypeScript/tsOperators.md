[[TypeScript]]

```typescript
interface Person {
  name: string
  age: number
}

type PersonKeys = keyof Person

let KEY: PersonKeys = `name`
KEY = `age`

// KEY = `job` - invalid input

type User = {
  _id: number
  name: string
  email: string
  createdAt: Date
}

// Exclude Pick

type UserKeysNoMeta1 = Exclude<keyof User, "_id" | "createdAt">
// type : name | email
type UserKeysNoMeta2 = Pick<User, "name" | "email">
// type : name | email

let u1: UserKeysNoMeta1 = "name"
```