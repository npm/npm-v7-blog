---
title: 'npm v7 Series - Beta Release! And: SemVer-Major Changes in npm v7'
date: 2020-08-10T17:00:00.000Z
---

[<< Why keep
`package-lock.json`?](https://blog.npmjs.org/post/621733939456933888/npm-v7-series-why-keep-package-lockjson)

A new beta version of npm appears!

**tl;dr** - Run `npm i -g npm@next-7` _right now_, and [tell us about any
problems you encounter with it](https://github.com/npm/cli/issues).

This is a big one, you're going to want to check it out.  As with any beta
software, it's likely to still have a few rough edges, but we're confident
that those will be polished down very quickly.  Your help finding them will
make that process go much faster.

This post outlines the major-version updates (aka "Breaking Changes" in
[SemVer](https://semver.org) lingo) in this release.  If you're going to
start using npm v7, it would be good to take a look over this list and make
sure nothing is going to ruin your day.

Despite a massive overhaul of most of npm's functionality, we are
endeavoring to make npm v7 as low-risk and low-effort as possible, for as
many users of npm as we possibly can.  Nevertheless, this list is fairly
long, because we endeavor to call out a change even if we think the impact
is likely minor.

For the massive majority of npm users, where your experience is "install my
stuff, do the right thing, let me get back to work", we are confident this
update will be entirely an improvement to your development experience.

## On the "Breaking" in "Breaking Changes"

The Semantic Versioning specification precisely defines what constitutes a
"breaking" change.  In a nutshell, it's any change that causes a you to
change _your_ code in order to start using _our_ code.  We hasten to point
this out, because a "breaking change" does not mean that something about
the update is "broken", necessarily.

We're sure that some things likely _are_ broken in this beta, because it's
best to be pessimistic about beta software.  But nothing is "broken" on
purpose here, and if you find a bug, we'd love for you to [let us
know](https://github.com/npm/cli/issues) so we can get it fixed before the
beta tag falls off.

## Peer Dependencies

We have identified automatic `peerDependencies` installation as a
potentially disruptive change for many users (albeit one that we are
confident is the correct behavior for a package manager), we have some
tools to minimize this disruption, based on the feedback we get.

We are confident that resolving package trees such that `peerDependencies`
are properly accounted for is the right thing to do.  After all, an error
here can result in a production issue that's very difficult to debug later,
especially if it occurs deep in a `node_modules` tree.  However, years of
_not_ resolving `peerDependencies` has allowed many projects to fail to
notice these problems.

In order to get unblocked and install your project in spite of
`peerDependencies` conflicts, you can use the `--legacy-peer-deps` flag at
install time.  It may be that the disruption is too great to take all at
once, and we have to have this flag enabled by default for a while as
projects gradually update their conflicting dependencies.  Our intent is to
let the beta give us some more data points to help make that decision
carefully.

Regardless of where we land on installs, the `npm ls` command will always
run with `--legacy-peer-deps` set to `false`, so that missing peer
dependencies will be highlighted when looking at the package tree.  We hope
that this will be useful encouragement to get projects to update out of a
broken state (or worse, a "works by accident" state).

## Beta Schedule

Beta versions of `v7` will go out weekly on Tuesdays until everything looks
good.  We'll decide to cut a GA release based on stability and getting the
last bits of housecleaning done.

## What's Missing From This Beta (Why Not GA?)

We have not yet gotten to 100% test coverage of the npm CLI codebase.  As
such, there are almost certainly bugs lying in wait.  We _do_ have 100%
test coverage of most of the commands, and all recently-updated
dependencies in the npm stack, so it's certainly more well-tested than any
version of npm before.

There's still lots of work we need to do in updating the documentation.
Prior to a GA release, we'll be going through all of our documentation with
a fine-toothed comb to make sure it's as accurate and helpful as possible.

That's about it!  It's ready to use for most purposes, and you should try
it out.

Now on to the list of changes.

-----

## Programmatic Usage

- [RFC
  20](https://github.com/npm/rfcs/blob/latest/accepted/0020-npm-option-handling.md)
  The CLI and its dependencies no longer use the `figgy-pudding` library
  for configs.  Configuration is done using a flat plain old JavaScript
  object.
- The `lib/fetch-package-metadata.js` module is removed.  Use
  [`pacote`](http://npm.im/pacote) to fetch package metadata.
- [`@npmcli/arborist`](http://npm.im/@npmcli/arborist) should be used to do
  most things programmatically involving dependency trees.
- The `onload-script` option is no longer supported.
- The `log-stream` option is no longer supported.
- `npm.load()` MUST be called with two arguments (the parsed cli options
  and a callback).
- `npm.root` alias for `npm.dir` removed.
- The `package.json` in npm now defines an `exports` field, making it no
  longer possible to `require()` npm's internal modules.  (This was always
  a bad idea, but now it won't work.)

## All Registry Interactions

The following affect all commands that contact the npm registry.

- `referer` header no longer sent
- `npm-command` header added

## All Lifecycle Scripts

The environment for lifecycle scripts (eg, build scripts, `npm test`, etc.)
has changed.

- [RFC
  21](https://github.com/npm/rfcs/blob/latest/accepted/0021-reduce-lifecycle-script-environment.md)
  Environment no longer includes `npm_package_*` fields, or `npm_config_*`
  fields for default configs.  `npm_package_json`, `npm_package_integrity`,
  `npm_package_resolved`, and `npm_command` environment variables added.

    (NB: this [will change a bit prior to a `v7.0.0` GA
    release](https://github.com/npm/rfcs/pull/183))

- [RFC
  22](https://github.com/npm/rfcs/blob/latest/accepted/0022-quieter-install-scripts.md)
  Scripts run during the normal course of installation are silenced unless
  they exit in error (ie, with a signal or non-zero exit status code), and
  are for a non-optional dependency.

- [RFC
  24](https://github.com/npm/rfcs/blob/latest/accepted/0024-npm-run-traverse-directory-tree.md)
  `PATH` environment variable includes all `node_modules/.bin` folders,
  even if found outside of an existing `node_modules` folder hierarchy.

- The `user`, `group`, `uid`, `gid`, and `unsafe-perms` configurations are
  no longer relevant.  When npm is run as root, scripts are always run with
  the effective `uid` and `gid` of the working directory owner.

- Commands that just run a single script (`npm test`, `npm start`, `npm
  stop`, and `npm restart`) will now run their script even if
  `--ignore-scripts` is set.  Prior to the GA v7.0.0 release, [they will
  _not_ run the pre/post scripts](https://github.com/npm/rfcs/pull/185),
  however.  (So, it'll be possible to run `npm test --ignore-scripts` to
  run your test but not your linter, for example.)

## npx

The `npx` binary was rewritten in npm v7, and the standalone `npx` package
deprecated when v7.0.0 hits GA.  `npx` uses the new `npm exec` command
instead of a separate argument parser and install process, with some
affordances to maintain backwards compatibility with the arguments it
accepted in previous versions.

This resulted in some shifts in its functionality:

- Any `npm` config value may be provided.
- To prevent security and user-experience problems from mistyping package
  names, `npx` prompts before installing anything.  Suppress this prompt
  with the `-y` or `--yes` option.
- The `--no-install` option is deprecated, and will be converted to `--no`.
- Shell fallback functionality is removed, as it is not advisable.
- The `-p` argument is a shorthand for `--parseable` in npm, but shorthand
  for `--package` in npx.  This is maintained, but only for the `npx`
  executable.  (Ie, running `npm exec -p foo` will be different from
  running `npx -p foo`.)
- The `--ignore-existing` option is removed.  Locally installed bins are
  always present in the executed process `PATH`.
- The `--npm` option is removed.  `npx` will always use the `npm` it ships
  with.
- The `--node-arg` and `-n` options are removed.
- The `--always-spawn` option is redundant, and thus removed.
- The `--shell` option is replaced with `--script-shell`, but maintained in
  the `npx` executable for backwards compatibility.

We do intend to continue supporting the `npx` that npm ships; just not the
`npm install -g npx` library that is out in the wild today.

## Files On Disk

- [RFC
  13](https://github.com/npm/rfcs/blob/latest/accepted/0013-no-package-json-_fields.md)
  Installed `package.json` files no longer are mutated to include extra
  metadata.  (This extra metadata is stored in the lockfile.)
- `package-lock.json` is updated to a newer format, using
  `"lockfileVersion": 2`.  This format is backwards-compatible with npm CLI
  versions using `"lockfileVersion": 1`, but older npm clients will print a
  warning about the version mismatch.
- `yarn.lock` files used as source of package metadata and resolution
  guidance, if available.  (Prior to v7, they were ignored.)

## Dependency Resolution

These changes affect `install`, `ci`, `install-test`, `install-ci-test`,
`update`, `prune`, `dedupe`, `uninstall`, `link`, and `audit fix`.

- [RFC
  25](https://github.com/npm/rfcs/blob/latest/accepted/0025-install-peer-deps.md)
  `peerDependencies` are installed by default.  This behavior can be
  disabled by setting the `legacy-peer-deps` configuration flag.

    **BREAKING CHANGE**: this can cause some packages to not be
    installable, if they have unresolveable peer dependency conflicts.
    While the correct solution is arguably to fix the conflict, this was
    not forced upon users for several years, and some have come to rely on
    this lack of correctness.  Use the `--legacy-peer-deps` config flag if
    impacted.

    See the note above about `--legacy-peer-deps`.

- [RFC
  23](https://github.com/npm/rfcs/blob/latest/accepted/0023-acceptDependencies.md)
  Support for `acceptDependencies` is added.  This can result in dependency
  resolutions that previous versions of npm will incorrectly flag as
  invalid.

- Git dependencies on known git hosts (GitHub, BitBucket, etc.) will always
  attempt to fetch package contents from the relevant tarball CDNs if
  possible, falling back to `git+ssh` for private packages.  `resolved`
  value in `package-lock.json` will always reflect the `git+ssh` url value.
  Saved value in `package.json` dependencies will always reflect the
  canonical shorthand value.

- Support for the `--link` flag (to install a link to a globall-installed
  copy of a module if present, otherwise install locally) has been removed.
  Local installs are always local, and `npm link <pkg>` must be used
  explicitly if desired.

- Installing a dependency with the same name as the root project no longer
  requires `--force`.  (That is, the `ENOSELF` error is removed.)

## Workspaces

- [RFC
  26](https://github.com/npm/rfcs/blob/latest/accepted/0026-workspaces.md)
  First phase of `workspaces` support is added.  This changes npm's
  behavior when a root project's `package.json` file contains a
  `workspaces` field.

## `npm update`

- [RFC
  19](https://github.com/npm/rfcs/blob/latest/accepted/0019-remove-update-depth-option.md)
  Update all dependencies when `npm update` is run without any arguments.
  As it is no longer relevant, `--depth` config flag removed from `npm
  update`.

## `npm outdated`

- [RFC
  27](https://github.com/npm/rfcs/blob/latest/accepted/0027-remove-depth-outdated.md)
  Remove `--depth` config from `npm outdated`.  Only top-level dependencies
  are shown, unless `--all` config option is set.

## `npm adduser`, `npm login`

- The `--sso` options are deprecated, and will print a warning.

## `npm audit`

- Output and data structure is significantly refactored to call attention
  to issues, identify classes of fixes not previously available, and remove
  extraneous data not used for any purpose.

   **BREAKING CHANGE**: Any tools consuming the output of `npm audit` will
   almost certainly need to be updated, as this has changed significantly,
   both in the readable and `--json` output styles.

## `npm dedupe`

- Performs a full dependency tree reification to disk.  As a result, `npm
  dedupe` can cause missing or invalid packages to be installed or updated,
  though it will only do this if required by the stated dependency
  semantics.

- Note that the `--prefer-dedupe` flag has been added, so that you may
  install in a maximally deduplicated state from the outset.

## `npm fund`

- Human readable output updated, reinstating depth level to the printed
  output.

## `npm ls`

- Extraneous dependencies are listed based on their location in the
  `node_modules` tree.
- `npm ls` only prints the first level of dependencies by default.  You can
  make it print more of the tree by using `--depth=<n>` to set a specific
  depth, or `--all` to print all of them.

## `npm pack`, `npm publish`

- Generated gzipped tarballs no longer contain the zlib OS indicator.  As a
  result, they are truly dependent _only_ on the contents of the package,
  and fully reproducible.  However, anyone relying on this byte to identify
  the operating system of a package's creation may no longer rely on it.

## `npm rebuild`

- Runs package installation scripts as well as re-creating links to bins.
  Properly respects the `--ignore-scripts` and `--bin-links=false`
  configuration options.

## `npm build`, `npm unbuild`

- These two internal commands were removed, as they are no longer needed.

## `npm test`

- When no test is specified, will fail with `missing script: test` rather
  than injecting a synthetic `echo 'Error: no test specified'` test script
  into the `package.json` data.

## Credits

Huge thanks to the people who wrote code for this update, as well as our
group of dedicated Open RFC call participants.  Your participation has
contributed immeasurably to the quality and design of npm.

[<< Why keep
`package-lock.json`?](https://blog.npmjs.org/post/621733939456933888/npm-v7-series-why-keep-package-lockjson)
