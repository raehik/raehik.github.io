---
title: GHC language extensions and my Haskell tendencies in 2022
date: 2022-10-06
---

[ghcup]: TODO
[stack]: TODO
[hpack]: TODO
[lexi-lambda-blog-2018-haskell-guide]: https://lexi-lambda.github.io/blog/2018/02/10/an-opinionated-guide-to-haskell-in-2018/
[GHC2021]: https://ghc.gitlab.haskell.org/ghc/doc/users_guide/exts/control.html

Back in 2018, Alexis King wrote an incredibly handy blog post titled [An
opinionated guide to Haskell in 2018][lexi-lambda-blog-2018-haskell-guide]. The
section on language extensions especially is detailed and discusses rationales
for sets of extensions. A year ago I started writing lots of Haskell, and used
her suggestions as a basis for my defaults. Since I tend to write Cursed and Bad
Code, I ended up expanding my default extensions to a recent maximum of 32.

As we come to the end of 2022, thanks to a recent [stack][] update, the
[`GHC2021`][] language set is finally convenient for all to use in combat against
bloated default extension lists. In this post, I'll comment on how and why you
should use it, other extensions you should know about, and a few "macro" style
points for Haskell projects.

# Extensions
## Background
The Haskell language is de facto defined by whatever the most popular compiler
GHC permits. However, it's also a standard, which GHC continues to respect. The
extensive research and community effort poured into GHC over the decades have
resulted in better abstractions, extended syntax and adjusted language
semantics. Many of these features are locked behind *language extensions*,
enabled either per-file or globally for a project.

There are a lot of these extensions. Some are deprecated
([`DatatypeContexts`][hs-ext-data-ctx]), others implied by common extensions
([`ExplicitForAll`][hs-ext-kindsigs]) others considered unusual style and rarely
seen ([`ImplicitParams`][hs-ext-implicitparams]). But some enable seemingly
rather basic features, like [`MultiParamTypeClasses`][hs-ext-mptc] and
[`GADTs`][hs-ext-gadts]. Perhaps you use [`DataKinds`][hs-ext-datakinds]
everywhere and are tired of typing out the `LANGUAGE` pragma. (I do, and I am.)

Default extension lists enable stating with clarity "this is the Haskell flavour
we're using". But really, most people are using the same base with a few extras
swapping in and out for taste. The GHC developers recognized this, and in GHC
9.2 introduced a "language set" [`GHC2021`][] comprising a good chunk of the
common safe extensions. My default set is down from 32 to just 10!

The user's guide page documents which extensions the flag enables. Allow me to
comment briefly on what it all means for semantics and permitted syntax.

## The `GHC2021` language set
### Using `GHC2021`
If you're using Cabal, this has been easy for a long time. In your `library` or
`test-suite` stanza or wherever, add `default-language: GHC2021`. For example:

```
library
  exposed-modules:
      ...
  ...
  default-language: GHC2021
```

Stack users can now benefit from this in a similar way. To use `GHC2021` as the
default language set for *every* part of your package (library, executable,
tests), add `language: GHC2021` to the top level of your `package.yaml`.
Example:

```yaml
name: binrep
...
language: GHC2021
```

### What `GHC2021` means for your project
#### Extends deriving mechanism
GHC can derive efficient type class instances for many type classes. `GHC2021`
enables them all. They do nothing if you don't request GHC derive such an
instance.

  * [`DeriveDataTypeable`][]
  * [`DeriveFoldable`][]
  * [`DeriveFunctor`][]
  * [`DeriveGeneric`][]
  * [`DeriveLift`][]
  * [`DeriveTraversable`][]
  * [`StandaloneDeriving`][]

## Other extensions I always use
### [`DerivingStrategies`][]
An essential instance derivation extension that somehow missed out on `GHC2021`
inclusion. GHC now has a few ways to derive instances:

  * via [`DeriveAnyClass`][], generate instance with no explicit definitions
    (useful with [`DefaultSignatures`][] to provide a "default" instance)
  * via [`DerivingVia`][], generate instance through a coercible type which
    already has an instance (to use with `newtype`s)
  * via standard mechanisms built into the compiler for certain type classes:
    `Eq`, `Ord`, `Functor` with [`DeriveFunctor`][], ...

GHC decides on the method to use. It's pretty straightforward... but it's also
rather hidden from the user. And judging from recent innovation, the deriving
mechanism may well continue to evolve. [`DerivingStrategies`][] allows the user
to specify which derivation method to use:

```haskell
data Tree a = Node a (Tree a) (Tree a) | Leaf
    deriving stock (Eq, Ord)

newtype BTree a = BTree (Tree a)
    deriving (Eq, Ord) via (Tree a)
```

Explicitness is always good.

### [`DerivingVia`][]
Mentioned above, this is another deriving extension. It generalizes
[`GeneralizedNewtypeDeriving`][] (lol), which is another way of saying you
should always use `DerivingVia` in its place. See the documentation for details.

`DerivingVia` actually implies `DerivingStrategies`, but while both remain
uncommon to see in Haskell code, I like to point them out separately.

### [`LambdaCase`][]
How'd this slip through the cracks? It adds a key piece of syntax:

```haskell
intToBool :: Int -> Bool
intToBool = \case 1 -> True
                  _ -> False
```

The only pointfree style that you can't disagree with. Keep him in your hearts,
and your `default-extensions`.

### [`NoStarIsType`][]
[ghc-proposal-0143-remove-star-kind]: https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0143-remove-star-kind.rst
[justin-kuritzkes-perfect-world]: https://www.youtube.com/watch?v=Kl3H4vMqYNo

Haskell originally used `*` to denote the kind of a type. As the GHC folks
began to muddy the line between terms and types, it's gotten painful to support
this syntax decision. Behind the scenes, the kind of a type has been `Type` for
a while... but a default extension `StarIsType` re-enables the old syntax for
compatibility. So the folks who like to multiply type-level naturals have to
fumble around with imports and language extensions.

In a perfect world, extensions like this would not exist.
But [this is not a perfect world][justin-kuritzkes-perfect-world].
Take comfort in the knowledge that some day, `NoStarIsType` will be the default.
Though, um, [one of the GHC proposals][ghc-proposal-0143-remove-star-kind]
concerning it gives a 7-year deprecation schedule. Oh.

### Larger jumps
TODO

  * [`GADTs`][]
  * [`RoleAnnotations`][]
  * [`DefaultSignatures`][]
  * [`TypeFamilies`][]
  * [`DataKinds`][]
  * [`MagicHash`][]

## Extensions to enable on-demand
### [`UndecidableInstances`][]
The idea of embracing undecidability tends to put people off, but don't be
afraid of her! This is required for much type class wizardry, in the cases where
GHC's heuristics can't guarantee instance resolution will terminate. It doesn't
impact runtime behaviour, but those restrictions keep you on the straight and
narrow, writing sensible code.

Enable only for files where compilation fails and GHC says you need
`UndecidableInstances` *and* you were expecting that to happen.

### [`AllowAmbiguousTypes`][]
This often goes hand in hand with [`TypeApplications`][]-heavy code. It
indicates a distinct function design, that certain functions can't be used
without an explicit type application, which is unusual to most Haskell
developers.

Similar to `UndecidableInstances`, enable this only when GHC tells you to.

### [`StarIsType`][]
TODO see [`NoStarIsType`][] above

### [`ApplicativeDo`][]
TODO fantastic but may contain issues, see Alexis King's blog, suggest
per-module

### [`CPP`][], [`TemplateHaskell`][]
TODO massive, scary extensions, only enable when using

## Special mentions
I don't use in my defaults but possible candidates

  * [`OverloadedStrings`][]: can force extra type annotations, prefer per-module
  * [`FunctionalDependencies`][]: less common but remains useful
  * [`PatternSynonyms`][]: amazing but huge, and without enabling you can still
    use (just not define), so sensible to be per-module

# Personal style points
## Orphan instances in their own module
Orphan instances muddy type class usage and design, but they are inevitable.
Libraries that export type classes that add behaviour for "common Haskell types"
(e.g. serialization libraries such as [binary][hackage-binary],
[cereal][hackage-cereal]) have to decide where to draw the line for piling on
dependencies -- if yours isn't included and neither library intends to depend on
the other, that's an orphan instance. (`newtype` spam can be a solution, but not
an enjoyable one.) Or as the writer of such a type class, you may like to group
a set of related instances into their own module for clarity (assuming this is
internal, and everything is re-exported elsewhere).

In both cases, I always place my orphan instances in their own module, usually
with no other exports. You can do this with an empty export list `()`
(instances get exported anyway). This [preamble snippet][gh-binrep-svec-aeson]
is handy:

```haskell
{-# OPTIONS_GHC -fno-warn-orphans #-}

module Data.Aeson.Extra.SizedVector() where
```

[gh-binrep-svec-aeson]: https://github.com/raehik/binrep/blob/64412cc94c00d79d485252c8029e2644ef707805/src/Data/Aeson/Extra/SizedVector.hs

Similarly when importing, I like to make the intent clear by importing nothing
`()` from the module, which actually imports all the instances:

```haskell
import Data.Aeson.Extra.SizedVector()
```

Why? I don't like orphan instances. They confuse me, force me to play detective
to figure out where an instance is defined. And I have to add a module-wide
pragma to ignore GHC's well-founded warnings. By placing them away from
non-orphan instances and other definitions, you reduce the risk of losing track
of them.

## `Types.hs` considered harmful
A small point on project structure. I've found early Haskell users tend to
bundle the types used throughout their project into singular `Types.hs` modules.
Naturally, this simplifies imports. But it indicates a module coupling problem
to me. I much prefer modules and module folders which are highly self-contained,
where it's hard to "accidentally" import something you don't need or is
unrelated. Consider the components of your project, the definitions they share,
and the definitions that are specific to certain areas.

## Parent export modules are good, actually
Stuffing lots of modules into a folder `Module/` and exporting only the
interesting stuff in the parent module `Module.hs` is a useful pattern.
Informally, it enables more fine-grained module splitting. Formally, you can use
it to state that all inner modules are "internal" and may change arbitrarily --
but still permit access to them. (You wouldn't stop your fellow wizard from
messing with library internals, would you?!)

## Cabal: Use the `^>=` version operator
Hackage maintains a versioning scheme along the lines of `MAJOR.MAJOR.MINOR`.
Users should expect that if they depend on version `1.2.3` of a package, all
versions `1.2.4`, `1.2.5` etc. should be compatible, up to `1.3.x`. Thus, it's
common to see in package files:

```yaml
dependencies:
- prettyprinter >= 1.7.2 && < 1.8
- ...
```

Many years ago, Cabal added an operator with this meaning!

```yaml
dependencies:
- prettyprinter ^>= 1.7.2
- ...
```

As a package maintainer, I appreciate it. Perhaps it should come with a short
comment, since I don't believe it's currently well used or understood.

# Acknowledgements & further reading
Thanks to all the Haskell folks who freely dispense their experience and
knowledge: Alexis King, the Kowainik team, everyone on the #haskell IRC
channel on Libera Chat, among others.

Some related links for further reading:

  * https://kowainik.github.io/posts/extensions
  * https://limperg.de/ghc-extensions/ GHC 8.6 (Oct 2018)
  * https://kowainik.github.io/posts/2019-02-06-style-guide
  * https://lexi-lambda.github.io/blog/2018/02/10/an-opinionated-guide-to-haskell-in-2018/
