[[JavaScript]]

- prototype and class inheritance

const Box box = new Box(50);
const Object cube = new box;
// const Object cube = new box({delete: 'size', add: zSize});
cube.zSize = 50;

box.instance // Box, Figure
box.prototype // null

cube.instance // Object
cube.prototype // box

// null is a instance of the Null class