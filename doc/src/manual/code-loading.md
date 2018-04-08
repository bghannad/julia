# Code Loading

Julia has two mechanisms for loading code:

1. **Code inclusion:** e.g. `include("source.jl")`. Inclusion allows you to split a single program across multiple source files. The expression `include("source.jl")` causes the contents of the file `source.jl` to be evaluated inside of the module where the `include` call occurs, much as if the text of `source.jl` were pasted into that file in place of the `include` call. If `include("source.jl")` is called multiple times, `source.jl` is evaluated multiple times. The included path, `source.jl`, is interpreted relative to the file where the `include` call occurs. This makes it simple to relocate program split across files. In the REPL, included paths are interpreted relative to the current working directory, `pwd()`.
2. **Package loading:** e.g. `import X` or `using X`. The import mechanism allows you to load a package—i.e. an independent, reusable collection of Julia code—and makes the resulting module available by the name `X` inside of the importing module. If the same `X` package is imported multiple times in the a Julia session, it is only loaded the first time: on subsequent imports, the importing module gets a reference to the same package. It should be noted, however, that `import X` can load different packages in different contexts: `X` can refer to one package named `X` in the main project but potentially different packages named `X` in each dependency. More on this below.

Code inclusion is quite straightforward: it simply parse and evaluates a source file in the context of the caller. Package loading is built on top of code inclusion and is quite a bit more complex. The rest of this chapter, therefore, focuses on package loading.

## Package Loading

A *package* is a source tree with a standard layout providing functionality which can be reused by other Julia projects. A package is loaded by `import X` or  `using X` statements. These statements also make the module named `X`, which results from loading the package code, available within the module where the import statement occurs. The meaning of `import X` is context-dependent—its meaning and behavior depend on what code it occurs in—and what it does depends on the answers to two questions:

1. **What** is `X` in this context?
2. **Where** can that `X` be found?

Understanding how Julia answers these questions is key to understanding package loading.

### Federation of packages

Julia supports federated management of packages. This means that multiple independent parties can maintain both public and private registries of packages, and that projects can depend on a mix of packages from all visible registries. Packages from various registries are all installed and managed using the same package management tools and workflows. One consequence of federation is that there cannot be a central authority for package naming. Different entities may use the same name to refer to unrelated packages. This possibility is unavoidable since these entities do not coordinate and may not even know about each other. Beacuse of the lack of a central naming authority, a single project can quite possibly end up depending on different packages with the same name. Julia's package loading mechanism handles this by not requiring package names to be globally unique; instead packages are identified by [universally unique identifiers](https://en.wikipedia.org/wiki/Universally_unique_identifier) (UUIDs) which are assigned to them before they are registered.

Since the decentralized naming problem is somewhat abstract, it may help to walk through a concrete scenario to understand the issue. Suppose you're developing an application called `App`, which uses two packages: `Pub` and  `Priv`. `Priv` is a private package that you created, whereas `Pub` is a public package that you use but don't control. When you created `Priv`, there was no public package by that name. Subsequently, however, an unrelated package also named `Priv` has been published and become popular. In fact, the `Pub` package has started to use it. Therefore, when you next upgrade `Pub` to get the latest bug fixes and features, `App` will end up—through no action of yours other than upgrading—depending on two different packages named `Priv`. `App` has a direct dependency on your private `Priv` package, and an indirect dependency, through `Pub`, on the new public `Priv` package. Since these two `Priv` packages are different but both required for `App` to continue working correctly, the expression `import Priv` must refer to different `Priv` packages depending on whether it occurs in the code of `App` or of `Pub`. Julia's package loading mechanism allows this by distinguishing the two `Priv` packages by context and UUID. How this distinction works is determined by environments, as explained in the following sections.

### Environments

An *environment* determines what `import X` and `using X` mean in various parts of your code and what file they cause to be loaded. Julia understands three kinds of environments:

1. **A project environment** is a directory with a project file and an optional manifest file. The project file determines what the names and identities of the direct dependencies available in the environemnt are, while the manifest file, if present, gives the complete dependency graph, exact versions of each dependency, and sufficient information to determine source locations of each dependency.
2. **A package directories** is an environment determined by the set of package subdirectories it contains. For example, if `X` is a subdirectory and `X/src/X.jl` exists, then `X` is a be top-level dependency available in the environment and `X/src/X.jl` is the source file to load when `import X` is called.
3. **A stacked environment** is a composite environment, composed of an ordered set of project environments and package directories, overlaid to make a single environment, in which all packages available in the contained environments are available. Julia's load path is a stacked environment.

As an abstraction, an environment must provide three maps: `roots`, `graph` and `paths`. When resolving the meaning of `import X`, the `roots` map and dependency `graph` are used to answer the "what" question while the `paths` map answers the "where" question. In more detail, the roles of these maps are:

- **roots:** `name::Symbol` ⟶ `uuid::UUID`

   The `roots` map in an environment maps package names to identities for top-level dependencies, i.e. packages which can be loaded from the `Main` module. When encountering `import X` in the main project in an environment, the Julia process looks up `roots[:X]` to determine the UUID of `X`.

- **graph:** `context::UUID` ⟶ `name::Symbol` ⟶ `uuid::UUID`

   The `graph` map is multilevel, which assigns to each UUID, `context`, another map of package names to UUIDs. When evaluating `import X` in the source of a package with UUID, `context`, Julia looks up `graph[context][:X]` to determine the UUID of `X`. Note that this means that `X` can refer to different package identities in the main project and in each package `context`.

- **paths:** `uuid::UUID` ⟶ `path::String`

   The `paths` map takes package UUIDs and assigns them to the path of an entry point source file to load in order to acquire that package. Once a name has been resolved to a UUID via the `roots` or `graph` maps (according to whether it is loaded from the main project or a dependency), in order to find out where to load that UUID from, the Julia process looks up `paths[uuid]` and includes that file.

Each kind of environment determines these three maps differently, as detailed in the following sections.

#### Project environments

A project environment is determined by a directory containing a project file named `Project.toml` and optionally a manifest file named `Manifest.toml`. These files can also be named or `JuliaProject.toml` and `JuliaManifest.toml` in which case `Project.toml` and `Manifest.toml` are ignored, allowing for file name collisions with other tools which could potentially consider a file named `Project.toml` or `Manifest.toml` significant.

**The roots map** of the environment is determined by the contents of the project file, specifically, its top-level `name` entry , its top-level `uuid` entry, and its `[deps]` section—all of which are may also be absent. Consider the following example project file for the hypothetical application, `App`:

```toml
name = "App"
uuid = "8f986787-14fe-4607-ba5d-fbff2944afa9"

[deps]
Priv = "ba13f791-ae1d-465a-978b-69c3ad90f72b"
Pub  = "c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1"
```

This project file implies the following `roots` map in Julia syntax:

```julia
roots = Dict(
    "App"  => UUID("8f986787-14fe-4607-ba5d-fbff2944afa9"),
    "Priv" => UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b"),
    "Pub"  => UUID("c07ecb7d-0dc9-4db7-8803-fadaaeaf08e1"),
)
```

Given this `roots` map, if when loading the code for `App` itself, one encounterd `import Priv` Julia looks up `roots["Priv"]` and gets `ba13f791-ae1d-465a-978b-69c3ad90f72b`, indicating which `Priv` package to load in that context.

**The depedency graph** of a project environment is determined by the contents of the manifest file, if present (if there is none, it is just an empty graph). The manifest file contains a stanza for each concrete direct or indirect dependency of a project, giving the UUID and exact version hash of each and optionally path of each. Consider the following example manifest file for `App`:

```toml
[[Priv]] # the private one
deps = ["Pub", "SomeOther"]
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
  SomeOther = "f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"

[[SomeOther]]
uuid = "f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"
git-tree-sha1 = "e808e36a5d7173974b90a15a353b564f3494092f"
version = "3.4.2"
```

This manifest file describes the total dependency graph of the `App` project: there are two different `Priv` packages that the application needs, one `Pub` package and another package called `SomeOther`. The dependency graph data structure looks like this:

```julia
graph = Dict{UUID,Dict{String,UUID}}(
    # Priv – the private one:
    UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b") => Dict{String,UUID}(
        "SomeOther" => UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"),
    ),
    # Priv – the public one:
    UUID("2d15fe94-a1f7-436c-a4d8-07a9a496e01c") => Dict{String,UUID}(),
    # Pub:
    UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b") => Dict{String,UUID}(
        "Priv" => UUID("2d15fe94-a1f7-436c-a4d8-07a9a496e01c"),
        "SomeOther" => UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62"),
    ),
    # SomeOther:
    UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62") => Dict{String,UUID}(),
)
```

Given this `graph` structure as an environment, if Julia encounters `import Priv` when loading the code for the `Pub` package, which has UUID `ba13f791-ae1d-465a-978b-69c3ad90f72b`, it looks up

```julia
graph[UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b")]["Priv"]
```

and gets `2d15fe94-a1f7-436c-a4d8-07a9a496e01c` , which indicates that in this context, `import Priv` refers to the _other_ `Priv` package—the public one with that UUID.

**The paths map** of a project environment is also determined by the manifest file, if present (if there is none, it is just an empty map). The path mapping for a package called `name` is determined by these two rules:

1. If the manifest stanza for a package has a `path` entry, then that path is interpreted as being relative to the manifest file. If the location is a directory, return `src/$name.jl`, otherwise return the path.
2. If the manifest stanza has `uuid` and `git-tree-sha1` entries, then compute a deterministic hash function of those two values—call it `slug`—and look for `packages/$name/$slug` in each directory in the global Julia variable `DEPOT_PATH`, returning the first one that exists.

In the example manifest file above, if we're looking for the path of the first `Priv` package with UUID `ba13f791-ae1d-465a-978b-69c3ad90f72b`, we see that its stanza has a `path` entry, so we look at `deps/Priv` relative to the `App` project directory—let's say it's `/home/me/projects/App`—which is where the manifest file lives, see that `deps/Priv` is a directory, and return `/home/me/projects/App/deps/Priv/src/Priv.jl` as the path to the private `Priv` package.

If we're looking for the other `Priv` package, the one with UUID `2d15fe94-a1f7-436c-a4d8-07a9a496e01c`, we see that its stanza does not have a `path` entry, but does have a `git-tree-sha1` entry and the `slug` for this pair of UUID and SHA-1 values is `HDkr` (the exact details of this computation aren't important), so if we had `DEPOT_PATH == ["/users/me/.julia", "/usr/local/julia"]` then Julia would look for

1. `/home/me/.julia/packages/Priv/HDkr/src/Priv.jl` and
2. `/usr/local/julia/packages/Priv/HDkr/src/Priv.jl`

returning the first one that exists. The `paths` map for the entire example manifest file could be:

```julia
paths = Dict{UUID,String}(
    # Priv – the private one:
    UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b") =>
        "/home/me/projects/App/deps/Priv/src/Priv.jl",
    # Priv – the public one:
    UUID("2d15fe94-a1f7-436c-a4d8-07a9a496e01c") =>
        "/home/me/.julia/packages/Priv/HDkr/src/Priv.jl",
    # Pub:
    UUID("ba13f791-ae1d-465a-978b-69c3ad90f72b") =>
        "/usr/local/julia/packages/Pub/oKpw/src/Pub.jl",
    # SomeOther:
    UUID("f7a24cb4-21fc-4002-ac70-f0e3a0dd3f62") =>
        "/home/me/.julia/packages/SomeOther/me9k/src/SomeOther.jl",
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
