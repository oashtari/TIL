Types and interfaces are mostly interchangeable.

By default all the fields you define on your types are required. It means that if the field will be missing you will get a type error. To make the field optional you can add a question mark before the colon:

code block with indentation or 4 spaces:

    type ExampleProps = {
    someField?: string
    }

code block with backticks:

```
type ExampleProps = {
someField?: string
}
```


The ampersand combines the two types into one. In TypeScript this is called a type intersection.

When I need to know what the name is of some element type, I usually check the [@types/react/global.d.ts](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/react/global.d.ts) file. It contains type definitions for types that have to be exposed globally (not in React namespace).