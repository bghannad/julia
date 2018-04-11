# Code Loading

Julia has two mechanisms for loading code:

1. **Code inclusion:** e.g. `include("source.jl")`. Inclusion allows you to split a single program across multiple source files. The expression `include("source.jl")` causes the contents of the file `source.jl` to be evaluated inside of the module where the `include` call occurs, much as if the text of `source.jl` were pasted into that file in place of the `include` call. If `include("source.jl")` is called multiple times, `source.jl` is evaluated multiple times. The included path, `source.jl`, is interpreted relative to the file where the `include` call occurs. This makes it simple to relocate a subtree of source files. In the REPL, included paths are interpreted relative to the current working directory, `pwd()`.
2. **Package loading:** e.g. `import X` or `using X`. The import mechanism allows you to load a package—i.e. an independent, reusable collection of Julia code, wrapped in a module—and makes the resulting module available by the name `X` inside of the importing module. If the same `X` package is imported multiple times in the same Julia session, it is only loaded the first time—on subsequent imports, the importing module gets a reference to the same module. It should be noted, however, that `import X` can load different packages in different contexts: `X` can refer to one package named `X` in the main project but potentially different packages named `X` in each dependency. More on this below.

Code inclusion is quite straightforward: it simply parses and evaluates a source file in the context of the caller. Package loading is built on top of code inclusion and is quite a bit more complex. The rest of this chapter, therefore, focuses on the behavior and mechanics of package loading.

## Package Loading

A *package* is a source tree with a standard layout providing functionality that can be reused by other Julia projects. A package is loaded by `import X` or  `using X` statements. These statements also make the module named `X`, which results from loading the package code, available within the module where the import statement occurs. The meaning of `import X` is context-dependent: its meaning and behavior depend on what code it occurs in. What it does depends on the answers to two questions:

1. **What** package is `X` in this context?
2. **Where** can that `X` package be found?

Understanding how Julia answers these questions is key to understanding package loading.

### Federation of packages

Julia supports federated management of packages. This means that multiple independent parties can maintain both public and private packages and registries of them, and that projects can depend on a mix of public and private packages from different registries. Packages from various registries are installed and managed using a common set of tools and workflows. The Pkg3 next-generation package manager [[docs](https://julialang.org/Pkg3.jl/latest/), [repo](https://github.com/JuliaLang/Pkg3.jl)] ships with Julia 0.7/1.0 and lets you install and manage dependencies of your projects, by creating and manipulating project files, which describe what your project depends on, and manifest files that snapshot exact versions of your project's complete dependency graph.

One consequence of federation is that there cannot be a central authority for package naming. Different entities may use the same name to refer to unrelated packages. This possibility is unavoidable since these entities do not coordinate and may not even know about each other. Beacuse of the lack of a central naming authority, a single project can quite possibly end up depending on different packages with the same name. Julia's package loading mechanism handles this by not requiring package names to be globally unique, even within the dependency graph of a single project. Instead, packages are identified by [universally unique identifiers](https://en.wikipedia.org/wiki/Universally_unique_identifier) (UUIDs) which are assigned to them before they are registered. The question *"what is `X`?"* is answered by determining the UUID of `X`.

Since the decentralized naming problem is somewhat abstract, it may help to walk through a concrete scenario to understand the issue. Suppose you're developing an application called `App`, which uses two packages: `Pub` and  `Priv`. `Priv` is a private package that you created, whereas `Pub` is a public package that you use but don't control. When you created `Priv`, there was no public package by that name. Subsequently, however, an unrelated package also named `Priv` has been published and become popular. In fact, the `Pub` package has started to use it. Therefore, when you next upgrade `Pub` to get the latest bug fixes and features, `App` will end up—through no action of yours other than upgrading—depending on two different packages named `Priv`. `App` has a direct dependency on your private `Priv` package, and an indirect dependency, through `Pub`, on the new public `Priv` package. Since these two `Priv` packages are different but both required for `App` to continue working correctly, the expression `import Priv` must refer to different `Priv` packages depending on whether it occurs in `App`'s code or in `Pub`'s code. Julia's package loading mechanism allows this by distinguishing the two `Priv` packages by context and UUID. How this distinction works is determined by environments, as explained in the following sections.

### Environments

An *environment* determines what `import X` and `using X` mean in various code contexts and what files these statements cause to be loaded. Julia understands three kinds of environments:

1. **A project environment** is a directory with a project file and an optional manifest file. The project file determines what the names and identities of the direct dependencies of a project are. The manifest file, if present, gives a complete dependency graph, including all direct and indirect dependencies, exact versions of each dependency, and sufficient information to locate and load the correct version.
2. **A package directory** is a directory containing the source trees of a set of packages. This kind of environment was the only kind that existed in Julia 0.6 and earlier. If `X` is a subdirectory of a package directory and `X/src/X.jl` exists, then the package `X` is  available in the package directory environment and `X/src/X.jl` is the source file by which it is loaded.
3. **A stacked environment** is an ordered set of project environments and package directories, overlaid to make a single composite environment in which all packages available in the constituent environments are available. Julia's load path, for example, is a stacked environment.

As an abstraction, an environment provides three maps: `roots`, `graph` and `paths`. When resolving the meaning of `import X`, `roots` and `graph` are used to determine the identity of `X` and thereby answer the question *"what is `X`?"*, while the `paths` map is used to locate the source of that package `X` and thereby anser the question *"where can `X` be found?" The specific roles of these three maps are:

- **roots:** `name::Symbol` ⟶ `uuid::UUID`

   An environment's `roots` map assigns package names to UUIDs for all the top-level dependencies that the environment makes availabe to the main project (i.e. the ones that can be loaded in `Main`). When Julia encounters `import X` in the main project, it looks up the identity of `X` as `roots[:X]`.

- **graph:** `context::UUID` ⟶ `name::Symbol` ⟶ `uuid::UUID`

   An environment's `graph` is a multilevel map which assigns, for each `context` UUID, a map from names to UUIDs, similar to the `roots` map but specific to that `context`. When Julia sees `import X` in the code of the package whose UUID is `context`, it looks up the identity of `X` as `graph[context][:X]`. In particular, this means that `import X` can refer to different packages depending on `context`.

- **paths:** `uuid::UUID` × `name::Symbol` ⟶ `path::String`

   The `paths` map assigns a UUID and a name to the location of the entry-point source file of that package. After the identity of `X` in `import X` has been resolved to a UUID via `roots` or `graph` (depending on whether it is loaded from the main project or an dependency), Julia determines what file to load to acquire `X` by looking up `paths[uuid, :X]` in the environment. Including this file should create a module named `X`. After the first time this package is loaded, any import resolving to the same `uuid` will simply create a new binding to the same already-loaded package module.

Each kind of environment defines these three maps differently, as detailed in the following sections.

#### Project environments

A project environment is determined by a directory containing a project file, `Project.toml`, and optionally a manifest, `Manifest.toml`. These files can also be named `JuliaProject.toml` and `JuliaManifest.toml`, in which case `Project.toml` and `Manifest.toml` are ignored, which allows coexistence with other tools that might consider files named `Project.toml` and `Manifest.toml` significant. For pure Julia projects, however, the names `Project.toml` and `Manifest.toml` should be preferred. The `roots`, `graph` and `paths` maps of a project environment are defined as follows.

**The roots map** of the environment is determined by the contents of the project file, specifically, its top-level `name` and `uuid` entries and its `[deps]` section (any of these may be absent). Consider the following example project file for the hypothetical application, `App`, as described above:

```toml
name = "App"
uuid = "8f986787-14fe-4607-ba5d-fbff2944afa9"

[deps]
Priv = "ba13f791-ae1d-465a-978b-69c3ad90f72b"
Pub  = "c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1"
```

This project file implies the following `roots` map as an explicit Julia dictionary:

```julia
roots = Dict(
    :App  => UUID("8f986787-14fe-4607-ba5d-fbff2944afa9"),
    :Priv => UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b"),
    :Pub  => UUID("c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1"),
)
```

Note that for efficiency reasons, `roots` maps are not materialized as dictionaries, but are instead queried through internal APIs. Given this `roots` map, in the code for `App` itself, `import Priv` causes Julia to look up `roots[:Priv]`, which yields `ba13f791-ae1d-465a-978b-69c3ad90f72b`, the UUID of the `Priv` package that is to be loaded in that context.

**The depedency graph** of a project environment is determined by the contents of the manifest file, if present (if there is no manifest file, `graph` is empty). A manifest file contains a stanza for each concrete direct or indirect dependency of a project, giving the UUID and exact version information for each dependency and optionally an explicit path to it. Take the following manifest file for our `App` example:

```toml
[[Priv]] # the private one
deps = ["Pub", "Zebra"]
uuid = "ba13f791-ae1d-465a-978b-69c3ad90f72b"
path = "deps/Priv"

[[Priv]] # the public one
uuid = "2d15fe94-a1f7-436c-a4d8-07a9a496e01c"
git-tree-sha1 = "1bf63d3be994fe83456a03b874b409cfd59a6373"
version = "0.1.5"

[[Pub]]
uuid = "ba13f791-ae1d-465a-978b-69c3ad90f72b"
git-tree-sha1 = "9ebd50e2b0dd1e110e842df3b433cb5869b0dd38"
version = "2.1.4"

  [Pub.deps]
  Priv = "2d15fe94-a1f7-436c-a4d8-07a9a496e01c"
  Zebra = "f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"

[[Zebra]]
uuid = "f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"
git-tree-sha1 = "e808e36a5d7173974b90a15a353b564f3494092f"
version = "3.4.2"
```

This manifest file describes a complete dependency graph for the `App` project. Specifically:

- There are two different `Priv` packages that the application needs—a private one which is a direct dependency, and a public one which is an indirect dependency through `Pub`.
- * The private `Priv` depends on the `Pub` and `Zebra` packages.
  * The public `Priv` has no depdendencies.
- The application also depends on the `Pub` package, which in turn depends on the public `Priv ` and the same `Zebra` package that the private `Priv` package depends on.

A materialized repreentation of this dependency `graph` looks like this:

```julia
graph = Dict{UUID,Dict{Symbol,UUID}}(
    # Priv – the private one:
    UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b") => Dict{Symbol,UUID}(
        :Zebra => UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"),
    ),
    # Priv – the public one:
    UUID("2d15fe94-a1f7-436c-a4d8-07a9a496e01c") => Dict{Symbol,UUID}(),
    # Pub:
    UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b") => Dict{Symbol,UUID}(
        :Priv  => UUID("2d15fe94-a1f7-436c-a4d8-07a9a496e01c"),
        :Zebra => UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"),
    ),
    # Zebra:
    UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62") => Dict{Symbol,UUID}(),
)
```

Again, for efficicency reasons, Julia doesn't necessary materialize this graph during package loading, instead it computes parts of it as needed. Given this dependency `graph`, when Julia sees `import Priv` in the `Pub` package—which has UUID `ba13f791-ae1d-465a-978b-69c3ad90f72b`—it looks up

```julia
graph[UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b")][:Priv]
```

and gets `2d15fe94-a1f7-436c-a4d8-07a9a496e01c` , which indicates that in the context of the `Pub` package,  `import Priv` refers to the public `Priv` package, not the private one as the app depends on directly. Thus, the name `Priv` can refer to different packages in the main project than it does in one of the packages dependencies, smoothly allowing for name collisions in a federated packag ecosystem.

What happens if `import Zebra` is evaluated in the main `App` project? Since `Zebra` does not appear in the project file, the import will fail even though `Zebra` *does* appear in the manifest file. Moreover, if `import Zebra` occurred in the public `Priv` package (the one with UUID `2d15fe94-a1f7-436c-a4d8-07a9a496e01c`), that would also fail since there is no `deps` entry in that package's manifest stanza, so `Priv` cannot load any packages. The `Zebra` package can only be loaded by packages for which it appear as an explicit dependency in the manifest file: the  `Pub` package and the private `Priv` package.

**The paths map** of a project environment is also determined by the manifest file if present (if absent, `paths` is an empty map). The path for a package called `X` is determined by these two rules:

1. If the manifest stanza for a package has a `path` entry, then that path, relative to the manifest file, is returned. If the location is a directory, return `src/X.jl`, otherwise return the path itself.
2. If the manifest stanza has `uuid` and `git-tree-sha1` entries, compute a deterministic hash function of those two values, call it `slug`, and look for the directory `packages/X/slug` in each directory in the Julia variable `DEPOT_PATH`. Return the first such directory which exists.

In the example manifest file above, to find the path of the first `Priv` packag (the one with UUID `ba13f791-ae1d-465a-978b-69c3ad90f72b`), Julia finds its stanza in the manifest file, sees that it has a `path` entry, looks at `deps/Priv` relative to the `App` project directory—say it's `/home/me/projects/App`—sees that `/home/me/projects/App/deps/Priv` is a directory, and therefore determines the path of `Priv` to be:

* `/home/me/projects/App/deps/Priv/src/Priv.jl`

If Julia was loading the other `Priv` package (the one with UUID `2d15fe94-a1f7-436c-a4d8-07a9a496e01c`), it would find its stanza by UUID, see that it does *not* have a `path` entry, but that it does have a `git-tree-sha1` entry. It then computes the `slug` for this UUID/SHA-1 pair, which is `HDkr` (the exact details of this computation aren't important, but it is consistent and predictable across Julia versions). This means that the path to this `Priv` package will be `packages/Priv/HDkr/src/Priv.jl` in one of the package depots. So if the contents of `DEPOT_PATH` is `["/users/me/.julia", "/usr/local/julia"]` then Julia will look for the following paths to see if any of them exist:

1. `/home/me/.julia/packages/Priv/HDkr/src/Priv.jl`
2. `/usr/local/julia/packages/Priv/HDkr/src/Priv.jl`

It uses the first of these which exists as the path of `Priv`. For our `App` example, the `paths` map for the entire `App` project environment could materialized as the following dictionary:

```julia
paths = Dict{Tuple{UUID,Symbol},String}(
    # Priv – the private one:
    (UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b"), :Priv) =>
        # relative entry-point inside `App` repo:
        "/home/me/projects/App/deps/Priv/src/Priv.jl",
    # Priv – the public one:
    (UUID("2d15fe94-a1f7-436c-a4d8-07a9a496e01c"), :Priv) =>
        # package installed in the user depot:
        "/home/me/.julia/packages/Priv/HDkr/src/Priv.jl",
    # Pub:
    (UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b"), :Pub) =>
        # package installed in the system depot:
        "/usr/local/julia/packages/Pub/oKpw/src/Pub.jl",
    # Zebra:
    (UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"), :Zebra) =>
        # package installed in the user depot:
        "/home/me/.julia/packages/Zebra/me9k/src/Zebra.jl",
)
```

#### Package directories

Package directories provide a kind of environment that approximates package loading in Julia 0.6 and earlier, and which resembles package loading in many other dynamic languages. The set of top-level packages made available by a package directory corresponds to the set of subdirectories it contains that look like packages: if `X` is a subdirectory and `X/src/X.jl` is a file, then `X` is considered to be a package and `X/src/X.jl` is the file you load to get `X`. Which packages can "see" each other via `graph` depends on if they contain project files and what is in the `[deps]` section of those project files.

**The roots map** is determined by the subdirectories of a package directory for which `X/src/X.jl` exists and the presence and, if present, the top-level `uuid` entry of the `X/Project.toml` file:

1. If `X/Project.toml` exists and has a top-level UUID entry, `uuid`, then `:X => uuid` goes in `roots`.
2. If `X/Project.toml` exists and does not have a top-level UUID entry, then `:X => dummy` goes in roots, where `dummy` is a fake UUID based on hashing the canonicalized absolute path of `X/Project.toml`.
3. If `X/Project.toml` does not exist, then `:X => nil` goes in `roots`, where `nil` is the [nil UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier#Nil_UUID), `00000000-0000-0000-0000-000000000000`.

**The dependency graph** of a project directory is determined by the presence and contents of project files in the subdirectory of each package. There are two rules:

- If a package subdirectory has no project file, then it does not appear in `graph` at all and import statements in its code are treated as top-level, just as if they occurred in the main project or REPL.
- If a package subdirectory has a project file with a real or dummy UUID (as described above), then the `graph` entry for that UUID is the `[deps]` map of that project file, empty if there is no section.

To better understand both the `roots` and `graph` rules for project directories, suppose a package directory had the following structure and content:

```
Aardvark/
    src/Aardvark.jl
        import Bobcat
        import Cobra

Bobcat/
    Project.toml:
        [deps]
        Cobra = "4725e24d-f727-424b-bca0-c4307a3456fa"
        Dingo = "7a7925be-828c-4418-bbeb-bac8dfc843bc"

    src/Bobcat.jl:
        import Cobra
        import Dingo

Cobra/
    Project.toml:
        uuid = "4725e24d-f727-424b-bca0-c4307a3456fa"
        [deps]
        Dingo = "7a7925be-828c-4418-bbeb-bac8dfc843bc"

    src/Cobra.jl:
        import Dingo

Dingo/
    Project.toml:
        uuid = "7a7925be-828c-4418-bbeb-bac8dfc843bc"

    src/Dingo.jl
        # no imports
```

The corresponding `roots` structure materialized as a Julia dictionary:

```julia
roots = Dict{Symbol,UUID}(
    :Aardvark => UUID("00000000-0000-0000-0000-000000000000"), # no project file, nil UUID
    :Bobcat   => UUID("85ad11c7-31f6-5d08-84db-0a4914d4cadf"), # dummy UUID based on path
    :Cobra    => UUID("4725e24d-f727-424b-bca0-c4307a3456fa"), # UUID from project file
    :Dingo    => UUID("7a7925be-828c-4418-bbeb-bac8dfc843bc"), # UUID from project file
)
```

And the corresponding `graph` structure materialized as a Julia dictionary:

```julia
graph = Dict{UUID,Dict{Symbol,UUID}}(
    # Bobcat:
    UUID("85ad11c7-31f6-5d08-84db-0a4914d4cadf") => Dict{Symbol,UUID}(
        :Cobra    => UUID("4725e24d-f727-424b-bca0-c4307a3456fa"),
        :Dingo    => UUID("7a7925be-828c-4418-bbeb-bac8dfc843bc"),
    ),
    # Cobra:
    UUID("4725e24d-f727-424b-bca0-c4307a3456fa") => Dict{Symbol,UUID}(
        :Dingo    => UUID("7a7925be-828c-4418-bbeb-bac8dfc843bc"),
    ),
    # Dingo:
    UUID("7a7925be-828c-4418-bbeb-bac8dfc843bc") => Dict{Symbol,UUID}(),
)
```

A few general rules to note:

1. A package without a project file can depend on any top-level dependency, and since every package in a package directory is available at the top-level, it can import all packages in the environment.
2. A package with a project file cannot depend on one without a project file since packages with project files can only load packages in `graph` and packages without project files do not appear in `graph`.
3. A package with a project file but no explicit top-level UUID entry can only be depended on by packages without project files since they have a dummy UUID which is only used internally by Julia and never user-visible. We've included an explicit dummy UUID value here, but only for demonstration.

We observe the following specifics instances of these rules in our example:

* `Aardvark` could import on any of `Bobcat`, `Cobra` or `Dingo` and does import `Bobcat` and `Cobra`.
* `Bobcat` can and does import both `Cobra` and `Dingo`, both of which have project files with explicit UUIDs and are declared as dependencies in the `[deps]` section of `Bobcat`'s project file. `Bobcat` could not depend on `Aardvark` since `Aardvark` does not have a project file or UUID.
* `Cobra` can and does import `Dingo`, which has a project file and UUID, and is explicitly declared as a dependency in the `[deps]` section of `Cobra`'s project file. `Cobra` could not depend on `Aardvark` or `Bobcat`: the former because it has no project file and the latter because it has no UUID.
* `Dingo` cannot import anything because it has a project file with no `[deps]` section.

**The paths map** in a package directory is extremely simple: it maps subdirectory names to their corresponding entry-point paths. In other words, if the path to our example project directory is `/home/me/AnimalPackages` then the `paths` map would be materialized as this dictionary:

```julia
paths = Dict{Tuple{UUID,Symbol},String}(
    (UUID("00000000-0000-0000-0000-000000000000"), :Aardvark) =>
        "/home/me/AnimalPackages/Aardvark/src/Aardvark.jl",
    (UUID("85ad11c7-31f6-5d08-84db-0a4914d4cadf"), :Bobcat) =>
        "/home/me/AnimalPackages/Bobcat/src/Bobcat.jl",
    (UUID("4725e24d-f727-424b-bca0-c4307a3456fa"), :Cobra) =>
        "/home/me/AnimalPackages/Cobra/src/Cobra.jl",
    (UUID("7a7925be-828c-4418-bbeb-bac8dfc843bc"), :Dingo) =>
        "/home/me/AnimalPackages/Dingo/src/Dingo.jl",
)
```

Since all packages in a package directory environment are, by definiton, subdirectories with the expected entry-point files in the top-level directory, the `paths` map is unsurprisingly uniform and consistent.

#### Environment stacks

### Finding `X`

The name collision between two different packages named `Priv` in the previous section brings us to the first question that Julia must answer to evaluate `import X` or `using X`:

Q: *What is `X` in this context?*

The answer to this question—the identity of `X`—is determined by a combination of three things:

1. The **module** in which the `import` or `using` expression is evaluated,
2. The **load path** as determined by the contents of the `LOAD_PATH` global array,
3. The **contents** of each directory in the load path.


When Julia loads a package `X`, if the package has a UUID associated with it, t stores that UUID in the module that is created. When subsequently loading packages, such as `import Y` in that module or any of its submodule, the identity of `Y` is resolved with respect to `X`.



------

If the dependencies of a project needed share a single namespace, this would be a big problem for you if you ever tried to upgrade `Public` since you would then have two different packages named `Private` as dependencies of your project. With a shared package namespace, the the only real recourse is to rename your private package to something else. This situation is not hypthetical—it's been encountered repeatedly in the wild.

To handle this kind of situation without  is why `import Private` can mean different things in



here may not have been any public package named `X`, but at some point you want to upgrade

Julia packages are identified by [universally unique identifiers](https://en.wikipedia.org/wiki/Universally_unique_identifier) (UUIDs), unique 128-bit values that can be generated independently with astronomically probability of collision. When a Julia package is created, a UUID is generated which identifies it persistently across various events, including:

- name changes,
- changes of repository location,
- changes of ownership,
- package forks.

In any situation where the question *"are these the same package?"* needs to be asked, the answer is determined by comparing UUIDs—two Julia source trees are the same package if and only if they have the same UUID. Accordingly, the question *"what is `X` in this context?"* when evaluating `import X` is answered by determining which UUID the name `X` corresponds to in the context in which the import occurs.

Julia package UUIDs serve a similar purpose to Java's reversed domain package names. For example, in Java you might write `import com.sun.net.httpserver.*;` to import all classes from the `com.sun.net.httpserver` package. This package name suggests that the `httpserver` package is maintained by Sun Microsystems, owner of the `sun.com` domain. Of course, Sun Microsystems no longer exists today, having been acquired by Oracle in 2010. The `com.sun` prefix for these packages, however, is permanent and cannot be updated to `com.oracle`. Even if Oracle were to sell the `sun.com` domain, all Java code using the `httpserver` package would still use the `com.sun` prefix. Since UUIDs have no inherent meaning, they avoid this problem—there's no reason to change a UUID since it entails no meaning. However, UUIDs are are hard for humans to remember and tedious to type. Accordingly, Julia's [package manager](https://julialang.org/Pkg3.jl/latest/) keeps UUIDs mostly out of sight. They don't appear in your Julia code, only in project files specifically designed to record the mapping from names to UUIDs in your projects.

Once the name `X` has been mapped to a UUID, answering the question *"has `X` already been loaded?"* is a simple matter of looking up `X`'s UUID in a table of loaded packages. If `X`'s UUID is in the table, then the loaded module associated with the UUID is bound to `X` in the module where the `import X` or `using X` expression is evaluated. If the `X`'s UUID is not in the global package table, then it must be loaded, which requires answering the next qeustion: *"where can `X` be loaded from?"* but with

for example, but since UUIDs have no meaning, they can handle package renames and even changing the controlling entitity of a package. In the case, for example, Sun Microsystems no longer exists, and `sun.com` redirects to `oracle.com`.



The load path is a stack of "environments", each of which provides three things:

1. `roots`: a map from top-level names to UUIDs
2. `graph`: a graph of dependencies from

searched for in the load path, which is controlled by the `LOAD_PATH` global array
