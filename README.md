
# Flake Parts

_Core of a distributed framework for writing Nix Flakes._

`flake-parts` provides the options that represent standard flake attributes and establishes a way of working with `system`. Opinionated features are provided by an ecosystem of modules that you can import.

# Why Modules?

Flakes are configuration. The module system lets you refactor configuration
into modules that can be shared.

It reduces the proliferation of custom Nix glue code, similar to what the
module system has done for NixOS configurations.

Unlike NixOS, but following Flakes' spirit, `flake-parts` is not a
monorepo with the implied goal of absorbing all of open source, but rather
a single module that other repositories can build upon, while ensuring a
baseline level of compatibility: which core attribute make up a flake and
how these are represented as module options.

# Getting Started

If your project does not have a flake yet:

```console
nix flake init -t github:hercules-ci/flake-parts
```

Otherwise, add the input,

```
    flake-parts.url = "github:hercules-ci/flake-parts";
    flake-parts.inputs.nixpkgs.follows = "nixpkgs";
```

then slide `mkFlake` between your outputs function head and body,

```
  outputs = { self, flake-parts, ... }:
    flake-parts.lib.mkFlake { inherit self; } {
      flake = {
        # Put your original flake attributes here.
      };
      systems = [
        # systems for which you want to build the `perSystem` attributes
        "x86_64-linux"
        # ...
      ];
    };
```

Now you can add the remaining module attributes like in the [the template](./template/default/flake.nix).

# Example

See [the template](./template/default/flake.nix).

# Options Reference

See [flake.parts](https://flake.parts/options.html)

# Top-level module parameters

 -  `config`, `options`, `lib`, ...: standard module system parameters.

 -  `getSystem`: function from system string to the `config` of the appropriate `perSystem`.

 -  `moduleWithSystem`: function that brings the `perSystem` module arguments.
    This allows a module to reference the defining flake without introducing
    global variables that need to be set by the user of the module.

    ```nix
    { moduleWithSystem, ... }:
    {
      nixosModules.default = moduleWithSystem (
        perSystem@{ config }:  # NOTE: only explicit params will be in perSystem
        nixos@{ ... }:
        {
          services.foo.package = perSystem.config.packages.foo;
          imports = [ ./nixos-foo.nix ];
        }
      );
    }
    ```

 -  `withSystem`: enter the scope of a system. Worked example:

    ```nix
    { withSystem, ... }:
    {
      # perSystem = ...;

      nixosConfigurations.foo = withSystem "x86_64-linux" (ctx@{ pkgs, ... }:
        pkgs.nixos ({ config, lib, packages, pkgs, ... }: {
          _module.args.packages = ctx.config.packages;
          imports = [ ./nixos-configuration.nix ];
          services.nginx.enable = true;
          environment.systemPackages = [
            packages.hello
          ];
        }));
    }
    ```

# `perSystem` module parameters

 -  `pkgs`: Defaults to `inputs.nixpkgs.legacyPackages.${system}`. Can be set via `config._module.args.pkgs`.

 -  `inputs'`: The flake `inputs` parameter, but with `system` pre-selected. Note the last character of the name, `'`, pronounced "prime".

    `system` selection is handled by the extensible function [`perInput`](https://flake.parts/options.html#opt-perInput).

 -  `self'`: The flake `self` parameter, but with `system` pre-selected. This might trigger an infinite recursion (#22), so prefer `config`.

 -  `system`: The system parameter, describing the architecture and platform of
    the host system (where the thing will run).

# Equivalences

 - Getting the locally defined `hello` package on/for an `x86_64-linux` host:
   - `nix build #hello` (assuming [`systems`](https://flake.parts/options.html#opt-systems) has `x86_64-linux`)
   - `config.packages.hello` (if `config` is the `perSystem` module argument)
   - `allSystems."x86_64-linux".packages.hello` (assuming [`systems`](https://flake.parts/options.html#opt-systems) has `x86_64-linux`)
   - `(getSystem "x86_64-linux").packages.hello)`
   - `withSystem "x86_64-linux" ({ config, ... }: config.packages.hello)`

Why so many ways?

1. Flakes counterintuitively handles `system` by enumerating all of them in attribute sets. `flake-parts` does not impose this restriction, but does need to support it.
2. `flake-parts` provides an extensible structure that is richer than the flakes interface alone. 
