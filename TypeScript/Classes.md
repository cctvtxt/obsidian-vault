[[TypeScript]]

```typescript

class TypeScrypt {
  version: string

  constructor(version: string) {
    this.version = version
  }
  info(name: string) {
    return `[${name}]: TS version is ${this.version}`
  }
}

class Car {
  readonly model: string
  constructor(model: string) {
    this.model = model
  }
}

// ! Modificators :
// Protected, Public (default), Private

class Animal {
  protected voice: string = ``
  public color: string = `black`
  private go() {
    console.log(`go`)
  }
}

class Cat extends Animal {
  public setVoice(voice: string): void {
    this.voice = voice
  }
}

const cat = new Cat()
cat.setVoice(`meow`)
cat.color

// we cant use cat.voice - method protected
// and cat.go - private
// only cat.color - public

// ! Abstract classes and methods

// ? they don't compile at all
// ? we can extends from it

abstract class Component {
  abstract render(): void
  abstract info(): string
}

class AppComponent extends Component {
  render(): void {
    console.log(`rendering`)
  }
  info(): string {
    return `info`
  }
}
```