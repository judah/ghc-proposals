---
author: Judah Jacobson
date-accepted: ""
ticket-url: ""
implemented: ""
---

This proposal is [discussed at this pull request](https://github.com/ghc-proposals/ghc-proposals/pull/0>).
**After creating the pull request, edit this file again, update the number in
the link, and delete this bold sentence.**

# Permissive Package Names

This proposal expands the notion of GHC package names to more kinds of strings
(for example, containing slashes).  It aims to improve GHC's integration with other
build systems such as [Bazel](https://bazel.build). This proposal would not
change the user-facing behavior of other existing build systems such as Cabal.
It would also not affect the behavior of GHC's internal package ids/keys.


## Motivation

Cabal (and, by extension, Hackage) organizes the open-source Haskell ecosystem
using a fairly restrictive namespace. Names can only contain alphanumeric
characters and dashes (e.g., `foo-bar`). Furthermore, each dash-separated
"component" must not be all digits (e.g.: `foo-234` or `foo-12-bar`).  The latter restriction
helps prevent ambiguity when referring to package versions as `{name}-{version}`.
GHC follows these restrictions, for example when parsing the argument of the
`-package` flag or when registering packages with `ghc-pkg`.

This approach can be overly restrictive for some build systems.  In
particular, [Bazel] is an open-source, language-agnostic build system with an
implementation for [Haskell rules].  Users write `BUILD` files in various
subdirectories, each of which define individual targets (e.g.,
`haskell_library`). Targets are indentified with a [label] syntax, for example
`//foo/bar:baz` would be defined in `foo/bar/BUILD` and usually concern source files also contained
in `foo/bar/...`. The Haskell rules control visibility of modules by registering
each library target as a separate GHC package.

[Bazel]: https://bazel.build
[Haskell rules]: https://haskell.build
[label]: https://docs.bazel.build/versions/master/build-ref.html#labels

Ideally, Haskell package names in Bazel would correspond directly to the directory
structure. However, GHC either restricts or doesn't allow some characters that
are valid labels (`'/"`, `':'`, `_` and '-'). To work around those limitation, the
haskell rules Z-encode labels into valid package names.  For example, `"//foo/bar-baz"`
is represented as `ZSZSfooZSbarZUbaz`.  This works, but users may encounter the 
encoded package names in error messagges, or need to specify them manually
with `:set -package`, or (less commonly) via the `PackageImports` extension.

We have also considered using `PackageImports` with Bazel more generally, to
represent more directory structure in Haskell modules.  For example, we could import
the file `foo/bar/A/B.hs` with

```haskell
import "/foo/bar" A.B
```

That way the import statement references both the Haskell module structure (`A.B`) and the
lower-cased directory structure ("/foo/bar").  That could potentially improve
readability, and simplify some tooling by making it easier to tell immediately what file
a given module was defined in.

## Proposed Change Specification

GHC, following Cabal, currently defines a valid package name as one or more
dash-separated components, where each component is not all digits.  In
`compiler/utils/Util.hs`, this is defined as:

```haskell
looksLikePackageName :: String -> Bool
looksLikePackageName = all (all isAlphaNum <&&> not . (all isDigit)) . split '-'
```

In BNF, that could be represented as

```
<package_name> ::= <component> | <component> "-" <package_name>
<component> ::= <alphaNums> |  <alpha> <alphaNums>
<alphaNums> = "" | <alphaNum> <alphaNums>
```

We propose expanding the set of valid package names to also include *any* string
of non-space, printing characters, as long as it starts with a "/".  By "visible" we mean In BNF:

```
<new_package_name> = <permissive_name> |  <package_name>
<permissive_name> = "/" <visibleChars>
<visibleChars> = "" | <visible> <visibleChars>
```

Which in Haskell would be represented as:

```haskell
looksLikePermissivePackageName :: String -> Bool
looksLikePermissivePackageName ('/' : ss) = all (isPrint <&&> not isSpace) ss
looksLikePermissivePackagegName ss = looksLikePackageName ss
```

## Examples

This section illustrates the specification through the use of examples of the
language change proposed. It is best to exemplify each point made in the
specification, though perhaps one example can cover several points. Contrived
examples are OK here. If the Motivation section describes something that is
hard to do without this proposal, this is a good place to show how easy that
thing is to do with the proposal.

## Effect and Interactions

GHC would accept the more permissive package names in several places:

- `ghc-pkg`, in the `name:` field.  For example: `name: /foo/bar`.
- `ghc_pkg`, in the `exposed_modules:` field, for reexported modules. For example:
  `exposed_modules: FooBarBaz from /foo/bar:Baz`
- The `-package` flag.  For example: `-package=/foo/bar`.  (See below for details).
- The `PackageImports` extension.  GHC's parser should allow more permissive
  package names in that syntax.  For example: `import "/foo/bar" Baz`.

For the `PackageImports` extension, we should guard the new behavior
behind a language  extension (for example: `-XPermissivePackageNames`).

The `-package` flag currently parses both a package *name* and an optional version.
For example, `-package=foo-1.2` becomes ("foo", "1.2")`.  With the new names,
the version number won't be picked up:  `-package=/foo/bar-1.2` would get
parsed as `(package, version) = ("/foo/bar-1.2", "")`.  This restriction should
be fine in practice, since multiple versions of the same package can just go in
different directories: `"/foo/bar-1.2", "/foo/bar/1.2", etc.  However, we should
probably recommend that packages with permissive names should have an empty
`ghc-pkg` `version:` field.

The `-package` flag may also be used for [thinning or renaming].  For example:
`-package "base (Data.Bool as Bool)"`  This syntax will continue to work, since spaces
are still forbidden in package names.
```

[thinning or renaming]: https://downloads.haskell.org/ghc/latest/docs/html/users_guide/packages.html#thinning-and-renaming-modules

That syntax should not be affected by the current scheme, since package
names must still consist of visible, non-print characters.

GHC defines CPP macros for each package dependency: `VERSION_pkgname` and
`MIN_VERSION_pkgname`.  Those macros will not generally be valid CPP for
permissive names (e.g., containing slashes).  For simplicity,
we should just omit the macros for names that start with a slash.

## Costs and Drawbacks

As mentioned above, this proposal does not intend to change the user-facing behavior of
Cabal or Hackage.  However, `ghc-pkg's implementation currently uses Cabal (as a library)
and its type `Distribution.InstalledPackageInfo.InstalledPackageInfo`, to convert between the
user-visible ".conf" format and GHC's internal package format.  This contrasts the rest of
GHC, which only uses the `ghc-boot` library's `GHC.PackageDb.InstalledPackageInfo`.

We have a couple options:

1. Make `ghc-pkg` itself be more permissive about package paths, by switching it to use
`GHC.PackageDb.InstalledPackageInfo` as well.  That may also involve moving the logic around
`.conf` files from Cabal-the-library into GHC itself (possibly: `ghc-boot`).
2. Make `Cabal`-the-library as permissive as GHC itself, but also add more checks in
`cabal`-the-tool and Hackage server to forbid more permissive package names.  That approach
could end up somewhat fragile, though.

## Alternatives

As mentioned above, the current approach in Bazel rules is to Z-encode labels into
package names.  However, package names are supposed to be user-visible, so this
is not a great user experience.

Bazel projects can also adopt some other conventions to reflect more of the filepaths
in their module names, none of which are ideal.  Suppose we are in a subdirectory
`some/project`, and we want to create a module `Foo.Bar`  One way is be to
duplicate the lower-case directory in the module name:

```
some/project/Some/Project/Foo/Bar.hs contains Some.Project.Foo.Bar
```

Another way is to CamelCase the lower-cased prefix:

```
some/project/Foo/Bar.hs contains Some.Project.Foo.Bar
```

But that approach breaks some assumptions about how hierarchical module names work.
It turns out that GHC is fine with this, as long as you pass it each source file explicitly.
However, some other tooling will fall down without extra help (for example, `hspec-discover`,
which needs to convert filepaths to module names).

We could also try to use a more user-friendly conversion when creating the package name.
For example, convert `"/"` to `"-"` and camel-case every directory:

```
some/project/Foo/Bar.hs` contains Foo.Bar with the package name "some-project"
```

However, in practice we've found that convention to be less ideal since directory components
may themselves contain dashes (e.g., `my-other/project` versus `m

## Unresolved Questions

Is the list of valid characters too permissive?  Are there other places where it would
cause problems?

Is the "leading slash" approach reasonable?  additionally, is it good enough for Bazel
or other tools?  For example, Bazel also has a notion of 'external repositories' where label
names may start with "@".

How should this proposal affect `Cabal`-the-library?

