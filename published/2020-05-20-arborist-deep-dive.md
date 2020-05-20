---
title: npm v7 Series - Arborist
date: 2020-05-20T17:00:00.000Z
tags:
  - npm7
  - cli
  - arborist
  - tree doctor
---

[<< Introduction](https://blog.npmjs.org/post/617484925547986944/npm-v7-series-introduction)

[@npmcli/arborist](https://github.com/npm/arborist/) is the dependency tree
manager for npm, new in npm v7.  It provides facilities for doing nearly
everything that npm does with package trees, and fully replaces large parts
of the npm CLI codebase.

Way back in the summer of 2019, I stumbled upon and wrote about [an old
bug](https://blog.npmjs.org/post/186190819147/an-old-bug) buried deep in
npm's [`read-package-tree`](http://npm.im/read-package-tree) module.  At
the time, I was just trying to work out why `npm install` was so much
slower than `npm ci`, and if there was anything that could be done about
it.  Stumbling across that weird old bug, and seeing the refactoring
required to fix it, is what led eventually to Arborist.

## hold on to your butts

In the
[introduction](https://blog.npmjs.org/post/617484925547986944/npm-v7-series-introduction)
to this series, we outlined a list of topics that will be explored.  Many
of the features and changes in npm v7 are related to the refactor to use
[Arborist](http://npm.im/@npmcli/arborist) for all of npm's tree management
work.

Since so much depends on this design, today we'll be laying the groundwork
by exploring the what, why, and how of this new package management engine.

If technical deep dives aren't your thing, and you just want to know the
cool new feature stuff, feel free to skip this one :)

## tl;dr - What Does This Mean For Me?

If you like to get your hands in the npm CLI codebase, then familiarity
with Arborist's concepts and interfaces will be useful.  As we're putting
it through its paces fast and furious in npm v7 development, the docs are
somewhat lagging, and my hope is that this post provides a good primer for
future contributors.

If you don't touch the npm CLI codebase itself, but you _do_ maintain other
tools that do things with package trees and `node_modules` folders, then
you might get a lot of value out of using Arborist.

If neither of these things apply to you, then the short answer is: better
performance, more predictability, faster feature delivery, and fewer bugs.
I can't overstate how nice it is to be working with a re-architected tree
management system, redesigned with the benefit of a decade of hindsight.

## Links and Realpaths

At the risk of rehashing [that old blog
post](https://blog.npmjs.org/post/186190819147/an-old-bug), the core
problem, which has led to a lot of excess work and bugfixing in the npm CLI
codebase, is that `read-package-tree` did not properly differentiate
between symlinked dependencies and regular installed dependencies, when
creating the logical tree of nodes.

Because the Node.js module system _does_ differentiate between symlinks and
real paths, always loading dependencies based on the `require()`-ing
module's real path, this led to a lot of rather complicated workarounds to
get things right over the years, with the codebase becoming ever more
difficult to maintain as it grew.

Coming across that bug filled me with the distinct mix of dread and
excitement that comes from realizing that you've been going in the wrong
direction for a long time, and now get to start over.

## Folder _Tree_, Dependency _Graph_

It's tempting to refer to a "dependency tree" when discussing package
management.  However, in practice, the relationships between dependencies
is not strictly a _tree_ but a _graph_.  It has cycles and overlapping
relationships, so a single node can play multiple roles within the system.

When the graph is put on disk for a given program, of course, it's reified
down to a discrete tree of files on disk, which are loaded at run time and
form a graph of modules in the program itself.

A big part of the job of a package manager is to take a set of declarations
about dependency constraints, find a graph that solves the constraints, and
then reify that graph onto disk such that the resulting program will load
the right things.  Depending on the limits and features of the module
system, this might require squashing the dependency graph down to a single
instance of each dependency name, or in the case of JavaScript, figuring
out what packages need to be nested under a `node_modules` folder such that
they will override a dependency by the same name further up the folder
tree.

Building the tree to satisfy the dependency graph is what Arborist does.

## Edges as First Class Citizen

Prior approaches to this problem in npm tended to represent the tree of
nested packages on disk as an object containing child objects.  Determining
if a dependency was met was then a matter of walking up the tree to find
the first node by that name.

While this makes intuitive sense, and elegantly models the resulting
reified folder structure, it also has a few problems.

First, the logic about satisfying dependency constraints was sort of
smeared out all over the place.  If a package depends on `foo@1.x`, it's
easy enough to walk up the tree and find it, and when there was a single
kind of dependency in npm, this was sufficient for most purposes.

However, `peerDependencies` are only allowed at or above the same level as
the dependent package (except for top-of-tree nodes),
`optionalDependencies` can be missing (but can't be a _different_ version
if present), `devDependencies` are only installed for the root, `git`
dependencies are only valid if they came from the same source, and so on.
Once we started deduping the tree in npm v3, this all got much harder to
reason about.

If we want to do something like `npm prune --no-optional`, then we have to
find all the nodes in the tree that are _only_ required by virtue of being
an optional dependency.  Ie, all optional dependencies, and their
dependencies, and so on.

While representing the dependency set as a tree is easy to load from disk,
and translates nicely to the set of packages on disk, it makes it very
complicated to go from a node in the tree and ask "if I remove all optional
dependencies, will I still need this?", or even provide an answer to the
question "what other nodes in the tree are depending on this one?"  The
limitations of this data structure meant that we had to walk the package
tree numerous times throughout the lifecycle of a given npm command, which
was part of the reason why `npm install` was significantly slower than `npm
ci`.

The approach taken in Arborist represents the relationship between two
dependency nodes as a discrete category of "thing".  Each node in the
dependency set is referenced by the
[`Node`](https://github.com/npm/arborist/blob/master/lib/node.js) class (or
in the case of symlinked packages, the
[`Link`](https://github.com/npm/arborist/blob/master/lib/link.js) class).
Dependency relationships are represented by an
[`Edge`](https://github.com/npm/arborist/blob/master/lib/edge.js) class.

A `Node` object has an `edgesOut` property that maps package names to
dependency relationships.  For example, `node.edgesOut.get('foo')` would
return the `Edge` representing its dependency on a package named `'foo'`.
It also has an `edgesIn` property which is the _set_ of nodes whose
dependencies are met by it.  For representing the _tree_ of how packages
are laid out on disk, `node.children` is a map representing the contents of
a node's `node_modules` folder, and `node.parent` is a reference to the
node containing this one as a child.

`Edge` objects have a `from` node, a `to` node, a `type`, a `spec`, and
some information about whether the relationship is currently met or not.

This allows us to walk the dependency graph, or the package tree, in a very
expressive manner.  And, because we can easily find whatever we need from
any point in the dependency set, we can avoid doing multiple full package
tree walks to calculate the impact of any changes.

## Automatic Updates

Arborist uses getter/setter properties to avoid a lot of the pitfalls that
led to CLI bugs in the past.  My thinking was, the _primary_ thing we do
with package trees in npm is mutate them; why not make that easy?

When a node's `parent` member is set, everything about it is automatically
updated.  Any edges that might have been resolving to it are re-evaluated,
the `node.children` maps are updated, and so on.  Removing a package from
the tree is as easy as `node.parent = null`.  At any point, we can always
loop over `node.edgesOut` to validate its dependencies, or `node.edgesIn`
to check its dependents, without having to remember to make another call to
refresh all of those links.

While this has meant a rather elaborate `set parent` method in the Node
class, it makes all of the logic around tree handling much simpler and
harder to get wrong.  There's just nothing to forget.  Stick the objects
together, and it's always in a correct state.

This "always accurate" contract makes it much easier to work with nodes,
links, and edges.  Set a node's parent, and it's immediately updated with
the right path, all the dependency relationships are updated, etc.

There are two situations where we don't guarantee everything stays
perfectly in sync right now.  The first is the extraneous/dev/optional/peer
flags, which require another function call to set, since it requires a full
tree walk.  The second is that it's possible to directly set `node.path` in
such a way that things get out of sync.  For now, the protection is "don't
do that", but it is a spot where the internal gears are exposed and
slightly dangerous, and an area for future improvement, especially if other
applications start using Arborist in novel ways.

## Lockfile As Continually Updated Metadata

Rather than treat the lockfile as a one-off data structure to be consulted
at the beginning and serialized back out at the end, Arborist has a
[`Shrinkwrap`](https://github.com/npm/arborist/blob/master/lib/shrinkwrap.js)
class that is kept continually up to date as nodes are moved around in the
tree.

This fully abstracts out the reading and writing of lockfiles, whether they
are `package-lock.json`, `npm-shrinkwrap.json`, or `yarn.lock`, and allows
us to [no longer rely on underscore-prefixed metadata in the package.json
files of installed
dependencies](https://github.com/npm/rfcs/blob/latest/accepted/0013-no-package-json-_fields.md).

When a build of the tree is done, we can call `tree.meta.save()`, and it'll
write back to disk.  If we loaded a `yarn.lock` file and made changes to
the tree, then it'll write the updates back there as well.

## Inventory of Nodes in a Project

The `root` node represents the main project that a user is working on. It's
the package you're working in when you run `npm install`.  In a world of
symlinked dependencies and
[workspaces](https://github.com/npm/rfcs/blob/latest/accepted/0026-workspaces.md),
there may be other nodes that are the "top" of their respective sub-trees
(in the sense that they are not in a parent node's `node_modules` folder).

The `root` node is one of the most important pieces of representing a
project.

One special feature of the root node is that it contains an
[`Inventory`](https://github.com/npm/arborist/blob/master/lib/inventory.js)
that provides an easy reference to all the nodes in the project.  This
allows us to walk over all the nodes in a project very simply, or select
based on a few common characteristics.

So, when you run `npm update mkdirp` in a large project, rather than having
to write a recursive tree walking function each time, we can just get the
collection with `tree.inventory.query('name', 'mkdirp')`.

## The `Arborist` Class

Generally, the npm CLI never interacts with `Node` or `Edge` objects
directly.  Everything that the CLI does is through the interfaces provided
by the
[`Arborist`](https://github.com/npm/arborist/tree/master/lib/arborist)
class.

The Arborist class has a constructor and a set of methods:

- [`loadActual()`](https://github.com/npm/arborist/blob/master/lib/arborist/load-actual.js)
  Load the tree of nodes representing the actual contents of the
  `node_modules` folder.
- [`loadVirtual()`](https://github.com/npm/arborist/blob/master/lib/arborist/load-virtual.js)
  Load the tree of nodes defined by the `npm-shrinkwrap.json` or
  `package-lock.json` file.
- [`buildIdealTree()`](https://github.com/npm/arborist/blob/master/lib/arborist/build-ideal-tree.js)
  Figure out what the `node_modules` folder _should_ be, either by reading
  the lockfile, or working it out from the `package.json` file.  This also
  supports arguments to add, remove, update, and so on.
- [`reify()`](https://github.com/npm/arborist/blob/master/lib/arborist/reify.js)
  Turn the ideal into the real.  While `buildIdealTree()` is strictly a
  read operation, `reify()` will write stuff to disk.
- [`audit()`](https://github.com/npm/arborist/blob/master/lib/arborist/audit.js)
  Ask the npm registry if anything is a security problem, and then work out
  what has to change in the tree to fix it.  (If called like
  `audit({fix:true})`, then it'll call out to `reify()` to fix the
  problems.)

And, coming soon:

- `dedupe()` Scan a tree for duplicated modules, and do our very best to
  squash them down as much as possible while maintaining dependency
  correctness.
- `prune()` Remove any packages from the tree that are not depended on by
  anything else.  (This can be combined with arguments to omit certain
  dependency types, to strip out all non-production deps, for example.)
- `rebuild()` Re-link bin scripts and execute installation scripts for all
  or some of the packages in the tree.

Maining a strict separation of concerns was one of the [core architectural
goals of npm
v7](https://blog.npmjs.org/post/617484925547986944/npm-v7-series-introduction),
so this division of methods, along with the borders between the data
classes and Arborist itself, allow us to have a system that is more easy to
reason about, because we don't have to hold as much in our human brains at
any given point in the process.

## Tree Building Algorithm

There are some markdown files in [Arborist's `notes`
folder](https://github.com/npm/arborist/tree/master/notes/), detailing in
even more specific detail exactly _how_ Arborist does certain things.

A few changes were made to the tree-building algorithm in v7, and to the
method used for
[reifying](https://github.com/npm/arborist/blob/master/notes/reify.md)
package trees.  The [tree building
approach](https://github.com/npm/arborist/blob/master/notes/ideal-tree.md)
builds upon the "maximally naive deduplication" approach developed by
Rebecca Turner when npm v3 introduced up-front deduplication, but adds two
new features.

In a nutshell, maximally naive deduplication starts from a given node in
the tree (typically the root node), and creates a queue of dependencies
that are currently missing or invalid.  Then, for each, it starts from the
node's `node_modules` folder, walks up the tree towards the root to find
the shallowest placement location that does not cause any conflicts.  The
newly placed node is added to the queue so _its_ dependencies can be
placed, and the process continues.

### `--prefer-dedupe`

In npm v7, a new `--prefer-dedupe` option is added to tell the tree
building algorithm to prefer deduplication over getting the latest version
of a dependency.

For example, consider this dependency graph:

```
root -> (a@1, b@1||2)
a -> (b@1)
```

By default, you might end up with a tree on disk that looks like this:

```
root
+-- a@1
|   +-- b@1
+-- b@2
```

But, this alternative tree is also correct, just more deduplicated and less
up to date:

```
root
+-- a@1
+-- b@1
```

By tracking the dependency edges throughout the tree at all times, we can
trivially make this kind of decision while building the tree.

### `peerDependencies`<a name="arborist-deep-dive-peer-dependencies"></a>

We'll get more into `peerDependencies` in a later post in this series, but
one design goal for npm v7 was to [automatically install
`peerDependencies`](https://github.com/npm/rfcs/blob/latest/accepted/0025-install-peer-deps.md),
as part of the "make npm yell at you less" and "manage your packages for
you" [design
goals](https://blog.npmjs.org/post/617484925547986944/npm-v7-series-introduction).

npm dropped support for installing `peerDependencies` in npm v4, due to the
technical challenges of doing so in a maximally naive deduplication
algorithm.  Tracking peer dependencies throughout that process requires
operating on hypothetical sub-trees rather than individual nodes, which was
somewhat challenging.

In Arborist, peer dependencies are included in the conflict detection by
doing exactly that, checking an the entire _set_ of peer dependencies at
each potential placement in the tree.  Since [peer dependencies can have
peer dependencies of their
own](https://twitter.com/izs/status/1198761415357526018), effectively we're
creating a little sub-tree and then finding a spot where it's safe to graft
it into the main one.

This means that a dependency graph like this:

```
root -> (dep, peer@1)
dep -> (transitivedep, peer@2)
transitivedep -> (PEER: peer@2)
```

will result in this correct tree:

```
root
+-- peer@1
+-- dep
    +-- peer@2
    +-- transitivedep
```

instead of the _incorrect_ tree created by npm v6:

```
root
+-- peer@1
+-- transitivedep <-- require('peer') here loads peer@1 intead of peer@2!
+-- dep
    +-- peer@2
```

### Staging Folders, Rollbacks, and Windows Folder Locking

Previous versions of npm use "staging" folders in the `node_modules` tree
in order to unpack a tarball onto disk and then move it into place in an
atomic operation.  If the install fails, the staging folders can be
removed, and the tree is left in its previous state.

However, on Windows, the file system locks files and folders for a bit of
time after immediately being modified, and rename and unlink are not
guaranteed to be atomic operations.  This has led over the years to some
Windows issues that can be difficult to reproduce, where users see their
builds fail with a mysterious `EPERM` error.  This only happens rarely, but
when 12 million people are installing billions of packages every day,
"rarely" is still quite a lot!

The [new approach to file
reification](https://github.com/npm/arborist/blob/master/notes/reify.md)
flips the staging process around.  Rather than staging a package and then
moving it into place, the _existing_ package folder is "retired" by moving
it aside.

This avoids (to the greatest extent possible) ever having to rename or
remove files and folders that were recently created.  In the case of
rollbacks on failure, of course, we may have to delete a recently-modified
directory, which can still fail in this way.  But, the surface area for
this category of failure has been dramatically reduced.

### Workspaces

All of these things add stability and performance, and it a bit easier for
the CLI team to work with the codebase.  For the most part, the user
experience impact of the Arborist refactor is that npm just works as it
always has.

One exciting exception where users will actually see a benefit of what
Arborist allows us to do is
[Workspaces](https://github.com/npm/arborist/pull/50).  In npm v6,
implementing something like Workspaces would have required significant
high-risk changes to our handling of symbolic links.

With Arborist, we've been able to focus on the "meat" of this feature.  We
added a new edge type called `workspace`, which is always resolved as a
symlink.  Then, the
[`@npmcli/map-workspaces`](http://npm.im/@npmcli/map-workspaces) module
reads the set of named workspaces so that they can be turned into these
special edges.  Lastly, we just resolve the dependency graph and reify the
resulting tree like normal.

## Current Arborist Project Status

The TODO list on Arborist has been getting shorter, and the bugs have been
getting smaller and easier to fix.  Along with the [npm v7 development
branch](https://github.com/npm/cli/tree/release/v7.0.0-beta), it's also
been put to use in some experimental community projects, which have
provided us with helpful play testing.

Preliminary testing has shown that it's faster than npm v6 in most cases,
but we've only just started the process of benchmarking it rigorously
against real world projects.  My expectation is that it'll remain somewhat
faster in most cases, and be much slower in a few cases, which will
highlight some bugs we still have to fix.

If you have a project where you are doing interesting things with
dependencies and packages, and want a more direct and programmatic API than
just running `npm` in a child process, we'd love your feedback.

[<< Introduction](https://blog.npmjs.org/post/617484925547986944/npm-v7-series-introduction)
