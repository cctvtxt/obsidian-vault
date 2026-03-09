[[TypeScript]]

```typescript
// Decorating in programming is simply wrapping one piece of code with another, thereby decorating it. A decorator (also known as a decorator function) can additionally refer to the design pattern that wraps a function with another function to extend its functionality.

// Advanced example

interface ComponentDecorator {
  selector: string
  template: string
}

function Comp(config: ComponentDecorator) {
  return function <T extends { new (...args: any[]): object }>(Constructor: T) {
    return class extends Constructor {
      constructor(...args: any[]) {
        super(...args)

        const el = document.querySelector(config.selector)!
        el.innerHTML = config.template
      }
    }
  }
}

function Bind(_: any, __: any, descriptor: PropertyDescriptor) {
  const original = descriptor.value
  return {
    configurable: true,
    numerable: false,
    get() {
      return original.bind(this)
    },
  }
}

@Comp({
  selector: `#card`,
  template: `
      <div class="card">
        <div class="card-content">
          <span class="card-title">Card Comp</span>
        </div>
      </div>
    `,
})
class CardComponent {
  constructor(public name: string) {}
  @Bind
  logName(): void {
    console.log(`Component: ${this.name}`)
  }
}

// 1 type - CLASS DECORATOR : if class decorator returns a value, then it will replace the class declaration according to the given constructor

const logClass = (constructor: Function) => {
  console.log(constructor)
}

@logClass
class Class_1 {
  constructor(public name: string, public age: number) {}
}

// 2 type - PROPERTY DECORATOR : can return either null or a property descriptor; target - the prototype of the class to which the dectorator is applied

const logProperty = (target: object, propertyKey: string | symbol) => {
  console.log(propertyKey)
}

class Class_2 {
  @logProperty
  secret: number

  constructor(public name: string, public age: number, secret: number) {
    this.secret = secret
  }
}

// 3 type - METHOD DECORATOR : can return either null or a property descriptor

const logMethod = (
  target: object,
  propertyKey: string | symbol,
  descriptor: PropertyDescriptor
) => {
  console.log(propertyKey)
}

class Class_3 {
  constructor(public name: string, public age: number) {}

  @logMethod
  public getPass(): string {
    return `${this.name},${this.age}`
  }
}

// 4 type - ACCESSOR DECORATOR : get/set

const logSet = (
  target: object,
  propertyKey: string | symbol,
  descriptor: PropertyDescriptor
) => {
  console.log(propertyKey)
}

class Class_4 {
  constructor(public name: string, public age: number) {}

  @logSet
  set myAge(age: number) {
    this.age = age
  }
}

// Factory Decorator example

function factory(value: any) {
  return function (target: any) {
    console.log(target)
  }
}

// Applying Factory Decorator

const enumerable = (value: boolean) => {
  return (
    target: any,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor
  ) => {
    descriptor.enumerable = value
  }
}

class Class_5 {
  constructor(public name: string, public age: number) {}

  @enumerable(false)
  public getPass(): string {
    return `${this.name},${this.age}`
  }
}

// Decorator composition syntax

// @f @g x

// @f
// @g
// x

const first = () => {
  console.log("first() completing")
  return (
    target: any,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor
  ) => {
    console.log("frirst() called")
  }
}

const second = () => {
  console.log("second() completing")
  return (
    target: any,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor
  ) => {
    console.log("second() called")
  }
}

class Class_6 {
  constructor(public name: string, public age: number) {}

  @first()
  @second()
  public getPass(): string {
    return `${this.name},${this.age}`
  }
}

// Call results :

// first() completing   // Factory 1
// second() completing  // Factory 2
// second() called      // Factory 2
// first() called       // Factory 1

// expressions for decorators are called and evaluated from top to bottom; results call decorators from bottom to top
```