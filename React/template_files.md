**README.md.** This is a markdown file that contains a description of your application. For example, Github will use this file to generate an html summary that you can see at the bottom of projects.

**package.json.** This file contains metadata relevant to the project. For example, it contains the name, version and description of our app. It also contains the dependencies list with external libraries that our app depends on.

**@types/*** packages contain type definitions for libraries originally written in JavaScript. 

TypeScript can parse this code, but it has no way of knowing what type the data attribute is restricted to. So for TypeScript, the data attribute will implicitly have type any. This type matches with absolutely anything, which defeats the purpose of type-checking.

If we know that the function is meant to be more specific, for instance, it only accepts the values of type string, we can create a *.d.ts file and describe it there manually.

This *.d.ts file name should match the module name we provide types for. For example, if this saveData function comes from the save-data module - we will create a save-data.d.ts file. We'll need to put this file where the TypeScript compiler will see it, usually in its src folder.

It is a convention that all the types-packages are published under the @types namespace. Those packages are provided by the [DefinitelyTyped](http://www.definitelytyped.org/) repository.

**yarn.lock.** This file is generated when you install the dependencies by running yarn in your project root. The file contains resolved dependencies versions along with their sub-dependencies. It is needed for consistent installations on different machines. If you use npm to manage dependencies, you will have a package-lock.json instead.

**tsconfig.json.** This contains the TypeScript configuration. We don't need to edit this file because the default settings work fine for us.

**.gitignore.** This file contains the list of files and folders that shouldn't end up in your git repository.


**PUBLIC FOLDER**
The **public folder** contains the static files for our app. They are not included in the compilation process and remain untouched during the build.

**index.html.** This file contains a special <div id="root"> that will be a mounting point for our React application.

**manifest.json.** This provides application metadata for Progressive Web Apps. For example, the file allows installation of your application on a mobile phone's home screen, similar to native apps. It contains the app name, icons, theme colors, and other data needed to make your app installable.

**favicon.ico, logo192.png, logo512.png.** These are icons for your application. There is favicon.ico, a small icon that is shown on browser tabs. Also, there are two bigger icons: logo192.png and logo512.png. They are referenced in manifest.json and will be used on mobile devices if your app will be added to the home screen.

**robots.txt.** This tells crawlers what resources they shouldn’t access. By default it allows everything.

**SRC FOLDER**

Take a look at the src folder. Files in this folder are processed by webpack and will be added to your app's bundle.

This folder contains a bunch of files with .tsx extension: index.tsx, App.tsx, App.test.tsx. It means that those files contain JSX code.

In a JavaScript React application, we could use either .jsx or .js extensions for such files. It would make no difference.

With TypeScript, you should use .tsx extensions on files that have JSX code, and .ts on files that don't.

This is important because otherwise there can be a syntactic clash. Both TypeScript and JSX use angle brackets, but for different purposes.

**index.tsx**

The most important file in the /src folder is index.tsx. It is an entry point for our application. It means that webpack will start to build our application from this file, and then will recursively include other files referenced by import statements.

**App.tsx**

Let's open src/App.tsx. If you use modern create-react-app, this file won't be very different to the regular JavaScript version.

**react-app-env.d.ts**
Another file with an interesting extension is react-app-env.d.ts. Let's take a look.

Files with *.d.ts extensions contain TypeScript types definitions. Usually, these are needed for libraries that were originally written in JavaScript.

TypeScript doesn't even see the static resources files. It is only interested in files with .tsx, .ts, and d.ts extensions. With some tweaking, it will also see .js and .jsx files.

