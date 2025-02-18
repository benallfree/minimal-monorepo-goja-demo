This repo demonstrates that goja-nodejs's module resolution as used by PocketBase does not properly traverse into parent directories when resolving modules.

In monorepos, packages are hoisted to the monorepo root `./node_modules` when possible. These are not visible to PocketBase projects running in subdirectories.

## Installation

```
bun i
```

## Test case

We chose the [makred](https://www.npmjs.com/package/marked) NPM package because it is known to be compatible with Goja.

## Working demo

1. In the root directory, there is a `pb_hooks/index.pb.js` which is simply `require('marked')`.
2. From the root, run `pocketbase serve --dir=pb_data --dev` and observe that the module is loaded successfully.

## Broken demo

```
cd projects/broken

# node_modules should NOT exist because it's a monorepo and all
# packages were hoisted to the root
ls node_modules

pocketbase serve --dir=pb_data --dev
```

Observe the error:

```
Failed to execute index.pb.js:
 - GoError: Invalid module at github.com/dop251/goja_nodejs/require.(*RequireModule).require-fm (native)
```

This error comes from [getCurrentModulePath](https://github.com/benallfree/goja_nodejs/blob/28407dfeec35522c06b2e296fdeefc42e6b6b78f/require/resolve.go#L229) in goja-nodejs where `frames[1].SrcName()` is empty.

From [path.Dir docs](https://pkg.go.dev/path#Dir):

> If the path is empty, Dir returns ".".

Since `path.Dir("")` returns `.`, `getCurrentModulePath` returns the same.

When this happens, [parent := path.Dir(start)](https://github.com/benallfree/goja_nodejs/blob/28407dfeec35522c06b2e296fdeefc42e6b6b78f/require/resolve.go#L213) returns `.` because `path.Dir(".")` is `.`. This causes ancestor traversal to fail early, and parent directories are never searched.

This problem can be observed with `pnpm` as well. Any package manager that hoists packages to the root will have this problem.

_goja-nodejs note: I see no evidence that traversal ever stops when encountering a `package.json`. This may be a deviation or bug from NodeJS's module resolution._
