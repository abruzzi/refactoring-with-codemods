# Refactoring with Codemods to Automate API Changes

*Codemods are powerful tools that help developers automate code refactoring tasks, especially when introducing breaking changes to APIs. They work by parsing code into an Abstract Syntax Tree (AST) and then applying transformations to make precise, large-scale changes with minimal manual effort. This article explores how codemods can help library developers manage breaking API changes, improve code hygiene, and reduce technical debt in large or distributed codebases. We’ll also walk through examples of codemods in action, from simple feature toggle removal to more complex React component refactorings.*

## Introduction

Imagine you’re a library developer, and you’ve created a brilliant utility that hundreds of thousands of other developers use every day. It works perfectly, and people love it. However, as is often the case, some usages have emerged that are beyond your original design. Now, you need to extend one of your APIs by adding another parameter or changing the existing signature to fix certain edge cases. But, with so many users relying on your library, how can you roll out these breaking changes without causing headaches?

This is where **codemods** come in—a powerful tool for automating large-scale code transformations, allowing developers to introduce breaking API changes, refactor legacy codebases, and maintain code hygiene with minimal manual effort.

In this article, we’ll explore what codemods are and the tools you can use to create them, such as **jscodeshift**, **Hypermod**, and [codemod.com](http://codemod.com/). We’ll walk through real-world examples, from cleaning up feature toggles to refactoring component hierarchies. You’ll also learn how to break down complex transformations into smaller, testable pieces—a practice known as codemod composition—to ensure flexibility and maintainability. Additionally, we’ll dive into the history of codemods, discussing where they originated, how they’ve evolved, and their current role in modern development workflows.

By the end, you’ll see how codemods can become a vital part of your toolkit for managing large-scale codebases, helping you keep your code clean and maintainable while handling even the most challenging refactoring tasks.

### Breaking Changes in APIs

Returning to the scenario of the library developer, after the initial release, new usage patterns begin to emerge, prompting the need to extend an API—perhaps by adding a parameter or modifying a function signature to address newly discovered edge cases.

For simple changes, a basic find-and-replace could work, or in more complex cases, you might resort to using tools like `sed` or `awk`. However, when your library is widely adopted, the scope of such a change becomes harder to manage. You can’t be sure how extensively the modification will impact your users, and the last thing you want is to break existing functionality that doesn’t require an update.

A common approach is to announce the breaking change, release a new version, and ask users to migrate at their own pace. But this workflow, while familiar, isn't ideal. It often doesn't scale well, especially for major shifts. Consider React’s transition from class components to function components with hooks—a paradigm shift that took years for large codebases to fully adopt. By the time teams managed to migrate, more breaking changes were often already on the horizon.

For library developers, this situation creates a burden. Maintaining multiple older versions to support users who haven’t migrated is both costly and time-consuming. For users, frequent changes risk eroding trust. They may hesitate to upgrade or start exploring more stable alternatives, perpetuating the cycle.

But what if you could help users manage these changes automatically? What if you could release a tool alongside your update that refactors their code for them—renaming functions, updating parameters, and removing unused code without requiring manual intervention?

That’s where **codemods** come in. Several libraries, including React and Next.js, have already embraced codemods to smooth the path of version bumps. For example, React provides codemods to handle the migration from older API patterns, like the old Context API, to newer ones.

So, what exactly is codemod we’re talking about here?

### What is a Codemod?

A **codemod** (code modification) is an automated script that transforms code to adhere to new APIs, syntax, or coding standards. Codemods leverage Abstract Syntax Tree (AST) manipulation to apply precise, large-scale changes consistently across codebases. It is originated at Facebook, initially developed to help their engineers manage large-scale refactoring tasks across vast codebases like React and other internal libraries. As the company scaled, keeping the codebase up to date with new APIs and coding patterns became challenging. 

Manually updating thousands of files across different repositories was inefficient and error-prone, so the concept of codemods—automated scripts that transform code—was introduced to tackle this problem.

The process typically involves three main steps:

1. **Parsing** the code into an AST, where each part of the code is represented as a tree structure.
2. **Modifying** the tree by applying a transformation, such as renaming a function or changing parameters.
3. **Rewriting** the modified tree back into the source code.

By using this approach, codemods ensure that changes are applied consistently across every file in a codebase, reducing the chance of human error. Codemods can also handle complex refactoring scenarios, such as changes to deeply nested structures or removing deprecated API usage.

If we visualise the process a bit, it would be something like this:

![codemod-process.png](Refactoring%20with%20Codemods%20to%20Automate%20API%20Changes%204836df67b2a54c89be8f40406da0fa9a/codemod-process.png)

*Now that we have a basic understanding of what codemods are, let’s look at the tools available to help you create and run them.*

### Tools for Codemods

1. **jscodeshift**: One of the most popular tools for writing codemods is **jscodeshift**, a toolkit maintained by Facebook. It simplifies the creation of codemods by providing a powerful API to manipulate ASTs. With jscodeshift, developers can search for specific patterns in the code and apply transformations at scale.
    
    You can use `jscodeshift` to identify and replace deprecated API calls with updated versions across an entire project.
    
2. **Hypermod**: **Hypermod** introduces AI assistance to the codemod writing process. Instead of manually crafting the codemod logic, developers can describe the desired transformation in plain English, and Hypermod will generate the codemod using `jscodeshift` for you. This makes codemod creation accessible even for developers who may not be familiar with AST manipulation.
    
    If you want to remove all instances of a specific feature toggle from your codebase, you can describe the task in Hypermod, and it will generate the required codemod automatically.
    
3. [**codemod.com**](http://codemod.com/): This community-driven platform allows developers to share and discover codemods. If you need a specific codemod for a common refactoring task or migration, you can search for existing codemods on [codemod.com](http://codemod.com/) Alternatively, you can publish codemods you’ve created to help others in the developer community.
    
    If you’re migrating an API and need a codemod to handle it, you can find a pre-built codemod on [codemod.com](http://codemod.com/), saving time on writing one from scratch.
    

These tools make codemods more accessible, even for complex transformations, and are invaluable for applying changes across hundreds or thousands of files.

### Clean A Stale Feature Toggle

Let’s start with a simple yet practical example to demonstrate the power of codemods. Imagine you’re using a [feature toggle](https://martinfowler.com/articles/feature-toggles.html) in your codebase to control the release of unfinished or experimental features. After the feature is live in production and working as expected, the next logical step is to clean up the toggle and any related logic.

For instance, consider the following code:

```tsx
const data = featureToggle('feature-x') ? {name: 'Feature X'} : undefined;
```

Once the feature is fully released and no longer needs a toggle, this can be simplified to:

```tsx
const data = { name: 'Feature X' };
```

The task involves finding all instances of `featureToggle` in the codebase, checking whether the toggle refers to `feature-x`, and then removing the conditional logic surrounding it. At the same time, we need to ensure that other feature toggles (like `feature-y`, which may still be in development) remain untouched. This means the codemod needs to *understand* the structure of the code to apply changes selectively.

### Understanding the AST

Before we dive into writing the codemod, let’s break down how this specific code snippet looks in an AST:

```tsx
{
  "program": {
    "type": "Program",
    "body": [
      {
        "type": "VariableDeclaration",
        "declarations": [
          {
            "type": "VariableDeclarator",
            "id": {
              "type": "Identifier",
              "name": "data"
            },
            "init": {
              "type": "ConditionalExpression",
              "test": {
                "type": "CallExpression",
                "callee": {
                  "type": "Identifier",
                  "name": "featureToggle"
                },
                "arguments": [
                  {
                    "type": "StringLiteral",
                    "value": "feature-x"
                  }
                ]
              },
              "consequent": {
                "type": "ObjectExpression",
                "properties": [
                  {
                    "type": "ObjectProperty",
                    "key": {
                      "type": "Identifier",
                      "name": "name"
                    },
                    "value": {
                      "type": "StringLiteral",
                      "value": "Feature X"
                    }
                  }
                ]
              },
              "alternate": {
                "type": "Identifier",
                "name": "undefined"
              }
            }
          }
        ],
        "kind": "const"
      }
    ]
  }
}
```

In this AST representation, the variable `data` is assigned using a **ConditionalExpression**. The **test** part of the expression calls `featureToggle('feature-x')`. If the test returns `true`, the **consequent** branch assigns `{name: 'Feature X'}` to `data`. If `false`, the **alternate** branch assigns `undefined` instead.

For a task that has clear input and output definition, I prefer to write test first and then implement it, and then add a few more specific variations (like use featureToggle call inside a if statement), and then extend it until it covers all the possible cases.

It’s pretty easy to do it with jscodeshfit as it has a few util that help us to write test to verify how a codemod should behalf:

```tsx
const transform = require("../remove-feature-toggle-x");

defineInlineTest(
  transform,
  {},
  `
  const data = featureToggle('feature-x') ? {name: 'Feature X'} : undefined;
  `,
  `
  const data = {name: 'Feature X'};
  `,
  "delete the surrounding conditional operator"
);
```

The `defineInlineTest` here is from jscodeshift library, and you can basically define the input and expected output, also a string to describe the intent of the test. There are a few other options (like the parser when transforming the tree) but we’ll skip them for now. And now if we run the test with a normal `jest` command, it fails as we don’t have any code written yet.

### Writing the Codemod

Firstly, let’s define a *transform* first. The simplest way to do it is to create a file `transform.js` with the following code structure:

```tsx
module.exports = function(fileInfo, api, options) {
  const j = api.jscodeshift;
  const root = j(fileInfo.source);

	// manipulate the tree nodes here
	
  return root.toSource();
};
```

It reads file into a tree and then allow us to use jscodeshift tree API to query, add or modify the tree nodes. In the end we can call `.toSource` to output the tree back to source code.

With the structure above in place, we now can start to implement the transform steps:

1. Finds all instances of `featureToggle`.
2. Verifies that the argument passed is `'feature-x'`.
3. Replaces the entire conditional expression with the **consequent** part, effectively removing the toggle.

Here’s how we can achieve this using `jscodeshift`:

```tsx
module.exports = function (fileInfo, api, options) {
  const j = api.jscodeshift;
  const root = j(fileInfo.source);

  // Find ConditionalExpression where the test is featureToggle('feature-x')
  root
    .find(j.ConditionalExpression, {
      test: {
        callee: { name: "featureToggle" },
        arguments: [{ value: "feature-x" }],
      },
    })
    .forEach((path) => {
      // Replace the ConditionalExpression with the 'consequent'
      j(path).replaceWith(path.node.consequent);
    });

  return root.toSource();
};
```

This codemod:

- Looks for `ConditionalExpression` nodes where the test calls `featureToggle('feature-x')`.
- When found, it replaces the entire conditional expression with the **consequent** (in this case, `{ name: 'Feature X' }`), effectively removing the toggle logic and leaving the clean, simplified code behind.

The example above gives you a clear idea of what a codemod looks like and how easily you can create a useful transformation that can be applied to a large codebase, significantly reducing manual effort.

### Codemods Improve Code Quality and Maintainability

Codemods aren’t just useful for managing breaking API changes—they can significantly improve code quality and maintainability. As codebases evolve, they often accumulate technical debt, including outdated feature toggles, deprecated methods, or tightly coupled components. Manually refactoring these pieces of code can be time-consuming and error-prone.

By automating such refactoring tasks, codemods ensure that your codebase remains clean and free of legacy patterns. Regularly applying codemods allows you to enforce new coding standards, remove unused code, and modernize your codebase without having to touch every file manually.

### Refactoring an Avatar Component

Let’s consider a more complex case. Suppose you’re working with a design system that includes an `Avatar` component. This component is tightly coupled to a `Tooltip`. Whenever a user passes a `name` prop into `Avatar`, it automatically wraps the avatar with a tooltip. 

![avatar-refactoring-trans.png](Refactoring%20with%20Codemods%20to%20Automate%20API%20Changes%204836df67b2a54c89be8f40406da0fa9a/avatar-refactoring-trans.png)

This coupling can limit flexibility and force developers to include unnecessary `Tooltip` dependencies even when they don't want to use them.

Here’s the existing `Avatar` implementation:

```tsx
import { Tooltip } from "@design-system/tooltip";

const Avatar = ({ name, image }: AvatarProps) => {
  if (name) {
    return (
      <Tooltip content={name}>
        <CircleImage image={image} />
      </Tooltip>
    );
  }

  return <CircleImage image={image} />;
};
```

The goal is to decouple the `Tooltip` from `Avatar` and give developers more flexibility by allowing them to decide whether to use a `Tooltip`. The refactored version of `Avatar` will simply render the image, and users can wrap it in a `Tooltip` as needed:

```tsx
const Avatar = ({ image }: AvatarProps) => {
  return <CircleImage image={image} />;
};
```

Here’s how a user can apply the `Tooltip` manually, if necessary:

```tsx
import { Tooltip } from "@design-system/tooltip";
import { Avatar } from "@design-system/avatar";

const UserProfile = () => {
  return (
    <Tooltip content="Juntao Qiu">
      <Avatar image="/juntao.qiu.avatar.png" />
    </Tooltip>
  );
};
```

Now, the problem is that there could be hundreds of `Avatar` usages across various files in the codebase. Manually refactoring every instance of `Avatar` would be a significant challenge, so we need to define a codemod here help us to finish the task.

We can use tools like ast-explorer to inspect the component, to see which nodes is the one we’re targeting. For example, the above JSX code is parsed as:

```tsx
{
  "type": "Program",
  "body": [
    {
      "type": "ExpressionStatement",
      "expression": {
        "type": "JSXElement",
        "openingElement": {
          "type": "JSXOpeningElement",
          "attributes": [
            {
              "type": "JSXAttribute",
              "name": {
                "type": "JSXIdentifier",
                "name": "name"
              },
              "value": {
                "type": "Literal",
                "value": "Juntao Qiu",
              }
            },
            {
              "type": "JSXAttribute",
              "name": {
                "type": "JSXIdentifier",
                "name": "image"
              },
              "value": {
                "type": "Literal",
                "value": "/juntao.qiu.avatar.png",
              }
            }
          ],
          "name": {
            "type": "JSXIdentifier",
            "name": "Avatar"
          },
          "selfClosing": true
        },
        "closingElement": null,
        "children": []
      }
    }
  ]
}
```

Based on the parsed tree, we can break down the transform into a few smaller tasks:

- Find `Avatar` usage in the component tree
- Check if `name` prop is presented
    - If it doesn’t we do nothing
    - Otherwise
        - Create a `Tooltip` node
        - Add `name` to the `Tooltip`
        - Remove name from `Avatar`
        - Add `Avatar` as a child node of `Tooltip`
        - Use the new `Tooltip` to replace the original `Avatar` node
- Do the above to the next `Avatar` usage if any

### Writing the Codemod

Let’s start from the find `Avatar` usage, like the `featureToggle` we have seen above, we can use `root.find` with search certiria to find all `Avatar` nodes:

```tsx
root
  .find(j.JSXElement, {
    openingElement: { name: { name: "Avatar" } },
  })
  .forEach((path) => {
    // now we can deal with each Avatar instance
  });
```

Then we could check if the `name` is present:

```tsx
root
  .find(j.JSXElement, {
    openingElement: { name: { name: "Avatar" } },
  })
  .forEach((path) => {
    const avatarNode = path.node;

    const nameAttr = avatarNode.openingElement.attributes.find(
      (attr) => attr.name.name === "name",
    );

    if (nameAttr) {
      const tooltipElement = createTooltipElement(
        nameAttr.value.value,
        avatarNode,
      );
      j(path).replaceWith(tooltipElement);
    }
  });
```

And for the `createTooltipElement`, we use jscodeshift API to create a new JSX node, with name and avatar as child node, then `replaceWith` the current `path`.

I can easily test out the input and output in [Hypermod](https://www.hypermod.io/):

![Screenshot 2024-09-13 at 4.18.19 PM.png](Refactoring%20with%20Codemods%20to%20Automate%20API%20Changes%204836df67b2a54c89be8f40406da0fa9a/Screenshot_2024-09-13_at_4.18.19_PM.png)

This codemod searches for all instances of `Avatar` in the codebase. If a `name` prop is found, the codemod removes the `name` prop from `Avatar`, wraps the `Avatar` inside a `Tooltip`, and passes the `name` prop to the `Tooltip`.

## The Unhappy Paths

As a diligent developer, you're aware that the happy path is only a small part of the full picture. There are many scenarios to consider when writing a transformation script to handle code automatically.

Firstly, people write code in a variety of styles. For example, someone might import the `Avatar` component but give it a different name:

```tsx
import { Avatar as AKAvatar } from "@design-system/avatar";

const UserInfo = () => (
  <AKAvatar name="Juntao Qiu" image="/juntao.qiu.avatar.png" />
);
```

A simple text search for `Avatar` won’t work here. You’ll need to detect the alias and apply the transformation using the correct name.

Another example arises when dealing with `Tooltip` imports. If the file already imports `Tooltip` and uses an alias, the codemod needs to detect that alias and apply the changes accordingly. You can't assume the `Tooltip` component will always be named `Tooltip`.

## Applying Composition

As you can see, there are plenty of edge cases to handle, especially in codebases beyond your control—such as external dependencies. This complexity means that using codemods requires careful supervision and review of the results.

However, if your codebase has some standardization tools in place, like a linter, you can take advantage of those to narrow down the code, making the transformation easier and reducing edge cases.

Let’s take a simple example:

```tsx
import { featureToggle } from "./utils/featureToggle";

const convertOld = (input: string) => {
  return input.toLowerCase();
};

const convertNew = (input: string) => {
  return input.toUpperCase();
};

const result = featureToggle("feature-convert-new")
  ? convertNew("Hello, world")
  : convertOld("Hello, world");

console.log(result);
```

After running the codemod, we want the result to look like this:

```tsx
const convertNew = (input: string) => {
  return input.toUpperCase();
};

const result = convertNew("Hello, world");

console.log(result);
```

Beyond removing the feature toggle logic, there are a few other tasks to handle:

- Removing the unused `convertOld` function.
- Cleaning up the unused `featureToggle` import.

Of course, you could write one large codemod to handle everything in a single pass and test it together. But a more maintainable approach is to treat codemod logic like product code: break the task into smaller, independent pieces.

### Breaking It Down

One particularly effective pattern is breaking the transformation down into smaller codemods and then composing them. The advantage of this approach is that each transformation can be tested individually, covering different cases without interference. Moreover, it allows you to reuse and compose them for different purposes.

For instance, you might break it down like this:

- A transformation to remove a specific feature toggle.
- Another transformation to clean up unused imports.
- A transformation to remove unused function declarations.

By composing these, you can create a pipeline of transformations:

```tsx
import { removeFeatureToggle } from "./remove-feature-toggle";
import { removeUnusedImport } from "./remove-unused-import";
import { createTransformer } from "./utils";
import { removeUnusedFunction } from "./remove-unused-function";

const removeFeatureConvertNew = removeFeatureToggle("feature-convert-new");

const transform = createTransformer([
  removeFeatureConvertNew,
  removeUnusedImport,
  removeUnusedFunction,
]);

export default transform;
```

In this pipeline, the transformation:

1. Removes the `feature-convert-new` toggle.
2. Cleans up the unused `import` statement.
3. Removes the `convertOld` function since it’s no longer used.

![Refactoring%20with%20Codemods%20to%20Automate%20API%20Changes%204836df67b2a54c89be8f40406da0fa9a/codemod-process-transforms.png](Refactoring%20with%20Codemods%20to%20Automate%20API%20Changes%204836df67b2a54c89be8f40406da0fa9a/codemod-process-transforms.png)

You could also extract additional codemods as needed, combining them in various orders depending on the desired outcome.

![Refactoring%20with%20Codemods%20to%20Automate%20API%20Changes%204836df67b2a54c89be8f40406da0fa9a/codemod-process-transforms-2.png](Refactoring%20with%20Codemods%20to%20Automate%20API%20Changes%204836df67b2a54c89be8f40406da0fa9a/codemod-process-transforms-2.png)

### The `createTransformer` Function

The implementation of the `createTransformer` function is relatively straightforward. It acts as a higher-order function that takes a list of smaller transform functions, iterates through the list to apply them to the root AST, and finally converts the modified AST back to source code.

```tsx
import { API, Collection, FileInfo, JSCodeshift, Options } from "jscodeshift";

type TransformFunction = { (j: JSCodeshift, root: Collection): void };

const createTransformer =
  (transforms: TransformFunction[]) =>
  (fileInfo: FileInfo, api: API, options: Options) => {
    const j = api.jscodeshift;
    const root = j(fileInfo.source);

    transforms.forEach((transform) => transform(j, root));
    return root.toSource(options.printOptions || { quote: "single" });
  };

export { createTransformer };
```

### Benefits of Codemod Composition

Composing codemods has several advantages:

1. **Reusability**: You can reuse individual transformations across different projects or combine them in new ways to suit different needs.
2. **Testability**: By breaking transformations into smaller pieces, you can test each part in isolation, leading to fewer bugs and more reliable refactors.
3. **Flexibility**: Composable codemods allow you to tackle complex tasks incrementally, making large-scale refactors more manageable.
4. **Maintainability**: Smaller, focused codemods are easier to maintain, debug, and extend as your codebase evolves.

By composing smaller transformations, you ensure that your codemods remain flexible and easy to adapt, whether you're tackling small, incremental changes or large, complex refactors across a vast codebase.

## Codemods in Other Languages

While this example focuses on JavaScript and JSX using **jscodeshift**, codemods can also be applied to other languages. For instance, **JavaParser** ([https://javaparser.org/](https://javaparser.org/)) offers a similar mechanism in Java, using AST manipulation to refactor Java code. This can be useful when making breaking API changes or refactoring large Java codebases in a structured, automated way.