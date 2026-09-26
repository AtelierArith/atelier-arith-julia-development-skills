---
name: creating-julia-app
description: Use when you create a CLI written in Julia packages or MCP servers
---

# Creating a Julia app

Apps are Julia packages that are intended to be run as "standalone programs" (by e.g. typing the name of the app in the terminal possibly together with some arguments or flags/options). This is in contrast to most Julia packages that are used as "libraries" and are loaded by other files or in the Julia REPL.

A Julia app is structured similar to a standard Julia library with the following additions:

A `@main` entry point in the package module (see the Julia help on `@main` for details)
An `[apps]` section in the `Project.toml` file listing the executable names that the package provides.
A very simple example of an app that prints the reversed input arguments would be:

```julia
# src/MyReverseApp.jl
module MyReverseApp

function (@main)(ARGS)
    for arg in ARGS
        print(stdout, reverse(arg), " ")
    end
    return
end

end # module
```

```toml
# Project.toml

# standard fields here

[apps]
reverse = {}
```

The empty table `{}` is to allow for giving metadata about the app. A package can define several apps, point an executable at a submodule, and set default Julia flags per app:

```toml
[apps]
main-app = {}
cli-app = { submodule = "CLI" }
fast-app = { julia_flags = ["--threads=auto", "--optimize=2"] }
```

## Installing and running

App support in Pkg is still experimental. Install the app with `Pkg.Apps.add` or `pkg> app add` (from a registry, a git URL, or a local path), or use `Pkg.Apps.develop` / `pkg> app develop` to run it live from a local checkout so edits are reflected immediately:

```julia-repl
pkg> app add MyReverseApp
pkg> app add https://github.com/example/MyReverseApp.jl
pkg> app develop path/to/MyReverseApp   # local path must be a git repository
```

Installing creates a shim in `~/.julia/bin`, which you must add to `PATH` yourself. After that, run the app by name:

```sh
$ reverse some input string
 emos tupni gnirts
```

Manage installed apps with `pkg> app status`, `pkg> app update [name]`, and `pkg> app rm name`. An app runs with the same Julia executable that installed it; override that for all apps with the `JULIA_APPS_JULIA_CMD` environment variable.

Installing an app installs third-party code that will be executed when the app runs. Prefer a registry entry or an immutable tag/commit (`#v1.2.3`, or a commit SHA) over a mutable branch, and review the source before running an app you did not write.

## MCP servers

A Julia MCP server is an app that speaks JSON-RPC over stdio: it reads requests from stdin and writes responses to stdout. Use the same `@main` / `[apps]` structure so it can be installed and launched by name, and keep stdout reserved for protocol messages — send logs to stderr. See the [Model Context Protocol](https://modelcontextprotocol.io/) specification for the wire format.

Learn more:

- [Apps](https://pkgdocs.julialang.org/v1/apps/)
- [Multiple Apps per Package](https://pkgdocs.julialang.org/v1/apps/#Multiple-Apps-per-Package)
