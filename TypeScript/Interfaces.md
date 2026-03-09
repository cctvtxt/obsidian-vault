[[TypeScript]]

```typescript
interface Rect {
  readonly id: string
  color?: string
  size: {
    width: number
    height: number
  }
}

const rect: Rect = {
  id: `1234`,
  size: {
    width: 20,
    height: 30,
  },
}

const newRect2 = <Rect>{}
const newRect3 = {} as Rect

interface RectWithArea extends Rect {
  getArea: () => number
}

const newRect4: RectWithArea = {
  id: `1234`,
  size: {
    width: 10,
    height: 20,
  },
  getArea(): number {
    return this.size.width * this.size.height
  },
}

interface IClock {
  time: Date
  setTime(date: Date): void
}

class Clock implements IClock {
  time: Date = new Date()
  setTime(date: Date): void {
    this.time = date
  }
}
interface Styles {
  [key: string]: string
}

const css: Styles = {
  border: `1px solid black`,
  marginTop: `2px`,
  borderRadius: `5px`,
}
```