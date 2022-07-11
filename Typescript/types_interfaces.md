Types and interfaces are mostly interchangeable.

By default all the fields you define on your types are required. It means that if the field will be missing you will get a type error. To make the field optional you can add a question mark before the colon:

    type ExampleProps = {
    someField?: string
    }

```
type ExampleProps = {
someField?: string
}
```


The ampersand combines the two types into one. In TypeScript this is called a type intersection.