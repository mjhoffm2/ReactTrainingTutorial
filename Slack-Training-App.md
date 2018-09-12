

Slack clone
- Entities
    - users
    - channels
    - messages
 - Requirements
    - add users
    - soft-delete users
    - create channels
    - soft-delete channels
    - join channels
    - leave channels
    - send messages
    - hard-delete messages



Each item in this list would make for a fairly substantial blog post.
Most can be split into multiple smaller posts if necessary.

1. Achieving 'Hello World'
     - Setting up Node, Webpack, TypeScript, React
     - Establish a bare-minimum manual build process
     - Get to the point where we have some kind of page that prints 'hello world' to the dom via react
 2. Basic Application Skeleton
     - Add a couple of trivial react components such as 'ParentComponent', 'ChildComponent', etc
     - Bare minimum Redux integration (Probably includes React-Redux)
         - trivial store+reducer (ex: Counter)
         - maybe one or two basic actions (if any)
     - Bare minimum React-Router integration
         - likely just two routes
     - React-Bootstrap integration, single component
     - Basic TypeScript introduction (only what is needed to understand examples)
 3. Improve development experience
     - Improve webpack/typescript configuration
     - Automate build process
     - Hot module reloading
     - Environment differentiation
 4. (C# Only) Code generation
     - Basic/Intermediate intro to TypeScript
     - TypeWriter
     - Generating models
     - Generating API methods
     - things exist for Java such as https://github.com/vojtechhabarta/typescript-generator but I do not have experience with them
 5. Fleshing out the application
     - Application structure
     - Redux/React-Redux integration and usage
     - React-Router integration and usage
     - React-Bootstrap usage
     - Some non-trivial react components
 6. TypeScript advanced features and concepts
     - Useful ES6+ Syntax and features
     - Creating and manipulating interfaces and types
     - Typescript Generics
     - Traps to avoid (ex: Readonly<>)
 7. React advanced features and concepts
     - Correct state management
     - Lifecycle methods and how they should be used
     - Redux vs Props vs Context
     - Performance
 8. Example of complete application
     - Would still be pretty basic but would have some non-trivial features
     - Source code
     - Explanation/documentation