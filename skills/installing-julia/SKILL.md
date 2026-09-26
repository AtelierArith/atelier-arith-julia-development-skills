---
name: installing-julia
description: Use when your machine does not have the JuliaLang runtime; guide the user to the official Julia installation page instead of running install scripts
---

# Installing Julia

If your machine does not have the Julia programming language runtime installed,
do **not** run installation scripts on the user's behalf. Installation changes
the user's machine (shell profiles, `PATH`, installed toolchains), so the user
should perform it deliberately and with full visibility.

Instead, point the user to the official Julia installation instructions and let
them choose and run the method themselves.

## What to do

1. Tell the user that Julia is not installed and that you will not install it
   automatically because installation modifies their machine.
2. Direct them to the official page: <https://julialang.org/install/>.
3. Summarize the officially offered options so they can pick one, without
   executing any of them:

   - **`juliaup`** — the recommended version manager; also lets you switch
     between multiple Julia versions. Official instructions:
     <https://julialang.org/install/#juliaup>
   - **Direct download** — download the platform-specific installer or archive
     from the downloads page and follow the on-page steps:
     <https://julialang.org/downloads/>
   - **Package managers** — official guidance for Homebrew, `winget`, and
     Linux distribution packages is linked from the install page.

4. After the user installs Julia, ask them to confirm by running:

   ```sh
   julia --version
   ```

   Then continue with the task (for example, package development with
   [[generating-julia-package]]).

## Notes

- Treat the official pages above as the source of truth. Installation commands
  and supported platforms change over time, so do not paste remembered commands
  as if they were current.
- For choosing a version to install (rather than how to install it), see
  [[finding-latest-julia-version]].
- If the user explicitly asks you to run the commands yourself, present the
  exact commands from the official page first and get their confirmation before
  running anything.
