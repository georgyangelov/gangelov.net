---
title: "Why named exports?"
date: 2022-07-04T21:36:00+03:00
draft: false
description: Some notes on why I don't like default exports
---

## Which is better: default export or named export?

Please use named exports unless you're absolutely 100% certain a named export wouldn't work for you.

## Accidental name changes

Exporting something as default basically means exporting it anonymously. Yes, it may have a name internally, but that doesn't matter for the export. That means it's very common to see this:

```ts
// views/user.ts
export default function UserView() {
  // ...
}

// routes/users.ts
import UsersView from '../views/user'

// ...
```

Notice how `UserView` suddenly got imported as `UsersView`?

## Default exports are not easily searchable

What if someone wants to find all references to the `UserView` function? Most JS users would just "Find All" in the whole project for `UserView`. And the `views/user.ts` won't come up in the results, because of the extra `s`.

## Default exports are not easily renameable

What if someone decides to rename `UserView`? Using some language tools like VSCode's TS support, you can right-click the function and rename it. In the `export default` case you will only rename the definition and not the usages. So you'll be introducing a name discrepancy.

## So what's the issue with different names?

The issue is that it's one more thing you need to keep track of mentally. If I see `<User>` in the code I shouldn't have to go to the top of the file to see if that's `UserView`, `UserPanel` or something else. We have a ton of those little "attention eaters" already, so why introduce another one?

## Auto-import is inconsistent

At least in VSCode, auto-import of default exports is inconsistent. It's understandable - if I type `User`, how would VSCode know I want `UserService`, `UserView` or `BlogPost` for that matter? It does a pretty good job of remembering choices but why go through all the trouble? If you're just going to import `UserView` as `UserView` (and you should), why not just make it a named export?

## Re-exporting introduces another inconsistency

Sometimes you have filed which aggregate exports. Maybe you're implementing a library. And there's a file where you have this:

```ts
export * from 'views/user'
export * from 'views/user-list'
```

but since you have default exports, you need to do this instead:

```ts
export *, { default as UserView } from 'views/user'
export *, { default as UsersList } from 'views/user-list'
```

- Why would I need to duplicate the name in one more place?
- It's no longer a default export, so we gave it a name regardless, so why not give it a name wherever it's defined?
- ...and I made the "extra s" error again, whoops.

## Want to export more than one thing?

You're going to use named exports anyway. So why not just export all things as named? This way you won't have to make the arbitrary decision that one of the exports is somehow more important than the rest of them.

## Named exports make the name stays the same

And this is really the point. All imports will have the same name, so by definition all problems described above are solved. If you absolutely have to rename something, you can still do it (`import { UserView as User } from '...'`) but now it's opt-in and can't be done accidentally.

So default exports or named exports? **Definitely named exports.**
