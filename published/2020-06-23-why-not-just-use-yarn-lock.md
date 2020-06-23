---
title: npm v7 Series - Why Keep `package-lock.json`?
date: 2020-06-23T17:00:00.000Z
tags:
  - npm7
  - cli
---

[<< Arborist Deep Dive](https://blog.npmjs.org/post/618653678433435649/npm-v7-series-arborist-deep-dive) [>> `npm ci` vs `npm install`]()

One common question we've gotten a few times now, once we announce that npm
v7 will include support for `yarn.lock` files, is "Why keep
`package-lock.json` at all, then?  Why not just use `yarn.lock` only?"

The simple answer is: because `yarn.lock` doesn't fully address npm's
needs, and relying on it exclusively would limit our ability to produce
optimal package installs or add features in the future.

## Basic Structure of a `yarn.lock` File

A `yarn.lock` file is a map of requested dependency specifiers to metadata
describing their resolution.  For example:

```
mkdirp@1.x:
  version "1.0.2"
  resolved "https://registry.yarnpkg.com/mkdirp/-/mkdirp-1.0.2.tgz#5ccd93437619ca7050b538573fc918327eba98fb"
  integrity sha512-N2REVrJ/X/jGPfit2d7zea2J1pf7EAR5chIUcfHffAZ7gmlam5U65sAm76+o4ntQbSRdTjYf7qZz3chuHlwXEA==
```

This says "Any dependency on `mkdirp@1.x` should resolve to _this_ exact
thing".  If multiple packages depend on `mkdirp@1.x`, they'll all get the
same resolution.

In npm v7, if a `yarn.lock` file exists, npm will use the metadata it
contains.  The `resolved` values will tell it where to fetch packages from,
and the `integrity` will be used to check that the result matches
expectations.  If packages are added or removed, then the `yarn.lock` file
will be updated.

npm will still create a `package-lock.json` file, and if a
`package-lock.json` file is present, it'll be used as the authoritative
definition of the tree shape to create.

So if it's good enough for Yarn, why doesn't npm just use that?

## Deterministic Build Results

Yarn installs are guaranteed to be deterministic given a single combination
of `yarn.lock` and Yarn version.  It is possible that a different version of
Yarn will result in a different tree layout on disk.

A `yarn.lock` file _does_ guarantee deterministic _resolutions_ of
dependencies.  For example, if `foo@1.x` resolves to `foo@1.2.3`, it'll
continue to resolve to that version number in subsequent installs for all
Yarn versions, given a consistent `yarn.lock` file.  But that (at least, in
itself) is not equivalent to guaranteeing a deterministic tree shape!

Consider this dependency graph:

```
root -> (foo@1, bar@1)
foo -> (baz@1)
bar -> (baz@2)
```

Either of these package trees would be just as correct as the other:

```
root
+-- foo
+-- bar
|   +-- baz@2
+-- baz@1

~~ OR ~~

+-- foo
|   +-- baz@1
+-- bar
+-- baz@2
```

A `yarn.lock` file can't tell you which one to use.  If the root package
(incorrectly, as it's an unlisted dep) does `require("baz")`, the result
would not be guaranteed _by the `yarn.lock` file_.  This is a form of
determinism that the `package-lock.json` file can provide, and a
`yarn.lock` file cannot.

In practice, of course, since Yarn has all the required information in the
`yarn.lock` file to make this choice, it _is_ deterministic as long as
everyone is using the same version of Yarn, so that the choice is being
made in exactly the same way.  Code doesn't change unless someone changes
it.  To its credit, Yarn is smart enough to not be subject to discrepancies
in package manifest load times when building the tree, or else
determinism would not be guaranteed.

As this is defined by the particulars of Yarn's algorithm rather than by
the data structure on disk (which does not identify the algorithm to be
used), that determinism guarantee is fundamentally weaker than what a
`package-lock.json` provides by fully specifying the shape of the package
tree on disk.

In other words, the Yarn tree building contract is split between the
`yarn.lock` file and the implementation of Yarn itself.  The npm tree
building contract is entirely specified by the `package-lock.json` file.
This makes it much harder for us to break by accident across npm versions,
and if we do (whether by mistake or on purpose), the change will be
reflected in the file in source control.

## Nesting and Deduplication

Furthermore, there is a class of nesting and deduplication cases where the
`yarn.lock` file does not accurately reflect the resolutions that will be
used by npm in practice, even when npm does use it as a source of metadata.
While npm uses the `yarn.lock` file as a reliable source of information, it
does not treat it as an authoritative set of constraints.

In some cases Yarn produces a tree with excessive duplication, which we
don't want to do.  So, following the Yarn algorithm exactly isn't ideal in
these cases.

Consider [this](http://npm.im/@isaacs/nested-yarn-lock-test) dependency
graph:

```
root -> (x@1.x, y@1.x, z@1.x)
x@1.1.0 -> ()
x@1.2.0 -> ()
y@1.0.0 -> (x@1.1, z@2.x)
z@1.0.0 -> ()
z@2.0.0 -> (x@1.x)
```

The root project depends on version `1.x` of `x`, `y`, and `z`.  The `y`
package depends on `x@1.1` and `z@2`.  `z` at version 1 has no
dependencies, but `z` at version 2 depends on `x@1.x`.

The resulting tree shape that npm produces looks like this:

```
root (x@1.x, y@1.x, z@1.x) <-- x@1.x dep here
+-- x 1.2.0                <-- x@1.x resolves to 1.2.0
+-- y (x@1.1, z@2.x)
|   +-- x 1.1.0            <-- x@1.x resolves to 1.1.0
|   +-- z 2.0.0 (x@1.x)    <-- x@1.x dep here
+-- z 1.0.0
```

`z@2.0.0` depends on `x@1.x`, and so does the `root` project.  The yarn
lock file maps `x@1.x` to `1.2.0`.  However, the dependency from the `z`
package, which also specifies `x@1.x`, will get `x@1.1.0` instead.

That is, even though the `x@1.x` dependency has a resolution in the
`yarn.lock` file stipulating that it should resolve to version `1.2.0`,
there is a _second_ `x@1.x` resolution which instead resolves to `1.1.0`.

If run with the `--prefer-dedupe` flag on npm, it'd go a step further, and
only install a single instance of `x`, like this:

```
root (x@1.x, y@1.x, z@1.x)
+-- x 1.1.0                <-- x@1.x resolves to 1.1.0 for everyone
+-- y (x@1.1, z@2.x)
|   +-- z 2.0.0 (x@1.x)
+-- z 1.0.0
```

This minimizes duplication, and the resulting package tree is captured in
the `package-lock.json` file.

Because `yarn.lock` only locks down _resolutions_ instead of locking down
the resulting package tree, Yarn produces this tree instead:

```
root (x@1.x, y@1.x, z@1.x) <-- x@1.x dep here
+-- x 1.2.0                <-- x@1.x resolves to 1.2.0
+-- y (x@1.1, z@2.x)
|   +-- x 1.1.0            <-- x@1.x resolves to 1.1.0
|   +-- z 2.0.0 (x@1.x)    <-- x@1.1.0 would be fine, but...
|       +-- x 1.2.0        <-- Yarn dupes to satisfy yarn.lock resolution
+-- z 1.0.0
```

The `x` package appears three times in the Yarn implementation, twice in
the default npm implementation, and only once (albeit, not the latest and
greatest version) in npm's `--prefer-dedupe` algorithm.

All three resulting trees are "correct", in the sense that every package is
getting a version of their dependencies that matches their stated
requirements.  But, we do not want to create package trees with excessive
duplication.  Consider what would happen if `x` was a large package with a
lot of dependencies of its own!

So, the only way that npm can optimize a package tree, while maintaining
deterministic reproducible builds, is to use a fundamentally different sort
of lock file.

## Capturing Results of User Intent

As mentioned above, in npm v7, a user can use `--prefer-dedupe` to have the
tree generation algorithm prefer deduplication rather than always updating
to latest.  This is usually best in any scenario where duplication should
be minimized.

If that config flag is set, then the resulting tree for the example above
would look like this:

```
root (x@1.x, y@1.x, z@1.x) <-- x@1.x dep here
+-- x 1.1.0                <-- x@1.x resolves to 1.1.0 for everyone
+-- y (x@1.1, z@2.x)
|   +-- z 2.0.0 (x@1.x)    <-- x@1.x dep here
+-- z 1.0.0
```

In this case, npm sees that, even though `x@1.2.0` is the latest package
version that satisfies the `x@1.x` requirement, choosing `x@1.1.0` instead
would still be acceptable, and would result in less duplication.

Without capturing the tree _shape_ in the lockfile, every user working on
the project would have to configure their client exactly the same way to
get the same results.  When the "implementation" can be changed by the user
in this way, this gives them a lot of power to optimize for their specific
conditions.  But, it also makes deterministic builds impossible if the
contract is implementation-dependent, which `yarn.lock` is.

Other examples where the algorithm would be different are:

- `--legacy-peer-deps`, which tells npm to completely ignore
  `peerDependencies`
- `--legacy-bundling`, which tells npm to not even try to flatten the tree
- `--global-style`, which installs all transitive dependencies nested under
  their top-level dependents

Capturing the result of resolutions, and relying on the algorithm to be
consistent, doesn't work when we give the user the ability to tweak the
package installation algorithm in use.

Locking down the resulting tree shape allows us to ship features like this
without breaking our contract to provide deterministic reproducible builds.

## Performance and Data Completeness

The `package-lock.json` file is not only useful for ensuring
deterministically reproducible builds.  We also lean on it to track and
store package metadata, saving considerably on `package.json` reads and
requests to the registry.  Since the `yarn.lock` file is so limited, it
doesn't have the metadata that we need to load on a regular basis.

In npm v7, the `package-lock.json` file contains everything npm will need
to fully build the package tree.  (This data is spread out in npm v6, so
when we see an older lockfile, we have to do a bit of extra digging up
front, but that's a one-time hit.)

So, even if it _did_ capture tree shape, we'd still have to use a file
other than `yarn.lock` to track this extra metadata.

## Future Possibilities

Approaches to package dependency layout on disk such as pnpm, yarn 2/berry,
and Yarn's PnP, can change the context of this calculation considerably.

We intend to explore a virtual file system approach in npm v8, modeled on
Tink, the proof of concept Kat Marchán wrote in 2019.  We've also talked
about migrating to something like pnpm's layout structure, though this is
in some ways an even bigger breaking change than Tink would be.

If all dependencies are stored in a central location, and only simulated in
their nested locations via symbolic links or a virtual filesystem, then
modeling the tree shape is far less of a concern.  However, we'd still need
more metadata than the `yarn.lock` file provides, and thus, it would make
more sense to update and streamline our existing `package-lock.json` format
rather than rely on `yarn.lock`.

## This is Not a "Considered Harmful" Post

I want to be very clear that, as far as I've ever been able to determine,
Yarn reliably produces correct package dependency resolutions.  And, for a
given Yarn version (all recent Yarn versions, as of this writing), it _is_
fully deterministic, just like npm.

While it is good that the `yarn.lock` file is sufficient for a specific
version of Yarn to generate deterministic builds, relying on an
implementation-dependent contract is not acceptable for use across multiple
tools.  This is all the more true by virtue of the fact that the
implementation and `yarn.lock` format are not documented or specified in
any formal way.  (This isn't a dig on Yarn; npm's aren't either.  Doing so
will be quite a bit of work.)

The best way to fully ensure build reliability and strict determinism for
the long term is to lock down the _results_ of the build process in the
contract itself, rather than naively trusting that future implementations
will continue to make the same choices, and effectively limiting our
ability to design an optimized package tree.

Deviations from that contract must be a result of explicit user intent, and
self-documenting by virtue of updating the saved contract on completion.

Only `package-lock.json` or something like it can provide this
functionality for npm.

[<< Arborist Deep Dive](https://blog.npmjs.org/post/618653678433435649/npm-v7-series-arborist-deep-dive) [>> `npm ci` vs `npm install`]()
