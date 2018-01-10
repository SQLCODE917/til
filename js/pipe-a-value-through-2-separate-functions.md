# Composition - pipe a value through 2 separate functions

Use a higher order function to compose an unlimited number of functions, and pipe in a value:
The value becomes the initial value for the `reduce` accumulator.
Then, the return value of the function replaces it.
Don't forget to make these function immutable!

```
const compose = (...fns) =>
  (argument) =>
    fns.reduce(
      (arg, fn) => fn(arg),
      argument
    )
```

In action:

```
const addOne = number => number+1;
const addTwo = number => number+2;
const startNumber = 0;

compose(addOne, addTwo)(startNumber)
```

Outputs `3`.
`startNumber` retains it's value.
If there was an attempt to mutate it, there would have been errors
