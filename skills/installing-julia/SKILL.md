---
name: installing-julia
description: Use when your machine does not have JuliaLang runtime
---

# Installing Julia

If your machine does not have the Julia programming language runtime installed, follow the instructions below to install it.

On macOS or Linux, run the following command:

```sh
$ curl -fsSL https://install.julialang.org | sh -s -- --yes
```

On Windows, run the following command in PowerShell:

```PS
PS> winget install --name Julia --id 9NJNWW8PVKMN -e -s msstore
```
