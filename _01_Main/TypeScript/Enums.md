[[TypeScript]]


```typescript
// Simple example of enum type
enum Directions {
  Up,
  Down,
  Left,
  Right,
}

Directions.Up // 0
Directions.Down // 1
Directions.Left // 2
Directions.Right // 3

Directions[0] // "Up"
Directions[1] // "Down"
Directions[2] // "Left"
Directions[3] // "Right"

// Custom index for enum elements

enum Dirs {
  Up = 2,
  Down = 4,
  Left = 6,
  Right = 8,
}

// Custom name for keys

enum links {
  youtube = "https://youtube.com/",
  vk = "https://vk.com/",
  facebook = "https://facebook.com/",
}

// Using
links.vk // "https://vk.com/"
links.youtube // "https://youtube.com/"

enum Membership {
  Simple,
  Standard,
  Premium,
  PrmiumPlus,
}

const membership = Membership.Premium
const membershipReverse = Membership[2]
console.log(membership) // 2
console.log(membershipReverse) // Premium

enum SocialMedia {
  vk = `VK`,
  facebook = `FACEBOOK`,
  instagram = `INSTAGRAM `,
}

const media = SocialMedia.facebook
console.log(media) // FACEBOOK
```