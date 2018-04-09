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

An *environment* determines what `import X` and `using X` mean in various parts of your code and what files these statements cause to be loaded. Julia understands three kinds of environments:

1. **A project environment** is a directory with a project file and an optional manifest file. The project file determines what the names and identities of the direct dependencies of a project are. The manifest file, if present, gives a complete dependency graph, including all direct and indirect dependencies, exact versions of each dependency, and sufficient information to locate and load the correct version.
2. **A package directory** is a directory containing, as subdirectories, the source trees of a set of packages. This is how package loading worked in older versions of Julia. For example, if `X` is a subdirectory and `X/src/X.jl` exists, then `X` is  available as dependency in the package directory's environment and `X/src/X.jl` is the source file loaded by `import X` in that environment.
3. **A stacked environment** is an ordered set of project environments and package directories, overlaid to make a single composite environment, in which all packages available in the contained environments are available. Julia's load path, for example, is a stacked environment. By adding environments to a stack, you can make additional top-level packages available.

As an abstraction, an environment provides three maps: `roots`, `graph` and `paths`. When resolving the meaning of `import X`, `roots` and `graph` are used to determine the identity of `X` and thereby answer the "what" question, while the `paths` map is used to locate the source of a specific version of `X` and thereby anser the "where" question. In more detail, the function and behavior of these three maps are:

- **roots:** `name::Symbol` ⟶ `uuid::UUID`

   An environment's `roots` map assigns package names to UUIDs for all the top-level dependencies that the environment makes availabe to the main project (i.e. the ones that can be loaded in `Main`). When Julia encounters `import X` in the main project, it looks up the UUID of `X` as `roots[:X]`.

- **graph:** `context::UUID` ⟶ `name::Symbol` ⟶ `uuid::UUID`

   An environment's `graph` is a multilevel map which assigns, for each `context` UUID, a map from names to UUIDs, similar to the `roots` map but specific to that `context`. When Julia sees `import X` in the package whose UUID is `context`, it looks up `graph[context][:X]` to determine `X`'s UUID. This means that `import X` can refer to different packages depending on `context`.

- **paths:** `uuid::UUID` ⟶ `path::String`

   The `paths` map assigns UUIDs to paths of source files for packages. Once the identity of `X` in `import X` has been resolved to a UUID via `roots` or `graph`, depending on whether it is loaded from the main project or an dependency,the Julia process looks up `paths[uuid]` and includes that file. Including that file must create a module named `X`—this module is the instantiation of package `X`.

Each kind of environment determines these three maps differently, as detailed in the following sections.

#### Project environments

A project environment is determined by a directory containing a project file named `Project.toml` and optionally a manifest file named `Manifest.toml`. These files can also be named `JuliaProject.toml` and `JuliaManifest.toml` in which case `Project.toml` and `Manifest.toml` are ignored if they exist, allowing for file name collisions with other tools which could potentially consider files named `Project.toml` and `Manifest.toml` significant. For pure Julia projects, the names `Project.toml` and `Manifest.toml` should be preferred. The `roots`, `graph` and `paths` maps of a project environment are defined as follows.

**The roots map** of the environment is determined by the contents of the project file, specifically, its top-level `name` and `uuid` entries and its `[deps]` section (any of these three may be absent). Take the following example project file for the hypothetical application, `App`, as described above:

```toml
name = "App"
uuid = "8f986787-14fe-4607-ba5d-fbff2944afa9"

[deps]
Priv = "ba13f791-ae1d-465a-978b-69c3ad90f72b"
Pub  = "c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1"
```

This project file gives the following `roots` map:

```julia
roots = Dict(
    :App  => UUID("8f986787-14fe-4607-ba5d-fbff2944afa9"),
    :Priv => UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b"),
    :Pub  => UUID("c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1"),
)
```

Note that for efficiency reasons, `roots` maps are not actually materialized as dictionaries in Julia but are instead implicitly queried through various internal APIs. Given the above `roots` map, when loading the code for `App` itself, `import Priv` causes Julia to look up `roots[:Priv]`, which yields `ba13f791-ae1d-465a-978b-69c3ad90f72b` indicating that this is the UUID of the `Priv` package to load in that context.

**The depedency graph** of a project environment is determined by the contents of the manifest file, if present (if there is no manifest file then `graph` is empty). The manifest file contains a stanza for each concrete direct or indirect dependency of a project, giving the UUID and exact version hash of each dependency and optionally an explicit path. Take the following manifest file for our `App` example:

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

This manifest file describes the complete dependency graph of the `App` project. There are two different `Priv` packages that the application needs—one is a direct dependency, the other an indirect dependency through `Pub`. It also depends on `Pub` package. The `Pub` package and the private `Priv` package both depend on another package called `Zebra` which is not a direct dependency of `App`. Thus, the materialized full dependency `graph` data structure looks like this:

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
    # SomeOther:
    UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62") => Dict{Symbol,UUID}(),
)
```

Again, for efficicency reasons, Julia doesn't necessary materialize this graph during package loading, but it computes portions of this graph on demand. Given this dependency `graph`, when Julia sees `import Priv` in the `Pub` package—which has UUID `ba13f791-ae1d-465a-978b-69c3ad90f72b`—it looks up

```julia
graph[UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b")][:Priv]
```

and gets `2d15fe94-a1f7-436c-a4d8-07a9a496e01c` , which indicates that in the context of the `Pub` package,  `import Priv` refers to the _other_ `Priv` package—the public one. Contrast this with what we described happening in the previous when evaluating `import Priv` in the context of the main project, resolved by `roots[:Priv]`, yielding `ba13f791-ae1d-465a-978b-69c3ad90f72b`, the UUID of the other, private `Priv` package.

What happens if `import Zebra` occurs in the main `App` project? Since `Zebra` does not occur in the project file, the import fails, even though `Zebra` does appear in the manifest file. Moreover, if `import Zebra` appears in the public `Priv` package (the one with UUID `2d15fe94-a1f7-436c-a4d8-07a9a496e01c` ), that will also fail since there is no `deps` entry map in that package's manifest stanza. Given this environment, the public `Priv` package cannot load *any* dependencies at all since it declares none. And the `Zebra` package can only be loaded by the  `Pub` package and the private `Priv` package—i.e. the two packages for which it is explicitly declared as a dependency in the manifest file.

**The paths map** of a project environment is also determined by the manifest file if present (if there is none, it is an empty map). The path for a package called `name` is determined by these two rules:

1. If the manifest stanza for a package has a `path` entry, then that path, relative to the manifest file, is returned. If the location is a directory, return `src/$name.jl`, otherwise return the path itself.
2. If the manifest stanza has `uuid` and `git-tree-sha1` entries, compute a deterministic hash function of those two values, call it `slug`, and look for the directory `packages/$name/$slug` in each directory in the Julia variable `DEPOT_PATH`. Return the first such directory which exists.

In the example manifest file above, to find the path of the first `Priv` package, the one with UUID `ba13f791-ae1d-465a-978b-69c3ad90f72b`, we look in its stanza and see that it has a `path` entry, so we look at `deps/Priv` relative to the `App` project directory, let's say it's `/home/me/projects/App`, see that `/home/me/projects/App/deps/Priv` is a directory, and therefore return

* `/home/me/projects/App/deps/Priv/src/Priv.jl`

as the path to the private `Priv` package. If we're looking for the other `Priv` package, the one with UUID `2d15fe94-a1f7-436c-a4d8-07a9a496e01c`, we see that its stanza does not have a `path` entry, but it does have a `git-tree-sha1` entry. The `slug` for this pair of UUID and SHA-1 values is `HDkr` (the exact details of this computation aren't important, but it is consistent and predictable across Julia versions). Thus, if we had

```
DEPOT_PATH == ["/users/me/.julia", "/usr/local/julia"]
```

then Julia would look for the following paths to see if any of them exist:

1. `/home/me/.julia/packages/Priv/HDkr/src/Priv.jl` and
2. `/usr/local/julia/packages/Priv/HDkr/src/Priv.jl`

Depending on the details of where things live on the local file syste, the `paths` map for the entire `App` example environment could be:

```julia
paths = Dict{UUID,String}(
    # Priv – the private one:
    UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b") =>
        # relative entry-point inside `App` repo:
        "/home/me/projects/App/deps/Priv/src/Priv.jl",
    # Priv – the public one:
    UUID("2d15fe94-a1f7-436c-a4d8-07a9a496e01c") =>
        # package installed in the user depot:
        "/home/me/.julia/packages/Priv/HDkr/src/Priv.jl",
    # Pub:
    UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b") =>
        # package installed in the system depot:
        "/usr/local/julia/packages/Pub/oKpw/src/Pub.jl",
    # Zebra:
    UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62") =>
        # package installed in the user depot:
        "/home/me/.julia/packages/SomeOther/me9k/src/Zebra.jl",
)
```

#### Package directories

Package directories provide a kind of environment that approximates package loading in Julia 0.6 and earlier, and which resembles package loading in many other dynamic languages.

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
