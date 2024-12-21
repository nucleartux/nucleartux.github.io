---
title: "Using TypeScript in a monorepo"
description: "How to configure TypeScript in a monorepo in a right way"
pubDate: "December 21 2024"
---

I usually try not to write about things already covered in documentation.

However, monorepo configuration in TypeScript always seemed complicated to me. I also noticed that many projects I have seen suffer from the same problem — the configuration is not optimal and could be improved.

Well, it’s actually much easier than I thought.

For example, I have the following project structure:

```
tsconfig.json
packages
├── app1
│   ├── src
│   └── tsconfig.json
├── app2
│   ├── src
│   └── tsconfig.json
├── shared
│   ├── src
│   └── tsconfig.json
```

Where `shared` is used by both `app1` and `app2`.

Previously, I had a configuration that looked like this:

```json
// packages/app1/tsconfig.json
{
    "include": ["src", "../shared/src"]
}
// packages/app2/tsconfig.json
{
    "include": ["src", "../shared/src"]
}
// packages/shared/tsconfig.json
{
    "include": ["src"]
}
```

Notice this `include` part, which exists in both apps. For TypeScript, it basically means that every time it checks one of these projects, it includes `shared` in the check.

To run checks for these projects, I used the following commands:

```shell
tsc -p packages/app1
```

and

```shell
tsc -p packages/app2
```

You have probably already figured out where the problem is. When I run the first command, both `packages/app1` and `packages/shared` are checked. When I run the second command, both `packages/app2` and `packages/shared` are checked.

How can I avoid wasting time compiling the same folder twice? It turned out that everything is much simpler than I thought.

I needed to do some modifications:

1. Add the composite setting to the `shared` project. This means that the project will be used as part of another.

```json
// packages/shared/tsconfig.json
{
  "compilerOptions": {
    "emitDeclarationOnly": true,
    "composite": true
  },
  "include": ["src"]
}
```

2. Add `references` and remove this folder from `include`. You also need to add the `outDir` and `emit` settings if you haven’t done so already.

```json
// packages/app1/tsconfig.json and packages/app2/tsconfig.json
{
  "compilerOptions": {
    "outDir": "dist",
    "emit": true
  },
  "references": [
    {
      "path": "../shared"
    }
  ]
}
```

You can safely add this `dist` folder to your `.gitignore` and hide it in your code editor. You don’t need these files while writing code, as your code editor will generate them automatically.

After this changes `shared` folder will be compiled only once.

And here is bonus tip. You can create the following TS config in the root of your project:

```json
// tsconfig.json
{
  "files": [],
  "references": [
    {
      "path": "packages/app1"
    },
    {
      "path": "packages/app2"
    },
    {
      "path": "packages/shared"
    }
  ]
}
```

and run check with command (please not that it uses `-b` flag instead of `-p`):

```shell
tsc -b tsconfig.json
```

With this command TS will automatically run typechecking for all referenced projects, but your `composite` folders will be checked only once.

With modifications described I was able to optimize build time from 49s to 13s.

Most important note: If you have a monorepo where each package is independent (i.e., it doesn't use other packages) or an application that uses all packages at once (making it just a monorepo for the sake of having a monorepo), you DON'T NEED such a configuration. You should probably use the `include` option.

[Leave a comment](https://bsky.app/profile/adrov.me/post/3ldtj6qigbs2j)
